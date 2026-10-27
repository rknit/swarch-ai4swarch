Agent: Claude Code, model `claude-sonnet-5-5` (run a: zero-shot, `context: none`, ไม่ได้อ่านไฟล์ของระบบ)

# ADR-0002: ใช้ event-driven (publish "OrderPlaced" ไปยัง message broker) สำหรับ post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ไม่ได้ระบุ (ดู Open questions)

## Context and problem

หลังลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำ post-order actions ดังนี้ (ตาม requirement):

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** (external email provider ใหม่) แทนตรรกะ mock ใน `emailservice`
2. ให้ **loyalty points** ผ่าน `loyaltyservice` ใหม่ (ยังไม่ได้สร้าง)
3. บันทึก **"order placed" analytics event** สำหรับทีม marketing

Marketing คาดว่าจะมี post-order actions เพิ่มปีหน้า (SMS, fraud scoring, การอัปเดต recommendations)

ปัญหาที่ต้องตัดสินใจ: **ผู้สั่งซื้อ (order flow) ควรเรียก action เหล่านี้อย่างไร** เพื่อให้เพิ่ม action ใหม่ได้ง่าย และความล้มเหลวของ action หนึ่ง (เช่น MailFast ล่ม) ไม่ทำให้การสั่งซื้อล้มเหลว

ข้อเท็จจริงจาก requirement: มี action 3 อย่างตอนนี้ และจะเพิ่มอย่างน้อย 3 อย่างในปีหน้า; MailFast เป็นระบบภายนอก; `loyaltyservice` ยังไม่มี

ข้อสมมติ **[Assumption]**: ขั้นตอน place order อยู่ใน service เดียว (ในที่นี้เรียกว่า "order-placing service") และปัจจุบันมี `emailservice` เป็น service แยก ซึ่งยังไม่ได้ตรวจสอบจากไฟล์ใด ๆ

## Decision drivers

1. **Modifiability** — เพิ่ม/เปลี่ยน/ถอด post-order action (SMS, fraud scoring, recommendations) โดยแก้ order-placing service น้อยที่สุด
2. **Availability / fault isolation** — MailFast, `loyaltyservice` หรือระบบ analytics ล่ม ต้องไม่ทำให้การสั่งซื้อล้ม
3. **Performance (latency ของการ checkout)** — ผู้ใช้ไม่ควรรอ action ที่ไม่จำเป็นต่อการยืนยันคำสั่งซื้อ
4. **Reliability / data consistency** — action ที่สำคัญ (loyalty points, อีเมล) ต้องไม่หายเงียบ ๆ
5. **Cost / operational complexity** — ต้นทุนการสร้างและดูแลโครงสร้างพื้นฐานเพิ่ม
6. **Fraud scoring ในอนาคต** — อาจต้องผลลัพธ์ก่อนยืนยันคำสั่งซื้อ (ไม่ใช่ post-order แท้) **[Assumption]**

## Considered options

