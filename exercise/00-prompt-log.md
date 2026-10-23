**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:**
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
**โมเดล:** …
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: …

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Kiro Agent (Claude sonnet) | `prompts/1-…` run A | system/online-boutique/protos/demo.proto | ใช่ | 4 | context ยังมีน้อย ทำให้ข้อมูลไม่ละเอียดเพียงพอ |
| 2 | 1-B | Kiro Agent (Claude sonnet) | `prompts/1-…` run B | system/online-boutique/kubernetes-manifests/adservice.yaml, system/online-boutique/kubernetes-manifests/cartservice.yaml, system/online-boutique/kubernetes-manifests/checkoutservice.yaml, system/online-boutique/kubernetes-manifests/currencyservice.yaml, system/online-boutique/kubernetes-manifests/emailservice.yaml, system/online-boutique/kubernetes-manifests/frontend.yaml, system/online-boutique/kubernetes-manifests/kustomization.yaml, system/online-boutique/kubernetes-manifests/loadgenerator.yaml, system/online-boutique/kubernetes-manifests/paymentservice.yaml, system/online-boutique/kubernetes-manifests/productcatalogservice.yaml, system/online-boutique/kubernetes-manifests/recommendationservice.yaml, system/online-boutique/kubernetes-manifests/shippingservice.yaml | ใช่ | 4 | มีข้อมูลที่เชื่อมโยงและเห็นภาพมากขึ้น|
| 3 | 2-a | Kiro agent (Claude sonnet) | `prompts/2-…` run a | none | ใช่ ตรงตามขอบเขตที่อนุญาตให้โมเดลเข้าถึงได้ | 3 | มีเพียงข้อมูลที่ตาม requirement แต่ยังมี context ไม่เพียงพอที่จะนำเอาไปใช้งานได้ทันที |
| 4 | 2-b | Kiro agent (Claude sonnet) | `prompts/2-…` run b | docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go | ใช่ ตรงตามขอบเขตที่อนุญาต | 5 | มีข้อมูลครบถ้วน และการตัดสินใจขึ้นกับข้อมูลที่มีอย่างตรงความเป็นจริง |
| 5 | 3 | Kiro agent (Claude sonnet) | `prompts/3-…` | scenario.md, 3-review.md | ใช่ ตรงตามขอบเขตที่อนุญาต | 5 | มีข้อมูลครบถ้วน และการตัดสินใจขึ้นกับข้อมูลที่มีอย่างตรงความเป็นจริง |
| 6 | bonus | Kiro agent (Claude sonnet) | prompts/bonus-recover-decision.md | system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go | ใช่ ตรงตามขอบเขตที่อนุญาต | 4 | มีข้อมูลการ recover ADR |
| 7 | | | | | | | |
