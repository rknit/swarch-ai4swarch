Agent: Claude Code, model Sonnet 5.5 (`claude-sonnet-5-5`) | run: b (with context) | language: Thai

# ADR-0002: ใช้ RabbitMQ เผยแพร่ event `OrderPlaced` เพื่อทำงานหลังสั่งซื้อแบบ asynchronous

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** รชต

## Context and problem

Requirement R-12 ต้องการให้หลังสั่งซื้อสำเร็จ ระบบ (1) ส่งอีเมลยืนยันผ่าน **MailFast** แทน logic mock ของ `emailservice` (2) ให้แต้มสะสมผ่าน `loyaltyservice` ที่ต้องสร้างใหม่ (3) บันทึก analytics event "order placed" และฝ่ายการตลาดคาดว่าปีหน้าจะมี post-order action เพิ่ม (SMS, fraud scoring, อัปเดต recommendations) (`docs/scenario.md`)

**ปัจจุบัน:** `checkoutservice.PlaceOrder` เรียก `paymentservice` (`chargeCard`), `shippingservice` (`shipOrder`) แล้วเรียก `emailservice` (`sendOrderConfirmation`) แบบ synchronous ผ่าน gRPC ตามลำดับ ถ้าอีเมลล้มเหลวจะ log warning แล้วคืนผลสำเร็จตามปกติ (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder` บรรทัด 252–279; `docs/scenario.md`) ที่อยู่ของแต่ละ downstream ถูกผูกใน `checkoutService` struct และ `mustMapEnv`/`mustConnGRPC` ตอนเริ่มระบบ (`main.go` บรรทัด 66–123) ดังนั้นการเพิ่ม consumer ใหม่แต่ละตัวแบบ synchronous หมายถึงแก้โค้ดและ deploy `checkoutservice` ใหม่ทุกครั้ง

**ข้อเท็จจริงที่มีผลต่อการตัดสินใจ** (`docs/scenario.md`):

| หัวข้อ | ข้อเท็จจริง | ผลต่อการออกแบบ |
|---|---|---|
| Checkout latency | p95 ของ `PlaceOrder` ≤ 500 ms (ปัจจุบันประมาณ 300 ms) | งบเหลือประมาณ 200 ms |
| MailFast | p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5 %, rate limit 50 emails/s | เรียกแบบ sync ใน `PlaceOrder` เกินงบ latency ทันที (1.5 s > 200 ms) |
| Peak load | 200 orders/s ช่วง 11.11 | ส่งอีเมลได้ 50/s จึงต้องมี buffer รองรับ (ดูหมายเหตุด้านล่าง) |
| Loyalty | ภายใน 1 นาที, exactly once | ยอมให้ช้าได้ แต่ห้ามให้แต้มซ้ำ |
| Availability | order ต้องไม่ล้มเพราะ email/loyalty/analytics ล่ม | ต้องแยก failure ของ post-order ออกจากเส้นทางหลักของ `PlaceOrder` |
| Team | 4 คน, ถนัด Go และ gRPC, 2 คนเคยใช้ RabbitMQ | |
| Operations | รันบน Kubernetes อยู่แล้ว, ชอบ open-source self-hosted มากกว่า managed cloud-only | |

หมายเหตุ: ที่ 200 orders/s กับ rate limit 50 emails/s อีเมลสะสมค้างประมาณ 150 ฉบับต่อวินาทีตลอดช่วง peak (ถ้าทุก order ส่งอีเมล) ระยะเวลา peak ที่แน่นอนและ SLA ของอีเมลไม่ได้ระบุ (ดู Open questions)

## Decision drivers

เรียงตามความสำคัญ:

1. **Availability:** order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม
2. **Performance:** p95 `PlaceOrder` ≤ 500 ms แม้ MailFast มี p95 1.5 s / p99 8 s
3. **Correctness (reliability):** loyalty ให้แต้ม exactly once ภายใน 1 นาที
4. **Modifiability:** post-order action ใหม่ (SMS, fraud scoring, recommendations) ต้องเพิ่มได้โดยไม่แก้ `checkoutservice` ซ้ำ ๆ
5. **Throughput:** รับ 200 orders/s ขณะที่ MailFast จำกัด 50 emails/s
6. **Cost / operability:** ทีม 4 คน, ใช้ open-source self-hosted บน Kubernetes, ใช้ทักษะที่มี (Go, gRPC, RabbitMQ บางส่วน)

## Considered options

1. ขยาย synchronous gRPC ใน `PlaceOrder`: เรียก MailFast adapter, `loyaltyservice` และ analytics ตามลำดับ (ต่อจากรูปแบบเดิมของ `emailservice`)
2. เรียกแบบ fire-and-forget ใน `checkoutservice` เอง: เปิด goroutine เรียก gRPC ทั้งสามหลังตอบ response (ไม่มี broker)
3. เผยแพร่ event `OrderPlaced` ไป **RabbitMQ** (self-hosted บน Kubernetes) แล้วให้ consumer แต่ละตัวรับไปทำงานของตนเอง (email, loyalty, analytics) โดยใช้ `order_id` เป็น idempotency key
4. ใช้ managed cloud messaging (เช่น Pub/Sub หรือบริการเทียบเท่า)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC ต่อท้าย `PlaceOrder` | เรียบง่ายที่สุด ตรงกับรูปแบบเดิม ทีมถนัด gRPC ไม่มี component ใหม่ | MailFast p95 1.5 s ทำให้ p95 ของ `PlaceOrder` เกิน 500 ms (ประมาณ 300 ms + 1.5 s); p99 8 s กระทบ checkout ตรง ๆ; ถ้าไม่จัดการ error ทุกจุด order อาจล้มเมื่อ downstream ล่ม; ไม่มี buffer เมื่อเกิน 50 emails/s; ทุก action ใหม่ต้องแก้และ deploy `checkoutservice` |
| 2. Fire-and-forget goroutine | latency ของ `PlaceOrder` ไม่ถูกกระทบ; ไม่มี component ใหม่; order ไม่ล้มเพราะ downstream | งานหายถ้า pod ของ `checkoutservice` restart ก่อนส่งสำเร็จ จึงรับประกัน "loyalty ภายใน 1 นาที" ไม่ได้; ไม่มี retry/ buffer/ rate limiting ในตัว (ต้องเขียนเอง); exactly once ทำเองยาก; ยังต้องแก้ `checkoutservice` ทุกครั้งที่เพิ่ม action |
| 3. RabbitMQ + event `OrderPlaced` | แยก failure ออกจาก `PlaceOrder` (availability); latency เพิ่มเพียงการ publish; queue เก็บงานข้ามการ restart และเป็น buffer ให้ email consumer ที่ถูกจำกัด 50/s; retry/ dead-letter ต่อ consumer; เพิ่ม consumer ใหม่ (SMS, fraud, recommendations) โดยไม่แก้ `checkoutservice` (modifiability); open-source, self-hosted บน Kubernetes; 2 ใน 4 คนเคยใช้ | เพิ่ม component stateful ที่ต้องดูแล (broker เป็นจุดล้มเหลวใหม่); delivery แบบ at-least-once จึงต้องทำ consumer ให้ idempotent; ผลลัพธ์เป็น eventual consistency (อีเมล/แต้มไม่เกิดทันที); event อาจหายถ้า publish ล้มเหลวหลัง commit การชำระเงิน (dual-write); ทีมที่เหลืออีก 2 คนต้องเรียนรู้ |
| 4. Managed cloud messaging | ไม่ต้องดูแล broker เอง; scale ได้ | ขัดกับความต้องการ open-source self-hosted และ "ไม่ผูก cloud-only" (`docs/scenario.md`); ผูกกับผู้ให้บริการ; ต้นทุนไม่ทราบ (ไม่มีข้อมูล) |

## Decision

เลือก **ตัวเลือก 3: เผยแพร่ event `OrderPlaced` ไป RabbitMQ (self-hosted บน Kubernetes)**

รายละเอียดการตัดสินใจ:

- `checkoutservice` publish event `OrderPlaced` (มี `order_id`, `user_id`, email, รายการสินค้า และยอดรวม ตามข้อมูลที่ `PlaceOrder` มีอยู่ใน `orderResult`) หลัง `chargeCard` และ `shipOrder` สำเร็จ แทนการเรียก `sendOrderConfirmation` แบบ sync โดย `order_id` (UUID ที่สร้างใน `PlaceOrder`, `main.go` บรรทัด 233) ใช้เป็น idempotency key
- **ความล้มเหลวของการ publish ต้องไม่ทำให้ order ล้ม:** ถ้า publish ไม่สำเร็จ ให้ log และตอบ order สำเร็จ ตามหลักเดียวกับที่ `PlaceOrder` ปฏิบัติต่ออีเมลอยู่แล้ว (`main.go` บรรทัด 273–277) รายละเอียดการกู้คืน event ที่หายไปดู Open questions
- แต่ละ post-order action เป็น consumer ของตัวเองบน queue แยก:
  - **email consumer** เรียก MailFast ด้วยอัตราไม่เกิน 50 emails/s และ retry เมื่อล้มเหลว (ประมาณ 0.5 %) ส่วนที่เกินจะรออยู่ใน queue
  - **`loyaltyservice`** ให้แต้มแบบ idempotent โดยอ้าง `order_id` (ปฏิเสธ/ข้ามถ้าเคยให้แต้มของ order นี้แล้ว) จึงได้ exactly-once effect บน delivery แบบ at-least-once
  - **analytics consumer** บันทึก event "order placed"
- action ในอนาคต (SMS, fraud scoring, recommendations) เพิ่มเป็น consumer/queue ใหม่ที่ bind กับ event เดิม โดยไม่ต้องแก้ `checkoutservice`

เหตุผล: เป็นตัวเลือกเดียวที่ตอบ driver 1–3 พร้อมกัน (ไม่ทำให้ order ล้ม, latency ไม่ขึ้นกับ MailFast, ไม่เสียงานเมื่อ pod restart) และยังรองรับ driver 4–5 (เพิ่ม consumer ได้ง่าย, queue เป็น buffer ของ rate limit) ในขณะที่เข้ากับข้อจำกัดด้าน open-source/self-hosted บน Kubernetes และทักษะของทีม

## Consequences

- **Positive:** latency ของ `PlaceOrder` แยกจาก MailFast, เพิ่มเฉพาะการ publish ซึ่งไม่ทราบค่าจริง ต้องวัด (performance)
- **Positive:** ความล้มเหลวของ email, loyalty และ analytics ไม่ทำให้ order ล้ม (availability)
- **Positive:** queue ดูดซับ burst 200 orders/s ไว้ให้ email consumer ส่งตามเพดาน 50 emails/s และ retry ต่อ consumer ได้ (performance, availability)
- **Positive:** เพิ่ม action ใหม่ด้วยการเพิ่ม consumer ไม่ต้องแก้ `checkoutservice` (modifiability)
- **Positive:** ใช้ open-source self-hosted ตามนโยบายบริษัทและทักษะ RabbitMQ ที่ทีมมีบางส่วน (cost)
- **Negative:** มี component stateful เพิ่มและเป็นจุดล้มเหลวใหม่ที่ต้อง monitor/ดูแล (operability, availability)
- **Negative:** eventual consistency: อีเมลและแต้มไม่เกิดทันทีหลังสั่งซื้อ (consistency)
- **Negative:** at-least-once delivery บังคับให้ทุก consumer ต้อง idempotent และเพิ่มความซับซ้อนในการทดสอบและ debug (modifiability)
- **Risks and mitigations:**
  - *Event หายตอน publish ล้มเหลวหลังชำระเงินแล้ว* (dual-write): ลดความเสี่ยงด้วย publisher confirms และ retry สั้น ๆ; ถ้ายอมรับไม่ได้ให้พิจารณา transactional outbox (ต้องมี data store ซึ่ง `checkoutservice` ปัจจุบันไม่มี: Not evidenced in the provided files)
  - *RabbitMQ ล่ม:* ทำ cluster/ persistent queue และตั้ง alert; ระหว่างล่ม order ยังสำเร็จแต่ post-order action ล่าช้าหรือขาด
  - *Backlog อีเมลช่วง peak:* เพิ่มการ monitor ความลึกของ queue และ alert
  - *ข้อความที่ประมวลผลไม่ได้ซ้ำ ๆ:* ใช้ dead-letter queue และ alert
  - *ทักษะ:* ให้ 2 คนที่เคยใช้ RabbitMQ เป็นผู้นำและแชร์ความรู้ให้ทีม

## Validation

- Load test ที่ 200 orders/s (ตัวเลข peak 11.11) ตรวจว่า p95 ของ `PlaceOrder` ≤ 500 ms (เทียบ baseline ประมาณ 300 ms)
- Fault injection: ทำให้ MailFast ช้า (p99 8 s) / ล้มเหลว, `loyaltyservice` และ RabbitMQ ล่ม แล้วตรวจว่า order ยังสำเร็จ (ต้อง 0 order ล้มจากสาเหตุเหล่านี้)
- ทดสอบ redelivery ซ้ำ (ส่ง message เดิมหลายครั้ง) ตรวจว่า loyalty ให้แต้มเพียงครั้งเดียวต่อ `order_id` และแต้มปรากฏภายใน 1 นาที
- ตรวจว่า email consumer ไม่เกิน 50 emails/s และ backlog หมดหลัง peak ภายในเวลาที่ธุรกิจกำหนด (รอค่าจาก Open questions)
- ทบทวน ADR นี้เมื่อเพิ่ม consumer ตัวแรกจาก SMS, fraud scoring หรือ recommendations

## Open questions

- ระยะเวลา peak ของ 11.11 คือเท่าใด และอีเมลยืนยันต้องถึงลูกค้าภายในกี่นาทีจึงยอมรับได้ (ใช้คำนวณขนาด backlog จาก 200 orders/s เทียบ 50 emails/s)
- Fraud scoring ในอนาคตต้องทำ **ก่อน** ยืนยัน order หรือไม่ (ถ้าใช่ จะเป็น synchronous step ในเส้นทาง `PlaceOrder` ไม่ใช่ post-order consumer และต้องทบทวนการตัดสินใจนี้)
- รับได้หรือไม่ที่ event จะหายในกรณี publish ล้มเหลวหลังชำระเงิน หรือต้องมี outbox/ reconciliation (ไม่มีข้อมูลเรื่อง data store ของ `checkoutservice` ในไฟล์ที่ให้มา)
- อีเมลที่ MailFast ล้มเหลว (ประมาณ 0.5 %) ควร retry กี่ครั้ง นานเท่าใด และมีช่องทางสำรองหรือไม่
- นโยบายแต้ม (อัตราการได้แต้ม, การคืนแต้มเมื่อ order ถูกยกเลิก) และ data store ของ `loyaltyservice`
- รูปแบบ analytics event (schema) และปลายทางที่ทีมการตลาดใช้ รวมถึงข้อกำหนดด้านความเป็นส่วนตัวของข้อมูลลูกค้าในอีเมล/ event
- ผลของ `emailservice` เดิม: จะเปลี่ยนให้เป็น email consumer ที่เรียก MailFast หรือแทนที่ด้วย service ใหม่ **[Inferred]** ว่าเป็นอย่างแรกเพราะ requirement ระบุว่า MailFast "replaces the mock `emailservice` logic"
- ค่า latency จริงของการ publish ไป RabbitMQ และขนาดทรัพยากร cluster ที่ต้องใช้ (ไม่มีข้อมูล)
