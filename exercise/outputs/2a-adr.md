> **Agent:** Kiro (Amazon Nova / Auto model, Kiro IDE)

# ADR-0002: เลือกรูปแบบการผสานรวมสำหรับ Post-Order Actions

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** ทีมสถาปัตยกรรม Online Boutique

## Context and problem

หลังจากที่ลูกค้าสั่งซื้อสำเร็จ ระบบ Online Boutique ต้องดำเนินการหลายอย่างพร้อมกัน ได้แก่:

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** (external email provider รายใหม่ที่แทนที่ `emailservice` เดิม)
2. มอบ loyalty points ผ่าน **loyaltyservice** (บริการใหม่ที่ต้องสร้างขึ้น)
3. บันทึก analytics event "order placed" สำหรับทีม marketing

นอกจากนี้ ทีม marketing ระบุว่าจะมี post-order actions เพิ่มเติมในปีหน้า เช่น SMS, fraud scoring, และ recommendations update ดังนั้นสถาปัตยกรรมต้องรองรับการขยายตัวในอนาคตได้ง่าย

ปัจจุบัน `checkoutservice` เรียก `emailservice` โดยตรงผ่าน gRPC แบบ synchronous ซึ่งหากยังคงรูปแบบนี้และเพิ่ม caller ใหม่ทุกครั้ง จะทำให้ `checkoutservice` มี coupling สูงและแก้ไขยากขึ้นเรื่อย ๆ

## Decision drivers

- **Modifiability:** ต้องเพิ่ม post-order action ใหม่ได้โดยไม่แก้ไข `checkoutservice`
- **Availability:** ความล้มเหลวของ loyaltyservice หรือ analytics ต้องไม่ทำให้คำสั่งซื้อล้มเหลว
- **Performance:** latency ของการ place order ต้องไม่เพิ่มขึ้นมากจากการเรียก downstream หลายตัว
- **Simplicity / cost:** ความซับซ้อนของระบบต้องอยู่ในระดับที่ทีมบำรุงรักษาได้

## Considered options

1. **Synchronous fan-out จาก `checkoutservice`** — `checkoutservice` เรียก MailFast, loyaltyservice และ analytics โดยตรงผ่าน gRPC/HTTP แบบ synchronous
2. **Asynchronous messaging ผ่าน message broker (event-driven)** — `checkoutservice` publish event `OrderPlaced` ไปยัง message broker; downstream services (MailFast adapter, loyaltyservice, analytics) subscribe และประมวลผลอิสระจากกัน
3. **Orchestration ผ่าน workflow service** — `checkoutservice` เรียก workflow service แยกต่างหาก ซึ่งจัดการลำดับและ retry ของ post-order actions

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous fan-out | เรียบง่าย ไม่ต้องใช้ infrastructure เพิ่ม, debug ง่าย, latency ของแต่ละขั้นตอนมองเห็นได้ชัด | `checkoutservice` ต้องแก้ทุกครั้งที่มี action ใหม่ (modifiability ต่ำ); ความล้มเหลวของ loyaltyservice หรือ analytics อาจทำให้ checkout ล้มเหลว (availability ต่ำ); latency รวมสูงขึ้นเพราะรอทุก call |
| 2. Asynchronous messaging | `checkoutservice` ไม่ต้องรู้จัก downstream services (loose coupling); เพิ่ม consumer ใหม่โดยไม่แก้ `checkoutservice` (modifiability สูง); ความล้มเหลวของ consumer ไม่กระทบ checkout (availability สูง); latency ของ checkout ไม่เพิ่ม | ต้องใช้ message broker เพิ่ม (cost, operational complexity); event delivery และ ordering ต้องออกแบบให้ดี; debugging ยากกว่า synchronous |
| 3. Workflow orchestration | จัดการ retry และ compensating transaction ได้ดี; มองเห็น flow ชัดเจน | เพิ่ม complexity สูงสุด; ต้องการ workflow engine; over-engineering สำหรับ use case ปัจจุบัน |

## Decision

เลือก **Option 2: Asynchronous messaging ผ่าน message broker**

`checkoutservice` publish event `OrderPlaced` ไปยัง message broker หลังจากคำสั่งซื้อสำเร็จ โดย consumers แต่ละตัวได้แก่ MailFast adapter, loyaltyservice และ analytics service subscribe event นี้แล้วทำงานอิสระ

เหตุผล: requirement ระบุชัดว่าจะมี post-order actions เพิ่มในอนาคต ดังนั้น modifiability เป็น driver หลัก Option 2 ตอบ driver นี้ได้ดีที่สุด และยังป้องกัน checkout ล้มเหลวจาก downstream failures ได้ด้วย ความซับซ้อนที่เพิ่มขึ้นจาก message broker ถือว่าคุ้มค่ากับ flexibility ที่ได้รับ

## Consequences

- **Positive:** `checkoutservice` ไม่ต้องแก้เมื่อเพิ่ม post-order action ใหม่ (quality attribute: modifiability)
- **Positive:** ความล้มเหลวของ loyaltyservice หรือ analytics ไม่กระทบ checkout flow (quality attribute: availability)
- **Positive:** latency ของ checkout ไม่เพิ่มจากการรอ downstream (quality attribute: performance)
- **Negative:** ต้องเพิ่ม message broker เข้า infrastructure ซึ่งเพิ่มค่าใช้จ่ายและภาระ operations (quality attribute: cost)
- **Negative:** distributed tracing และ debugging ทำได้ยากกว่าแบบ synchronous (quality attribute: maintainability)
- **Risks and mitigations:** หาก message broker ล่ม post-order actions ทั้งหมดจะหยุดทำงาน — บรรเทาด้วยการใช้ managed broker ที่มี high availability หรือตั้ง dead-letter queue เพื่อ retry

## Validation

- ทดสอบ end-to-end: สั่งซื้อ 1 รายการ แล้วตรวจว่า MailFast ได้รับ request, loyaltyservice บันทึก points, และ analytics บันทึก event ภายใน SLA ที่กำหนด
- ทดสอบ fault isolation: ปิด loyaltyservice แล้วตรวจว่า checkout ยังสำเร็จและ message ถูก queue ไว้
- กำหนด review date หลัง deploy 30 วัน เพื่อวัด latency ของ checkout และ error rate ของ consumers

## Open questions

- **Message broker ใด?** ยังไม่ได้เลือก (เช่น Kafka, RabbitMQ, Google Pub/Sub) — ต้องพิจารณา infrastructure ที่มีอยู่และ team skill
- **Guaranteed delivery?** ต้องการ at-least-once หรือ exactly-once สำหรับ loyalty points?
- **MailFast API:** ยังไม่ทราบ schema และ authentication method ของ MailFast
- **Analytics target:** ยังไม่ทราบว่า marketing ใช้ระบบใด (BigQuery, Segment, custom service?)
- **loyaltyservice SLA:** ต้องการ latency และ availability เท่าใดสำหรับการมอบ points?
