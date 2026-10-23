> **Agent:** Kiro (Amazon Kiro IDE) — Model: Auto (server-selected)

# ADR-BONUS: ใช้ gRPC และ proto ไฟล์ร่วมกัน (demo.proto) สำหรับการสื่อสารระหว่าง service ทุกตัว

- **Status:** Reconstructed
- **Date:** 2026-10-21
- **Deciders:** ทีม Online Boutique (Google Cloud Platform) — ไม่ปรากฏชื่อในหลักฐาน

---

## Context and problem

Online Boutique เป็นระบบ e-commerce แบบ microservices ที่ประกอบด้วย service อย่างน้อย 10 ตัว
(CartService, CheckoutService, ProductCatalogService, ShippingService, CurrencyService, PaymentService, EmailService, RecommendationService, AdService, Frontend)
แต่ละตัวเขียนด้วยภาษาที่แตกต่างกัน (Go, Python, Java, Node.js เป็นต้น)

ปัญหาที่ต้องตัดสินใจคือ: **service เหล่านี้จะสื่อสารกันด้วยโปรโตคอลและกลไก interface-definition อะไร?**

ระบบต้องตอบสนองต่อ user action แบบ synchronous (เช่น การดูสินค้า การ checkout) ซึ่งต้องการ response กลับทันที
นอกจากนี้ยังเป็น demo project ของ Google ที่ใช้สำหรับสาธิตเทคโนโลยี Kubernetes และ GCP จึงมีข้อพิจารณาด้าน educational clarity ด้วย

---

## Decision drivers

- **Interoperability** — service เขียนหลายภาษา ต้องมี contract ที่ทุกภาษา generate client/server code ได้
- **Type safety** — ต้องการ strongly-typed interface เพื่อลด runtime error ข้าม service boundary
- **Developer experience** — ต้องการให้ทีมเห็น API ทุกตัวในที่เดียว (discoverability)
- **Performance** — latency ต้องต่ำพอสำหรับ synchronous call chain ใน checkout flow
- **Observability** — ต้องผสาน distributed tracing (OpenTelemetry) ได้ง่าย

---

## Considered options

1. **gRPC + Protobuf (single shared demo.proto)**
2. **REST/HTTP JSON พร้อม OpenAPI spec แยกต่าง service**
3. **Message broker (async) เช่น Kafka / RabbitMQ**

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| gRPC + Protobuf (single proto) | strongly-typed contract, code generation หลายภาษา, HTTP/2 (multiplexing, header compression), integration กับ OpenTelemetry built-in, API ทุกตัวอยู่ในไฟล์เดียว | ต้อง compile proto ก่อนใช้งาน, binary format debug ยากกว่า JSON, ยังเป็น synchronous จึง cascade failure ได้ |
| REST/HTTP JSON | debug ง่าย, เครื่องมือมาก, ไม่ต้อง compile | ไม่มี schema enforcement, boilerplate มาก, overhead ของ JSON parsing |
| Message broker (async) | decoupled, resilient, ดีสำหรับ eventually-consistent flow | ซับซ้อนกว่า, ไม่เหมาะกับ flow ที่ต้องการ response ทันที เช่น "ราคาเท่าไร" |

---

## Decision

เลือก **gRPC กับ Protobuf** โดยนิยาม service และ message ทั้งหมดรวมไว้ใน `demo.proto` ไฟล์เดียว

เหตุผลหลักคือ system มีทั้ง multi-language service และ synchronous request-response flow ที่ต้องการ
strongly-typed contract gRPC ตอบโจทย์ทั้งสองข้อพร้อมกัน การรวม proto ทุก service ไว้ในไฟล์เดียว
ทำให้ทีมเห็นภาพรวม API ของ system ได้ทันที ซึ่งเหมาะกับ demo/educational project เป็นพิเศษ

---

## Consequences

- **Positive:** type-safe cross-language API (performance: ลด runtime error); HTTP/2 ช่วยเรื่อง latency
  (performance); ผสาน OpenTelemetry ผ่าน `otelgrpc` handler ได้โดยตรง (observability)
- **Negative:** service ทุกตัว depend บน `demo.proto` ไฟล์เดียว — การเปลี่ยน message ใด ๆ
  กระทบทุก service พร้อมกัน (modifiability); ทุก call เป็น synchronous จึงเกิด cascade failure ได้
  หาก service ใด service หนึ่งล่ม (availability)
