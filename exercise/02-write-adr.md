**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement:
> After an order is placed, Online Boutique must:
> 1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
> 2. award **loyalty points** through a new `loyaltyservice` (to be built);
> 3. record an **"order placed" analytics event** for the marketing team.
>
> Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement:
> After an order is placed, Online Boutique must:
> 1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
> 2. award **loyalty points** through a new `loyaltyservice` (to be built);
> 3. record an **"order placed" analytics event** for the marketing team.
>
> Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย ส่วนหัวไฟล์ระบุว่าไม่ได้อ่านไฟล์ของระบบ ตัวเลขที่ต้องใช้ถูกย้ายไปเป็น Open questions เช่น SLA, ปริมาณ order ช่วงพีก, rate limit และ SLA ของ MailFast | ใช้ครบและตรงกับ scenario: p95 `PlaceOrder` ≤ 500 ms (ปัจจุบันประมาณ 300 ms), 200 orders/s ช่วง 11.11, MailFast p95 1.5 s / p99 8 s / ล้มเหลวประมาณ 0.5 % / 50 emails/s, แต้มภายใน 1 นาที, ทีม 4 คน (2 คนเคยใช้ RabbitMQ) และคำนวณต่อเอง เช่น งบ latency เหลือประมาณ 200 ms และอีเมลค้างสะสมประมาณ 150 ฉบับต่อวินาทีช่วงพีก |
| ทางเลือกที่พิจารณา (ระบุ) | 3 ทางเลือก: (A) sync gRPC/HTTP ตรง ๆ, (B) event-driven ผ่าน message broker (เลือก), (C) post-order orchestrator service ไม่ระบุยี่ห้อ broker | 4 ทางเลือก: (1) sync gRPC ต่อท้าย `PlaceOrder`, (2) fire-and-forget goroutine ใน `checkoutservice`, (3) RabbitMQ self-hosted + event `OrderPlaced` (เลือก), (4) managed cloud messaging ระบุ RabbitMQ ชัดเจน |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | **แต้ม:** consumer ต้อง idempotent โดยใช้ order id เป็น key, ทดสอบ duplicate delivery ใน Validation แต่ไม่ได้อ้างข้อกำหนด exactly once หรือ 1 นาที<br>**Rate limit:** พูดถึงแค่ "MailFast ล่มหรือถูก rate-limit → retry พร้อม backoff, alert เมื่อ queue สะสม" และถาม rate limit ใน Open questions เพราะไม่ทราบค่า | **แต้ม:** `loyaltyservice` ให้แต้มแบบ idempotent โดยอ้าง `order_id` (UUID จาก `PlaceOrder`, `main.go` บรรทัด 233) ข้ามถ้าเคยให้แล้ว จึงได้ exactly-once effect บน at-least-once delivery และมี Validation ตรวจ redelivery กับเกณฑ์ 1 นาที<br>**Rate limit:** email consumer ส่งไม่เกิน 50 emails/s, ส่วนเกินรอใน queue, retry ความล้มเหลว 0.5 %, monitor ความลึกของ queue และตรวจ backlog หลังพีก |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่มีตัวเลขหรือข้อเท็จจริงของระบบที่แต่งขึ้น ข้อสมมติติดป้าย **[Assumption]** เช่น order flow อยู่ใน service เดียวและมี `emailservice` แยก<br>มีเนื้อหาที่เติมเองและไม่ได้มาจาก scenario: transactional outbox, DLQ, ทบทวน ADR "เมื่อครบ 6 เดือนหลัง deploy" และ p95 เป็นตัววัดใน Validation<br>ถามข้อมูลที่ scenario มีให้แล้ว เช่น ทีมมีทักษะ operate broker หรือไม่, ปัจจุบันมี broker หรือไม่ | ไม่พบตัวเลขที่แต่งขึ้น ตัวเลขทั้งหมดตรงกับ scenario หรือคำนวณจากมัน และส่วนที่ไม่มีข้อมูลเขียนว่า "Not evidenced" / "ไม่มีข้อมูล" / **[Inferred]**รายละเอียดโค้ดมาจาก `main.go` (บรรทัด 66–123, 233, 252–279)<br>เติมเองเล็กน้อย: ตัวอย่าง "Pub/Sub" ในตัวเลือก 4 และคำว่า "scale ได้" ของ managed messaging |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เปลี่ยนจาก "publish ตรงแล้ว log ถ้าล้มเหลว" (ร่าง AI เสนอ outbox เป็นแค่ทางเลือกถ้ายอมรับการสูญหายไม่ได้) เป็นใช้ **transactional outbox** ที่ producer เป็นเจ้าของ พร้อม dead-letter ของ producer เอง ชนิด data store ไม่บังคับ (document-based ได้เพราะรูปทรง event ไม่คงที่) และยอมรับว่า event อาจหายถ้าเขียน outbox ล้มเหลว | event สำคัญต้องไม่หายง่ายหลังชำระเงินแล้ว (reliability) และแต่ละ producer ดูแลของตัวเองได้โดยไม่ผูกกันข้าม service (modifiability) |
| 2 | เปลี่ยน fraud scoring จากคำถามค้างว่าต้องทำก่อนยืนยันหรือไม่ เป็น **async** และย้าย `shipOrder` ออกจาก `PlaceOrder` ให้จัดส่งได้หลัง order valid (ยืนยันแล้ว + ผ่าน fraud) เท่านั้น เพิ่ม cancel path (ยกเลิก order, แจ้งอีเมลผ่าน outbox/dead-letter, เริ่มคืนเงิน) และกำหนดเวลาสูงสุดของ fraud scoring แบบ fail open | ไม่ให้ fraud scoring เพิ่ม latency ของ checkout (performance) แต่ยังไม่ส่งของก่อนตรวจ; สมมติว่าอัตรา fraud ต่ำ จึงยอมรับความเสี่ยงของ fail open; ถ้า `paymentservice` ไม่รองรับการคืนเงินให้ทบทวนด้วย ADR ใหม่ |
| 3 | เพิ่มสมมติฐาน peak อย่างน้อย 30 นาที (คำนวณ backlog อีเมลประมาณ 270,000 ฉบับ ระบายประมาณ 90 นาที), กำหนดให้ retry ของอีเมลเป็น exponential backoff แบบมีเพดาน ส่วนที่ล้มเหลวไป dead-letter และใช้ cron job ตรวจ dead-letter แทนการปล่อยเป็นคำถามค้างทั้งหมด | scenario ไม่ระบุระยะเวลา peak จึงต้องมีฐานสำหรับประเมิน backlog เทียบ rate limit 50 emails/s (performance, availability); ค่า retry และเวลา fraud scoring ที่แน่นอนยังต้องวิเคราะห์เพิ่ม จึงคงไว้ใน Open questions |
