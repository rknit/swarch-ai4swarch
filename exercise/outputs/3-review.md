Agent: Claude Code, model Sonnet 5.5 (`claude-sonnet-5-5`) | reviewed: `docs/adr/0002-use-rabbitmq-for-order-placed-events.md` | language: Thai

# Review ของ ADR-0002 (สไตล์ ATAM)

อ่านเฉพาะ ADR และ `docs/scenario.md` เท่านั้น จึงไม่ได้ตรวจเลขบรรทัดใน `main.go` ที่ ADR อ้างถึง

## 1. Quality-attribute scenarios

| # | Source · Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|
| S1 (Performance) | ลูกค้าเรียก `PlaceOrder` | 11.11, 200 orders/s, MailFast p99 8 s | `checkoutservice` ชำระเงิน จัดส่ง publish `OrderPlaced` แล้วตอบทันที | p95 `PlaceOrder` ≤ 500 ms (baseline ประมาณ 300 ms) |
| S2 (Availability) | RabbitMQ, MailFast, `loyaltyservice` หรือ analytics ล่ม | ช่วง peak | order ยังสำเร็จ | order ล้มเพราะสาเหตุเหล่านี้ = 0 |
| S3 (Correctness) | order ถูกชำระแล้วและ broker ส่ง message ซ้ำ | ช่วง peak, ปกติ | ให้แต้มเพียงครั้งเดียว | แต้มปรากฏภายใน 1 นาที, ซ้ำ = 0, ตกหล่น = 0 |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | ถ้า publish ล้มเหลวหลัง `chargeCard` ADR ให้ "log แล้วตอบสำเร็จ" event จึงหายเงียบ ๆ และยังไม่มีทางกู้คืน (outbox ถูกเลื่อนไป Open questions) | ลูกค้าจ่ายเงินแล้วแต่ไม่ได้แต้ม ขัดกับ S3 (ภายใน 1 นาที, ไม่ตกหล่น) และขัดกับคำอ้างว่า "ตอบ driver 1–3 พร้อมกัน" |
| **Risk** | อีเมลสะสม 150/s ตลอด peak (200 − 50) หาก peak ยาว T ระบบจะระบายหมดหลัง peak อีก 3T (เช่น peak 1 ชม. ค้างประมาณ 540,000 ฉบับ ระบายอีก 3 ชม.) ไม่มี SLA อีเมล, ขนาด queue หรือ TTL | queue โตไม่จำกัดกระทบ memory/disk ของ broker ที่ใช้ร่วมกับ loyalty; ลูกค้าได้อีเมลช้าหลายชั่วโมง |
| **Risk** | การคุมเพดาน 50 emails/s ระบุไว้ที่ "email consumer" แต่ RabbitMQ ไม่บังคับ rate limit ข้าม replica; ถ้า scale consumer หลาย pod จะเกิน limit ของ MailFast. retry (ประมาณ 0.5 %) ก็กินโควต้าเดียวกัน | ถูก MailFast throttle/ปฏิเสธ ทำให้ backlog แย่ลง |
| **Risk** | idempotency ถูกออกแบบเฉพาะ loyalty; email consumer ที่ retry/redeliver อาจส่งอีเมลซ้ำ และ `loyaltyservice` ยังไม่มี data store ที่รองรับ dedup แบบ atomic (ค้างใน Open questions) | exactly-once effect เป็นจริงได้เมื่อมี unique constraint บน `order_id` เท่านั้น |
| **Sensitivity point** | latency ของ publish (publisher confirm + "retry สั้น ๆ") อยู่ใน critical path โดยตรง และงบเหลือประมาณ 200 ms | broker ช้าหรือ confirm timeout ทำให้ S1 พังได้ แต่ ADR ไม่กำหนด timeout และบอกเองว่า "ไม่ทราบค่าจริง" |
| **Trade-off point** | "order ต้องไม่ล้ม" (ยอมทิ้ง event) ชนกับ "แต้มต้องไม่ตกหล่น" (ต้อง block หรือมี outbox) | ADR เลือกด้าน availability โดยไม่ระบุว่ายอมแลกความถูกต้องของแต้มเท่าไร |
| **Non-risk** | แยก queue ต่อ consumer (email, loyalty, analytics): ความช้าของอีเมลไม่ขวาง loyalty จึงรักษา SLA 1 นาทีของ loyalty ได้แม้ email backlog ยาว | ตรงกับ S2/S3 โดยตรง ควรบังคับเป็นข้อกำหนด |
| **Non-risk** | ปฏิเสธ Option 1 และ 2 ด้วยเหตุผลที่ตรวจสอบได้: sync 1.5 s เกินงบ 200 ms; goroutine เสียงานเมื่อ pod restart | ตัวเลขมาจาก scenario จริง |
| **Non-risk** | at-least-once + idempotency key `order_id` (UUID สร้างใน `PlaceOrder`) เป็นแนวทางที่ถูกต้องสำหรับ exactly-once effect | สอดคล้อง S3 หากมี dedup store |