- **Risks and mitigations:** Cascade failure — checkoutservice มี error propagation แบบ fail-fast
  (return gRPC error ขึ้นไป) แต่ไม่มี circuit breaker ปรากฏในโค้ด **[Inferred]**;
  proto evolution — ต้องระวัง breaking change เช่น เปลี่ยน field number

---

## Validation

ตรวจสอบได้โดย: (1) ดู generated code ใน `genproto/` ของแต่ละ service ว่า import จาก demo.proto เดียวกัน
(2) integration test ผ่าน load generator; (3) ดู distributed trace บน Jaeger/Cloud Trace ว่า span ต่อเนื่องข้าม service

---

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | service ทุกตัวสื่อสารด้วย gRPC | Evidenced | `demo.proto` line 1–end: ทุก service (CartService, CheckoutService ฯลฯ) นิยามเป็น `service` block ใน proto; `checkoutservice/main.go`: `mustConnGRPC(...)` เรียก service อื่นทุกตัว; `frontend/main.go`: `mustConnGRPC(...)` ต่อทุก downstream service |
| 2 | มี proto ไฟล์เดียว (demo.proto) ที่ใช้ร่วมกันทุก service | Evidenced | `demo.proto`: มี `service` block ครบทุก service (CartService, CheckoutService, ProductCatalogService, ShippingService, CurrencyService, PaymentService, EmailService, RecommendationService, AdService) อยู่ในไฟล์เดียว |
| 3 | การสื่อสารเป็นแบบ synchronous | Evidenced | `checkoutservice/main.go` ฟังก์ชัน `PlaceOrder`: เรียก `getUserCart`, `prepOrderItems`, `chargeCard`, `shipOrder`, `sendOrderConfirmation` ตามลำดับ รอ response แต่ละขั้นก่อนดำเนินต่อ (sequential blocking calls) |
| 4 | เลือก gRPC เพราะต้องการ strongly-typed contract ข้าม multi-language service | Inferred | หลักฐานแสดงว่า service ใช้หลายภาษา (ชื่อ service และ import แตกต่างกัน) และใช้ proto-generated types เช่น `pb.NewCartServiceClient`, `pb.PlaceOrderRequest` แต่ไม่มีเอกสารอธิบายเหตุผลการเลือก gRPC ตรง ๆ |
| 5 | เลือก proto ไฟล์เดียวเพื่อ discoverability / ความง่ายของ demo project | Inferred | ไม่มีเอกสารระบุเหตุผล ทราบเพียงว่ามีไฟล์เดียวจากหลักฐาน; การรวมไว้ไฟล์เดียวอาจสะท้อน educational intent ของโปรเจกต์ แต่ไม่ได้ระบุในโค้ด |
| 6 | ใช้ OpenTelemetry (`otelgrpc`) สำหรับ distributed tracing | Evidenced | `frontend/main.go` import `go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc` และ `mustConnGRPC` ส่ง `grpc.WithStatsHandler(otelgrpc.NewClientHandler())`; `checkoutservice/main.go` เช่นเดียวกัน |
| 7 | ไม่มี circuit breaker / retry logic ใน checkout flow | Evidenced | `checkoutservice/main.go`: ทุก helper function (เช่น `chargeCard`, `shipOrder`) คืน error ทันทีโดยไม่มี retry loop หรือ circuit breaker library ปรากฏในโค้ด |
| 8 | cascade failure อาจเกิดขึ้นหาก downstream service ล่ม | Inferred | เป็นผลต่อเนื่องตรงของ synchronous call chain ที่ไม่มี circuit breaker แต่ไม่มีหลักฐานที่บันทึก failure scenario ไว้ |
| 9 | การรวม proto ไฟล์เดียวทำให้ทุก service ต้อง compile ใหม่เมื่อ spec เปลี่ยน | Assumption | เป็นลักษณะทั่วไปของ monolithic proto file ในระบบ gRPC — ไม่มีหลักฐานเฉพาะจาก repo นี้ |
| 10 | gRPC ให้ latency ต่ำกว่า REST/JSON เพราะใช้ HTTP/2 และ binary encoding | Assumption | เป็น industry knowledge ทั่วไป ไม่มีการวัด benchmark ใน repo |