1. **Option A — Synchronous gRPC/HTTP calls โดยตรง:** order-placing service เรียก MailFast adapter, `loyaltyservice` และ analytics ทีละตัว (หรือแบบขนาน) ภายใน request เดียวกัน
2. **Option B — Event-driven ด้วย message broker:** order-placing service publish event `OrderPlaced` หนึ่งครั้ง; แต่ละ consumer (email, loyalty, analytics และอื่น ๆ ในอนาคต) subscribe และประมวลผลแบบ asynchronous
3. **Option C — Post-order orchestrator service:** สร้าง service ใหม่ที่รับคำสั่ง "order placed" จาก order-placing service แล้วเรียก action ต่าง ๆ เป็นลำดับ/ขนาน พร้อม retry เอง (ยังเป็น request/response แต่ซ่อนไว้หลัง service เดียว)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A — Direct synchronous calls | ง่ายที่สุด ไม่ต้องมีโครงสร้างพื้นฐานใหม่; debug ตามลำดับการเรียกได้ตรงไปตรงมา; ผลลัพธ์รู้ทันที (เหมาะกับ fraud scoring ที่ต้องรอผล) | Coupling สูง: เพิ่ม action ทุกครั้งต้องแก้และ deploy order-placing service (modifiability ต่ำ); latency ของ checkout เพิ่มตามจำนวน action; ถ้า MailFast ช้า/ล่ม กระทบ checkout หรือต้องเขียน timeout/retry เองในทุกที่ |
| B — Event + message broker | Decoupling: เพิ่ม consumer ใหม่ (SMS, recommendations) โดยไม่แก้ผู้ publish; fault isolation ดี; checkout ตอบเร็ว; broker ช่วย retry/buffer ได้; analytics ใช้ event เดียวกันได้ | ต้องมี broker เพิ่ม (ต้นทุนและงาน operate); eventual consistency (อีเมล/points มาช้ากว่าการยืนยัน); ต้องออกแบบ idempotency และ duplicate delivery; ต้องรับประกันว่า publish ไม่หาย (เช่น outbox) ซึ่งเพิ่มความซับซ้อน; trace ปัญหายากขึ้น; ไม่เหมาะกับ action ที่ต้องรอผลก่อนยืนยันคำสั่งซื้อ |
| C — Orchestrator service | แยก logic post-order ออกจาก order-placing service จุดเดียว; ควบคุม retry/ลำดับได้ชัด; ไม่ต้องมี broker | เป็น service เพิ่มอีกตัวที่ต้องแก้ทุกครั้งที่เพิ่ม action (modifiability ปานกลาง); เป็น single point of failure ถ้าไม่ทำ async/queue เอง; ถ้าเรียกแบบ sync ปัญหา latency/availability ยังอยู่เพียงย้ายที่ |

## Decision

เลือก **Option B: event-driven ด้วย message broker** สำหรับ post-order actions ที่ไม่ต้องรอผลก่อนยืนยันคำสั่งซื้อ (อีเมลยืนยัน, loyalty points, analytics event และ SMS/recommendations ในอนาคต)

เหตุผลตาม drivers:

- **Modifiability (driver 1):** requirement ระบุชัดว่าจะมี action เพิ่มปีหน้า Option B เป็นตัวเดียวที่การเพิ่ม action = เพิ่ม consumer โดยไม่แก้ผู้ publish
- **Availability และ Performance (driver 2, 3):** การเรียกแบบ async ทำให้ MailFast หรือ `loyaltyservice` ล่มไม่กระทบการสั่งซื้อ และไม่เพิ่ม latency ของ checkout
- **Reliability (driver 4):** broker พร้อม retry และ dead-letter ช่วยไม่ให้ action สำคัญหาย โดยต้องใช้ consumer แบบ idempotent
- **Fraud scoring (driver 6):** ถ้าต้องบล็อกคำสั่งซื้อ ให้เรียกแบบ synchronous ก่อนยืนยัน (แยกออกจากการตัดสินใจนี้) ไม่ใช้ event path

