**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| frontend → cartservice | `frontend.yaml` — env `CART_SERVICE_ADDR=cartservice:7070` | ✅ | มีการเชื่อมต่อด้วย ENV ดังกล่าวจริง |
| frontend → checkoutservice | `frontend.yaml` — env `CHECKOUT_SERVICE_ADDR=checkoutservice:5050` | ✅ | มีการเชื่อมต่อด้วย ENV ดังกล่าวจริง |
| checkoutservice → paymentservice | `checkoutservice.yaml` — env `PAYMENT_SERVICE_ADDR=paymentservice:50051` | ✅ | มีการเชื่อมต่อด้วย ENV ดังกล่าวจริง |
| cartservice → redis-cart | `cartservice.yaml` — env `REDIS_ADDR=redis-cart:6379` | ✅ | มีการเชื่อมต่อด้วย ENV ดังกล่าวจริง |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` — env `PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550` | ✅ | มีการเชื่อมต่อด้วย ENV ดังกล่าวจริง |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): …

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

> Agent ไม่สามารถที่จะเข้าใจ dependency ของการ call ระหว่าง service ต่าง ๆ สิ่งนี้เป็นเพราะว่า ใน A Agent สามารถผ่านได้แค่เพียง contract ของ gRPC service เท่านั้น แต่ B จะได้เรียนรู้การเชื่อมต่อระหว่าง service ระบบจึงสามารถสรุปข้อมูลได้ว่าการเชื่อมต่อภายในระบบเป็นอย่างไร

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

> สิ่งที่อาจจะเกิดขึ้นก็คือถ้าเอาแผนภาพนี้ส่งออกโดยไม่ได้ตรวจสอบแล้วก็คือว่า dependency ระหว่าง service และ component ต่าง ๆ อาจจะผิดได้และอาจจะทำให้สมาชิกของทีมเข้าใจผิดเกี่ยวกับการจัดวางระบบได้