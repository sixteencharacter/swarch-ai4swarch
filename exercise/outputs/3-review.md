> Agent: Kiro (Auto model, Amazon Kiro IDE)

# บทวิจารณ์สถาปัตยกรรม: ADR-0002 — Post-Order Actions Integration Pattern

---

## 1. Quality-Attribute Scenarios

| # | Source | Stimulus | Environment | Response | Response Measure |
|---|---|---|---|---|---|
| QAS-1 | ผู้ใช้ปลายทาง | กด "Place Order" ขณะ peak load ช่วง 11.11 (200 orders/s) | ระบบทำงานปกติ; Kafka cluster พร้อมใช้งาน | `PlaceOrder` publish event แล้วตอบกลับสำเร็จ | p95 latency ≤ 500 ms (scenario: ปัจจุบัน ~300 ms, budget เหลือ ~200 ms) |
| QAS-2 | MailFast adapter | Kafka message ถูก consume; MailFast ตอบช้า (p95 1.5 s) หรือล้มเหลว (0.5%) | ระบบทำงานปกติ | order สำเร็จ; email ถูก retry จาก queue | MailFast failures ไม่ทำให้ order fail; email ถูกส่งในที่สุด (eventual delivery) |
| QAS-3 | ระบบ | event `order.placed` ถูก publish ซ้ำ (Kafka redelivery หรือ producer retry) | `loyaltyservice` กำลังทำงาน | loyalty points ถูกมอบเพียงครั้งเดียว | ไม่มี duplicate points สำหรับ `order_id` เดียวกัน (exactly-once semantic) |

---

## 2. Findings Table

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | Kafka ถูกเลือกโดยไม่มี evidence ว่าทีมมีประสบการณ์ใช้งาน ขณะที่ ADR ระบุว่าทีม 2 คนมีประสบการณ์ RabbitMQ แต่ไม่มีใครมีประสบการณ์ Kafka | การ operate Kafka cluster บน Kubernetes (ZooKeeper/KRaft, partition tuning, consumer group management) ซับซ้อนกว่า RabbitMQ อย่างมีนัยสำคัญ — เสี่ยงต่อ operational incident จากทีมที่ไม่คุ้นเคย |
| **Risk** | MailFast rate limit 50 emails/s แต่ peak load คือ 200 orders/s — ADR ระบุว่าจะใช้ client-side rate limiting แต่ไม่ได้อธิบาย mechanism ว่า message ส่วนที่เกิน limit จะถูกจัดการอย่างไร | ที่ 200 orders/s จะมี email 150 รายการ/s ที่ต้องรอ queue — หาก consumer ดึง message เร็วกว่า rate limit อาจเกิด email backlog สะสม หรือ MailFast ตอบ HTTP 429 ซึ่งทำให้ต้องออกแบบ retry/backoff อย่างระมัดระวัง |
| **Risk** | ADR ไม่กำหนด SLA ของ Kafka เอง — หาก Kafka ล่มระหว่าง `checkoutservice` กำลัง publish event order จะ fail หรือไม่? | ADR ระบุว่า "deploy Kafka แบบ cluster" แต่ไม่ได้กำหนด availability target หรือ behavior ของ `checkoutservice` เมื่อ Kafka unavailable (เช่น ยังคืน success หรือ fail?) |
| **Non-risk** | การแยก post-order actions ออกเป็น consumer แต่ละตัว (email, loyalty, analytics) ช่วยให้เพิ่ม action ใหม่โดยไม่แก้ `checkoutservice` | สอดคล้องกับ requirement ที่ marketing คาดว่าจะมี action เพิ่มปีหน้า — modifiability ดี |
| **Non-risk** | การ publish event แทน synchronous call แก้ปัญหา latency ได้โดยตรง | MailFast p95 1.5 s จะไม่อยู่ใน critical path ของ `PlaceOrder` อีกต่อไป — ตอบ QAS-1 ได้ชัดเจน |
| **Sensitivity point** | ความพร้อมใช้งาน (availability) ของ Kafka cluster | หาก Kafka ล่มและ `checkoutservice` ไม่มี fallback (เช่น local queue หรือ circuit breaker) post-order actions ทั้งหมดจะหยุดชะงัก หรือ order จะ fail — quality attribute ด้าน availability ขึ้นอยู่กับ Kafka โดยตรง |
| **Sensitivity point** | Idempotency implementation ของ `loyaltyservice` | ใช้ `order_id` เป็น key ถูกต้อง แต่ต้องใช้ database-level unique constraint หรือ transactional check — implementation ที่ผิดพลาด (เช่น check-then-insert แบบ non-atomic) ทำให้ duplicate points เกิดได้ |
| **Trade-off point** | Kafka vs RabbitMQ: ความสามารถด้าน analytics และ throughput สูงกว่า แต่แลกกับ operational complexity ที่สูงกว่าสำหรับทีม 4 คนที่ไม่มีประสบการณ์ Kafka | Performance และ analytics (Kafka) ↔ Operability และ familiarity (RabbitMQ) — เป็น trade-off ที่ ADR ไม่ได้วิเคราะห์ให้ชัดเจน |