การตัดสินใจนี้เป็น Proposed จนกว่าจะได้ข้อมูลใน Open questions (โดยเฉพาะ SLA ของอีเมล/points และความสามารถของทีมในการ operate broker)

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่โดยไม่แก้ order-placing service (quality attribute: modifiability)
- **Positive:** MailFast/`loyaltyservice`/analytics ล่มไม่ทำให้สั่งซื้อล้ม (quality attribute: availability)
- **Positive:** checkout ไม่ต้องรอ action ปลายทาง (quality attribute: performance)
- **Positive:** เปลี่ยนจาก mock `emailservice` เป็น MailFast ทำได้ภายใน email consumer โดยไม่กระทบ service อื่น (quality attribute: modifiability)
- **Negative:** ต้องมี message broker เพิ่มและดูแลมัน (quality attribute: cost / operability)
- **Negative:** อีเมลและ points เป็น eventual consistency ผู้ใช้อาจเห็นช้ากว่าการยืนยัน (quality attribute: consistency / usability)
- **Negative:** debug และ trace ข้าม service ยากขึ้น (quality attribute: testability / observability)
- **Risks and mitigations:**
  - *Event หายหรือซ้ำ* → ใช้ transactional outbox ฝั่ง publisher, consumer ต้อง idempotent (ใช้ order id เป็น key), ตั้ง retry และ dead-letter queue (reliability)
  - *Broker เป็น single point of failure* → ใช้ broker แบบ replicated/managed ตามที่ทีมรองรับ (availability)
  - *MailFast ล่มหรือถูก rate-limit* → retry พร้อม backoff ใน email consumer และ alert เมื่อ queue สะสม (availability)
  - *Event schema เปลี่ยนแล้วทำให้ consumer พัง* → กำหนด schema และ versioning ของ `OrderPlaced` (modifiability)
  - *การเก็บข้อมูลลูกค้าใน event (PII) ไปยัง analytics/MailFast* → ส่งเฉพาะข้อมูลที่จำเป็น (security / privacy; ดู Open questions)

## Validation

- ทดลอง fault injection: ปิด MailFast adapter และ `loyaltyservice` แล้ววัดว่า success rate ของการสั่งซื้อไม่ลดลง
- วัด latency ของ checkout (p95) ก่อน/หลัง ต้องไม่เพิ่มขึ้นจากการเพิ่ม post-order actions
- วัด end-to-end delay จาก `OrderPlaced` ถึงอีเมลถูกส่ง และ points ถูกบันทึก เทียบกับ SLA ที่ตกลง (ยังไม่มีค่า — ดู Open questions)
- ทดสอบ duplicate delivery ว่าผู้ใช้ไม่ได้รับอีเมลหรือ points ซ้ำ
- พิสูจน์ความสามารถในการขยาย: เพิ่ม consumer ตัวที่ 4 (เช่น SMS) แบบ proof-of-concept โดยไม่แก้ผู้ publish
- ทบทวน ADR นี้เมื่อมี action ตัวแรกที่ต้องการผลลัพธ์แบบ synchronous (เช่น fraud scoring) หรือเมื่อครบ 6 เดือนหลัง deploy

## Open questions

- ผู้ตัดสินใจ (Deciders) คือใคร และใครเป็นผู้อนุมัติ?
- สถาปัตยกรรมปัจจุบันของ order flow เป็นอย่างไร (ใครเรียก `emailservice` อยู่ และเรียกด้วยโปรโตคอลอะไร)? ADR นี้ไม่ได้อ่านไฟล์ระบบจึงไม่ทราบ
- ปัจจุบันมี message broker ในระบบหรือไม่ และทีมมีทักษะ/งบประมาณ operate broker หรือไม่?
- SLA ของแต่ละ action คืออะไร (อีเมลต้องถึงภายในกี่วินาที/นาที, points ต้องแสดงเมื่อไหร่)?
- ปริมาณคำสั่งซื้อต่อวินาทีหรือต่อวัน (peak) เท่าไร? ใช้ประเมินขนาด broker
- MailFast มี API, rate limit, SLA, ราคา และการรับประกัน idempotency อย่างไร?
- `loyaltyservice` ต้องมี data store ของตัวเองหรือไม่ และกติกาการให้ points คืออะไร (รวมกรณียกเลิก/คืนเงิน)?
- Analytics event ส่งไปที่ไหน (ระบบ analytics ของ marketing คืออะไร) และข้อมูลใดห้ามส่ง (PII/กฎหมายคุ้มครองข้อมูล)?
- Fraud scoring ต้องบล็อกคำสั่งซื้อก่อนยืนยันหรือไม่? ถ้าใช่ ต้องแยกเป็นการตัดสินใจใน ADR ใหม่
- ต้องการ ordering guarantee ระหว่าง event ของคำสั่งซื้อเดียวกันหรือไม่?