## 3. Missing or weak

- **ไม่ได้พิจารณา:** transactional outbox หรือ durable local buffer เป็น option เต็มรูปแบบ (ADR ปัดตกเพราะ `checkoutservice` ไม่มี data store ทั้งที่เป็นต้นทุนที่ประเมินได้); broker ทางเลือกที่เบากว่า (NATS, Redis Streams); service กลาง "post-order" ตัวเดียวที่รับ gRPC แล้วจัดการ fan-out เอง
- **Option 4 ถูกปัดตกด้วยข้อความที่เกินจริง:** ADR ว่า "ขัดกับ … 'ไม่ผูก cloud-only'" แต่ scenario เขียนเพียงว่า company **prefers** open-source self-hosted ไม่ใช่ข้อห้าม และ "ต้นทุนไม่ทราบ" ใช้ปัดตกไม่ได้ เพราะ option 3 ก็ไม่ได้ประเมินต้นทุน operate
- **คำอ้างเรื่อง modifiability เกินจริง:** ADR เขียนเองว่า fraud scoring อาจต้อง sync ก่อนยืนยัน order ซึ่งอยู่นอกโมเดลนี้ และไม่พูดถึง schema evolution/versioning ของ event (consumer ใหม่อาจต้องการ field เพิ่ม ทำให้ต้องแก้ publisher)
- **คำอ้างด้านความเป็นส่วนตัว:** event มีอีเมลลูกค้าและรายการสินค้า ถูกเก็บใน queue/DLQ แต่ ADR ยังไม่กำหนดการเข้ารหัสหรืออายุข้อมูล (อยู่ใน Open questions เท่านั้น)
- **Mitigation ยังคลุมเครือ:** "cluster/persistent queue" ไม่ระบุ durable/quorum queue, persistent message, ขนาดและ resource ที่ทีม 4 คนต้องดูแลเอง (ADR ยอมรับว่าไม่มีข้อมูล)
- **Consequence แรกเขียนปนกัน:** "Positive" แต่เนื้อหาคือ "ไม่ทราบค่าจริง" จึงยังไม่ใช่ผลที่พิสูจน์แล้ว
- ประโยคสรุป "เป็นตัวเลือกเดียวที่ตอบ driver 1–3 พร้อมกัน" ไม่ถูกต้องตราบที่ driver 3 ยังไม่ปิดช่องเรื่อง event หาย

## 4. Verdict: **Accept with changes**

ทิศทาง (async ผ่าน broker, queue แยกต่อ consumer, idempotency ด้วย `order_id`) ถูกต้องและสอดคล้องกับ scenario แต่ยังมีช่องว่างสำคัญ 3 ข้อที่ต้องแก้ก่อนเปลี่ยน Status จาก Proposed

1. **ปิดช่อง event หาย:** เลือกกลไกหนึ่งอย่าง (outbox หรือ reconciliation job ที่ตรวจ order ที่ไม่มีแต้ม) และกำหนด timeout/retry ของ publish ภายในงบ 200 ms
2. **คำนวณ backlog อีเมลและกำหนด SLA:** ระบุระยะเวลา peak, ขนาด/TTL ของ queue, และ rate limiter ที่ใช้ร่วมกันทุก replica (นับ retry ด้วย)
3. **ทำให้ idempotency และ durability ชัดเจน:** ระบุ dedup store ของ `loyaltyservice` (unique บน `order_id`), การกันอีเมลซ้ำ, durable/quorum queue พร้อม persistent message และแก้เหตุผลปัดตก Option 4 ให้ตรงกับ scenario
