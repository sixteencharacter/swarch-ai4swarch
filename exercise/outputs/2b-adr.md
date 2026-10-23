> Agent: Kiro (Auto model, Amazon Kiro IDE)

# ADR-0002: เลือกรูปแบบการรวม (Integration Pattern) สำหรับ Post-Order Actions

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** ทีมพัฒนา Online Boutique (4 นักพัฒนา)

## Context and problem

ปัจจุบัน `checkoutservice.PlaceOrder` เรียกใช้ `emailservice` แบบ synchronous ผ่าน gRPC หลังจากชำระเงินและจัดส่งสำเร็จ
(`system/online-boutique/src/checkoutservice/main.go`, ฟังก์ชัน `PlaceOrder` บรรทัดที่เรียก `cs.sendOrderConfirmation`)
หาก email call ล้มเหลว ระบบจะแค่ log คำเตือนและยังคืนผลสำเร็จให้ผู้ใช้

ความต้องการใหม่ (R-12) กำหนดให้หลังจาก order ถูกวางแล้ว ระบบต้องทำสิ่งต่อไปนี้:

1. ส่ง email ยืนยัน order ผ่าน **MailFast** (external provider แทน `emailservice` เดิม)
2. มอบ loyalty points ผ่าน `loyaltyservice` ที่จะสร้างใหม่
3. บันทึก analytics event "order placed" สำหรับทีม marketing

ฝ่าย marketing ยังคาดว่าจะมี post-order action เพิ่มเติมในปีหน้า (SMS, fraud scoring, recommendations update)

ข้อจำกัดสำคัญจาก scenario:

| หัวข้อ | ตัวเลข |
|---|---|
| Checkout latency | p95 ≤ 500 ms (ปัจจุบัน ~300 ms เหลือ budget ประมาณ 200 ms) |
| Peak load | 200 orders/s ในช่วง 11.11 |
| MailFast latency | p95 **1.5 s**, p99 **8 s**; fail rate ~0.5%; rate limit 50 emails/s |
| Loyalty points | ต้องได้รับภายใน 1 นาที; **exactly once** (ห้ามได้คะแนนซ้ำ) |
| Availability | order ต้องไม่ fail เพราะ email/loyalty/analytics ล้มเหลว |
| ทีม | Go และ gRPC เชี่ยวชาญ; 2 คนมีประสบการณ์ RabbitMQ |
| Infrastructure | Kubernetes; เน้น open-source, self-hosted |

ปัญหาหลัก: MailFast มี p95 latency 1.5 s ซึ่งสูงกว่า checkout latency budget ที่เหลืออยู่ (200 ms) มาก หากเรียกแบบ synchronous จะทำให้ p95 ของ `PlaceOrder` เกิน 500 ms อย่างแน่นอน

## Decision drivers

1. **Performance:** p95 ของ `PlaceOrder` ต้องไม่เกิน 500 ms — MailFast p95 1.5 s ตัดความเป็นไปได้ของ synchronous call
2. **Availability:** order ต้องสำเร็จแม้ email, loyalty หรือ analytics จะล่ม
3. **Exactly-once delivery สำหรับ loyalty points:** ห้ามให้คะแนนซ้ำ
4. **Modifiability:** รองรับ post-order action ใหม่ในปีหน้าโดยไม่ต้องแก้ `checkoutservice`
5. **Operability:** ทีมมี 4 คน, ต้องการ component ที่ดูแลได้บน Kubernetes, เน้น open-source

## Considered options

1. Synchronous gRPC calls จาก `checkoutservice` (ขยายจากแบบเดิม)
2. Message broker (RabbitMQ) — publish event แล้วให้ consumer แต่ละตัวทำงาน asynchronously
3. Choreography via Outbox Pattern — `checkoutservice` เขียน event ลง database outbox แล้ว relay ผ่าน message broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| **1. Synchronous gRPC** | ง่ายที่สุด; tracing และ error handling ตรงไปตรงมา; ทีมคุ้นเคย | MailFast p95 1.5 s ทำให้ latency เกิน 500 ms **ละเมิด performance requirement**; order fail ถ้า downstream ล่ม; เพิ่ม action ใหม่ต้องแก้ `checkoutservice` ทุกครั้ง (modifiability ต่ำ) |
| **2. RabbitMQ message broker** | `checkoutservice` แค่ publish event → latency ต่ำ; downstream ล่มไม่กระทบ order; เพิ่ม consumer ใหม่ได้โดยไม่แก้ publisher; ทีม 2 คนมีประสบการณ์; runs on Kubernetes | ต้องดูแล RabbitMQ cluster; exactly-once ต้องออกแบบเพิ่ม (idempotent consumer + message deduplication); observability ซับซ้อนขึ้น |
| **3. Outbox Pattern + broker** | Guaranteed delivery แม้ `checkoutservice` crash หลัง commit; exactly-once ง่ายกว่า option 2 | ซับซ้อนมากที่สุด; ต้องการ transactional outbox table ใน `checkoutservice`; `checkoutservice` ปัจจุบันไม่มี relational DB (Not evidenced in the provided files ว่ามี DB); overhead สูงสำหรับทีม 4 คน |

