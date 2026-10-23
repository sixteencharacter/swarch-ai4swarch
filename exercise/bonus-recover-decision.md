**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด *การสูญหายของความรู้* หรือ knowledge vaporization), AICH4
> **คำถาม:** เมื่อ AI อธิบายว่า*ทำไม*ระบบจึงถูกสร้างแบบนี้ คำอธิบายส่วนใดมาจากหลักฐาน และส่วนใดเป็นการเดาอย่างมั่นใจ

## ลงมือทำ

เริ่มแชตหรือ session ใหม่ แล้วส่งข้อความนี้ให้ agent

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
language: Thai
```

บรรทัด `language: Thai` ทำให้ ADR ที่กู้คืนเป็นภาษาไทย หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้

agent จะเขียน ADR ที่กู้คืนมา พร้อมตาราง **Claim check** ไว้ที่ [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md)

## ตรวจสอบตาราง claim check

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า *Evidenced* (มีหลักฐาน) อย่างน้อยสองข้อ และ *Inferred* (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| # | ข้อกล่าวอ้าง (ย่อ) | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล) |
|---|---|---|---|---|
| 1 |  service ทุกตัวสื่อสารด้วย gRPC | Evidenced | Evidenced | demo.proto |
| 2 | มี proto ไฟล์เดียว (demo.proto) ที่ใช้ร่วมกันทุก service | Evidenced | Evidenced | demo.proto |
| 3 | การสื่อสารเป็นแบบ synchronous | Evidenced | Evidenced | checkoutservice/main.gp |
| 4 | cascade failure อาจเกิดขึ้นหาก downstream service ล่ม | Inferred | Inferred | ไม่มีการ implement circuit breaker ไว้ |
| 5 | เลือก proto ไฟล์เดียวเพื่อ discoverability / ความง่ายของ demo project | Inferred | Inferred | ไม่มีการกล่าวไว้อย่างชัดเจนว่าเพื่อความเข้าใจง่าย แต่การกระทำอย่างงี้ทำให้เข้าใจง่ายขึ้น เหมือนเป็น demo project |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า *Evidenced* กี่ข้อ (5 ข้อ)

## สะท้อนคิด

1. agent เคยระบุการเดาว่าเป็น *Evidenced* หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี
> จากคำตอบที่ AI ได้ generate มาข้างต้น พบว่า ยังไม่มีการเดาและระบุว่าเป็น Evidenced โดยตรวจสอบด้วยการตามไปอ่าน code จริง ๆ ซึ่งถูกระบุรายละเอียดใน evidence 
2. หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด
> อาจจะเข้าใจผิดเกี่ยวกับ decision บางอย่างเช่นการเลือก grpc เพราะ latency ตำ่กว่า REST/JSON ด้วยการเชื่อ assumption ของ AI โดยไม่ตรวจสอบ ว่ามีหลักฐานจริง ๆ 
3. AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต
> สามารถกู้คืนการตัดสินใจที่มาจาก constraint ทาง coding ได้เช่น synchronous / asynchronous แต่ไม่สามารถกู้คืนความจำเป็นที่เกิดจากฝั่ง business / real-world usecase ได้ ยกตัวอย่างเช่น ความจำเป็นในการที่ใช้งาน kafka เนื่องจากความสามารถในการ analytics ซึ่งลดเวลา และ cost ในการพัฒนาได้ จึงทำให้ kafka ถูกเลือก