---

## 3. Missing or Weak

- **Option ที่ไม่ได้พิจารณา:** RabbitMQ ซึ่งทีม 2 คนมีประสบการณ์จริง ถูกระบุเป็น Option 2 แต่ถูกตัดออกโดยไม่มีการเปรียบเทียบโดยตรงกับ Kafka ใน Decision section — การเลือก Kafka แทน RabbitMQ ขาด rationale ที่ชัดเจน (เช่น ทำไม analytics ต้องการ Kafka โดยเฉพาะ ทั้งที่ RabbitMQ + analytics consumer ก็ทำได้)
- **Consequence ที่ขาดไป:** ADR ไม่ระบุว่า `checkoutservice` จะทำอย่างไรเมื่อ Kafka unavailable ขณะ publish — ยังคืน success (at-risk of losing events) หรือ fail (กระทบ availability requirement)?
- **Claim ที่ไม่มี evidence รองรับ:** "publish event ใช้เวลาน้อยกว่า 10 ms" — ไม่มีการอ้างอิง benchmark หรือ Kafka producer configuration (synchronous vs asynchronous ack) ที่รองรับตัวเลขนี้ หากใช้ `acks=all` latency อาจสูงกว่านี้มาก
- **Rate limit MailFast:** ADR ระบุว่าต้องการ client-side rate limiting แต่ไม่ได้ระบุ mechanism (token bucket, leaky bucket) หรือ behavior เมื่อ queue backlog ยาวเกิน SLA 1 นาทีของ loyalty points

---

## 4. Verdict

**Accept with changes**

สามเปลี่ยนแปลงสำคัญที่ต้องทำก่อน approve:

1. **อธิบาย rationale ของการเลือก Kafka แทน RabbitMQ ให้ชัดเจน** — โดยเฉพาะเมื่อทีมมีประสบการณ์ RabbitMQ อยู่แล้ว หาก Kafka ถูกเลือกเพราะ analytics use case ให้ระบุว่า use case นั้นต้องการ log retention หรือ replay ที่ RabbitMQ ทำไม่ได้จริง
2. **กำหนด behavior ของ `checkoutservice` เมื่อ Kafka unavailable** — ชัดเจนว่าจะ fail fast หรือ accept แล้ว risk losing event พร้อม mitigation (เช่น Outbox Pattern lite หรือ local retry buffer)
3. **ระบุ Kafka producer configuration และ MailFast rate limiting mechanism** ให้ชัดเจน — claim "< 10 ms" ต้องมี ack mode รองรับ และ rate limiting ต้องระบุ algorithm และ behavior เมื่อ backlog เกิน SLA
