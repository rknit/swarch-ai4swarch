**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** รชต คณิตปัญญาเจริญ 6632184921
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: Claude Code
**โมเดล:** Claude Sonnet 5.5
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Sonnet 5.5 | `prompts/1-…` run A | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ ทุกไฟล์อยู่ในขอบเขต (ไฟล์งาน + `demo.proto` ซึ่งเป็น evidence ที่ระบุ) ไม่มีการรันคำสั่ง terminal |5  | |
| 2 | 1-B | Claude Code + Sonnet 5.5 | `prompts/1-…` run B | `prompts/1-recover-architecture.md`, `protos/demo.proto`, `kubernetes-manifests/` (13 ไฟล์: `adservice.yaml`, `cartservice.yaml`, `checkoutservice.yaml`, `currencyservice.yaml`, `emailservice.yaml`, `frontend.yaml`, `kustomization.yaml`, `loadgenerator.yaml`, `paymentservice.yaml`, `productcatalogservice.yaml`, `README.md`, `recommendationservice.yaml`, `shippingservice.yaml`) และรัน `ls` ดูรายชื่อไฟล์ใน `kubernetes-manifests/` | ใช่ ทุกไฟล์อยู่ในขอบเขต (ไฟล์งาน + `demo.proto` + โฟลเดอร์ `kubernetes-manifests/`) ข้อยกเว้น: agent รัน `ls` ซึ่งเป็นคำสั่ง terminal ที่ `AGENTS.md` ห้าม (agent รายงานเรื่องนี้ในสรุปเอง) | 4 | |
| 3 | 2-a | Claude Code + Sonnet 5.5 | `prompts/2-…` run a | `prompts/2-write-adr.md`, `docs/adr/template.md` | ใช่ ทุกไฟล์อยู่ในขอบเขต (ไฟล์งาน + template ADR; `context: none` จึงไม่อ่านไฟล์อื่น) ไม่มีการรันคำสั่ง terminal | 5 | |
| 4 | 2-b | | `prompts/2-…` run b | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | ใช่ ทุกไฟล์อยู่ในขอบเขต (ไฟล์งาน + template ADR + ไฟล์ทั้งสามที่ระบุใน `context`) ไม่มีการรันคำสั่ง terminal | 5 | |
| 5 | 3 | | `prompts/3-…` | `prompts/3-review-adr.md`, `docs/adr/0002-use-rabbitmq-for-order-placed-events.md`, `docs/scenario.md` | ใช่ ทุกไฟล์อยู่ในขอบเขต (ไฟล์งาน + ADR ที่ระบุใน `adr` + `docs/scenario.md` ซึ่งงานนี้อนุญาต) ไม่มีการรันคำสั่ง terminal และไม่ได้เปิด `main.go` | 4 | |
| 6 | | | | | | | |
| 7 | | | | | | | |