## Decision

เลือก **Option 2: RabbitMQ message broker**

หลังจาก `PlaceOrder` สำเร็จ (payment + shipping เสร็จ) `checkoutservice` จะ publish event `order.placed` ไปยัง RabbitMQ exchange
Consumer แยกกันสามตัวจะรับ event นี้:

- **MailFast adapter** (service ใหม่หรือแทนที่ `emailservice`) — ส่ง email ยืนยันผ่าน MailFast HTTP API
- **`loyaltyservice`** (service ใหม่) — มอบ loyalty points พร้อม idempotency key จาก `order_id`
- **analytics consumer** — บันทึก event สำหรับทีม marketing

เหตุผลที่เลือก:
- publish event ใช้เวลาน้อยกว่า 1 ms ทำให้ `PlaceOrder` ยังอยู่ใน latency budget (performance)
- downstream ล่มไม่ทำให้ order fail (availability)
- loyalty consumer ใช้ `order_id` เป็น idempotency key เพื่อป้องกันการให้คะแนนซ้ำ (exactly-once)
- เพิ่ม consumer ใหม่ได้โดยไม่แก้ `checkoutservice` (modifiability)
- RabbitMQ เป็น open-source, self-hosted บน Kubernetes ได้ ทีมมีประสบการณ์บางส่วน (operability)

Option 3 ถูกตัดออกเพราะ `checkoutservice` ไม่มี relational database อยู่แล้ว การเพิ่ม outbox table จะต้องเพิ่ม dependency ใหม่ที่ใหญ่กว่า RabbitMQ อย่างมีนัยสำคัญ

## Consequences

- **Positive:** `PlaceOrder` publish event ใน < 1 ms — p95 latency ยังอยู่ในงบ 500 ms (performance)
- **Positive:** email, loyalty หรือ analytics ล่มไม่กระทบ order (availability)
- **Positive:** เพิ่ม post-order action ใหม่ = เพิ่ม consumer ใหม่ ไม่ต้องแก้ `checkoutservice` (modifiability)
- **Negative:** ต้องดูแล RabbitMQ cluster บน Kubernetes — เพิ่ม operational complexity (operability)
- **Negative:** distributed tracing ข้าม message boundary ยากกว่า synchronous (observability)
- **Negative:** consumer ที่ล้มเหลวต้อง retry — ต้องออกแบบ dead-letter queue และ retry policy (reliability)
- **Risks and mitigations:**
  - _Exactly-once สำหรับ loyalty points:_ ใช้ `order_id` เป็น idempotency key ใน `loyaltyservice`; consumer ต้อง idempotent; ใช้ RabbitMQ message deduplication หรือ consumer-side dedup
  - _RabbitMQ เป็น single point of failure:_ deploy RabbitMQ แบบ cluster (quorum queues) บน Kubernetes เพื่อ high availability
  - _Rate limit MailFast 50 emails/s:_ MailFast adapter ต้องมี rate-limiting logic หรือ queue throttling; ที่ 200 orders/s จะเกิน limit — **[Inferred]** อาจต้องใช้ token bucket หรือ delay queue

## Validation

1. Load test `PlaceOrder` ที่ 200 req/s และวัด p95 latency — ต้องไม่เกิน 500 ms
2. ทดสอบ `loyaltyservice` โดย publish event `order.placed` ซ้ำกัน (`order_id` เดิม) และตรวจสอบว่า loyalty points ถูกให้เพียงครั้งเดียว
3. หยุด MailFast adapter แล้ว place order — order ต้องสำเร็จ; ตรวจสอบว่า message ยังค้างใน queue และถูกประมวลผลเมื่อ adapter กลับมา
4. ตรวจสอบ analytics events ใน pipeline ของ marketing team ว่าตรงกับ order ที่เกิดขึ้นจริง

---

## Open questions

1. `checkoutservice` จะรับรอง delivery ของ message ในระดับไหน (at-least-once vs. at-most-once)? ถ้า `checkoutservice` crash หลัง PlaceOrder สำเร็จแต่ก่อน publish — event จะหายหรือไม่? (ต้องการข้อมูลเพิ่มเติมเรื่อง transaction boundary)
2. MailFast rate limit 50 emails/s เทียบกับ peak 200 orders/s — ต้องการกลยุทธ์ throttling ที่ชัดเจนก่อน implement
3. SLA ของ analytics event: marketing ยอมรับ delay ได้นานแค่ไหน? (ส่งผลต่อ retry policy ของ consumer)
4. `loyaltyservice` จะใช้ data store ชนิดใด? ต้องการ persistent storage ที่รองรับ idempotency check
5. มีนโยบาย data retention สำหรับ RabbitMQ message หรือไม่?
