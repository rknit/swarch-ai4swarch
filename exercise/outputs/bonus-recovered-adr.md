Agent: Claude Code, model claude-sonnet-5-5

# ADR-RECON-01: ใช้ synchronous gRPC และ `demo.proto` ไฟล์เดียวร่วมกันสำหรับการสื่อสารระหว่าง service

- **Status:** Reconstructed
- **Date:** 2026-10-27 (วันที่สร้าง ADR ที่ reconstruct นี้; วันที่ตัดสินใจจริง Not evidenced in the provided files)
- **Deciders:** Not evidenced in the provided files (ไฟล์ license ระบุเพียง Google LLC)

## Context and problem

Online Boutique แยกเป็นหลาย service ที่ต้องร่วมกันทำงานเดียว เช่น `PlaceOrder` ที่ต้องใช้ข้อมูลจาก cart, product catalog, currency, shipping, payment และ email ต้องเลือกรูปแบบการสื่อสารและวิธีกำหนดสัญญา (contract) ระหว่าง service

ข้อเท็จจริงที่เห็นจากไฟล์ที่อนุญาตให้อ่าน:

- `demo.proto` เป็นไฟล์เดียว (`package hipstershop`) ที่นิยาม service ทั้งหมด: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`
- `checkoutservice/main.go` เป็น gRPC client ของ 6 service (shipping, product catalog, cart, currency, email, payment) และเรียกแบบ blocking ตามลำดับภายใน `PlaceOrder`
- `frontend/main.go` เป็น HTTP server (gorilla/mux) ที่เปิด gRPC client ไปยัง 6 service: currency, product catalog, cart, recommendation, shipping, checkout, ad (รวม 7 connection)
- ในไฟล์ทั้งสามไม่พบ message broker, queue หรือ client library แบบ async

**ข้อจำกัดของคำถาม:** คำว่า "ทุก service" ไม่ตรงกับหลักฐานทั้งหมด `frontend/main.go` รับ HTTP จากผู้ใช้ และมี `shoppingAssistantSvcAddr` ที่ไม่มี gRPC connection ใน `main()` ส่วน `/assistant` และ `/bot` เป็น HTTP route (ไม่เห็นว่า handler ปลายทางใช้โปรโตคอลใด เพราะไม่ได้อยู่ในไฟล์ที่อนุญาต) และ `demo.proto` ไม่มี service สำหรับ shopping assistant ดังนั้น ADR นี้ครอบคลุมการสื่อสารระหว่าง service ที่อยู่ใน `demo.proto` เป็นหลัก

## Decision drivers

ลำดับตามที่ **[Inferred]** จากหลักฐาน ไม่ได้มีเอกสารระบุ:

- Modifiability / ความสอดคล้องของ contract: ทุก service ใช้ชนิดข้อมูลร่วมกัน (`Money`, `Address`, `CartItem`) ซึ่งหลาย service อ้างอิง
- Interoperability: ผู้เขียน `checkoutservice` และ `frontend` เป็น Go (หลักฐาน: `package main` และ import `google.golang.org/grpc`) ส่วนภาษาของ service อื่นไม่เห็นในไฟล์ที่อนุญาต
- Observability: ต้อง propagate trace context ข้าม service (`otelgrpc`, `propagation.TraceContext`)
- Consistency ของ business flow: การสั่งซื้อต้องได้ผลลัพธ์ (ราคา, tracking id) กลับมาภายใน request เดียว
- Simplicity ในการสาธิต: เป็นระบบตัวอย่างของ Google (`Copyright Google LLC`) **[Assumption]**

## Considered options

1. Synchronous gRPC + `demo.proto` ไฟล์เดียวร่วมกัน (ตัวเลือกที่ใช้อยู่)
2. Synchronous gRPC + ไฟล์ `.proto` แยกต่อ service (แต่ละ service เป็นเจ้าของ contract ของตน)
3. REST/JSON over HTTP แบบ synchronous
4. Asynchronous messaging ผ่าน message broker (event-driven) สำหรับ flow ที่ไม่ต้องรอผล เช่น email

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC + proto เดียว | contract และชนิดข้อมูลร่วม (`Money`, `Address`) นิยามที่เดียว ไม่ซ้ำซ้อน; client/server stub สร้างจาก schema เดียว (`pb.New...Client`); ตรวจสอบความเข้ากันของ API ได้ที่ compile time; ผลลัพธ์ได้ใน request เดียว; trace propagation ผ่าน `otelgrpc` ทำได้ตรงไปตรงมา | ทุก service coupled กับไฟล์เดียว: แก้ไฟล์นี้กระทบทุก service ที่ generate stub; ต้อง regenerate และ deploy พร้อมกันเมื่อเปลี่ยน message ร่วม; caller ต้องรอ callee (temporal coupling); availability ของ `PlaceOrder` ขึ้นกับ 6 service; ไม่มี retry/compensation ใน `PlaceOrder` |
| 2. Sync gRPC + proto แยกต่อ service | แต่ละทีมวิวัฒนาการ contract อิสระ; ลด blast radius ของการแก้ไฟล์ | ชนิดร่วมอย่าง `Money` ต้องซ้ำหรือแยกเป็น shared package; ต้องจัดการ versioning หลายไฟล์ |
| 3. REST/JSON sync | เครื่องมือทั่วไป ดีบักง่าย เรียกจากเบราว์เซอร์ได้ตรง | ไม่มี schema/stub ที่บังคับ contract; payload ใหญ่/ช้ากว่า binary **[Assumption]**; ยังคงเป็น sync coupling |
| 4. Async messaging | ลด temporal coupling; ทนต่อ service ล่มชั่วคราว; เหมาะกับ email/การแจ้งเตือน | ต้องมี broker เพิ่ม (ซับซ้อน/ต้นทุนปฏิบัติการ); ผลลัพธ์ที่ต้องใช้ทันที (quote, payment) ทำยาก; ต้องออกแบบ eventual consistency |

## Decision

เลือก **ตัวเลือก 1**: ทุกการเรียกระหว่าง service ที่นิยามใน `demo.proto` ใช้ synchronous gRPC และใช้ `demo.proto` ไฟล์เดียวเป็น contract กลาง

เหตุผล (ส่วนใหญ่ **[Inferred]**): flow ของ `PlaceOrder` ต้องการผลลัพธ์ของแต่ละขั้นเพื่อใช้ในขั้นถัดไป (ราคาจาก catalog → แปลงสกุลเงิน → รวมยอด → charge → ship) ซึ่ง request/response แบบ sync ตรงกับ flow นี้ที่สุด และ proto เดียวทำให้ชนิดข้อมูลร่วมสอดคล้องกันโดยไม่ต้องจัดการหลายไฟล์ ส่วนเหตุผลที่ผู้ออกแบบจริงเลือก ไม่มีหลักฐานในไฟล์ที่อนุญาต

## Consequences

- **Positive:** contract สอดคล้องทั้งระบบและตรวจพบความไม่เข้ากันได้เร็ว (quality attribute: modifiability / interoperability); flow การสั่งซื้ออ่านและดีบักง่ายเพราะเป็นลำดับ blocking เดียว (quality attribute: modifiability / testability); trace ต่อเนื่องข้าม service (quality attribute: observability)
- **Negative:** `PlaceOrder` ล้มหรือช้าเมื่อ dependency ใดตอบช้า เพราะเรียกแบบ blocking ต่อกัน (quality attribute: availability, performance/latency); การเปลี่ยน `demo.proto` ส่งผลถึงทุก service ที่ใช้ (quality attribute: modifiability, deployability); `frontend` ผูกกับ 7 service โดยตรง (quality attribute: modifiability)
- **Risks and mitigations:**
  - ความไม่สอดคล้องของข้อมูลถ้า step กลางล้ม: ใน `PlaceOrder` หลัง `chargeCard` สำเร็จ ถ้า `shipOrder` ล้มจะคืน `codes.Unavailable` โดยไม่เห็นการ refund/compensation (availability, consistency) ไม่เห็นกลไกลดความเสี่ยงในไฟล์ที่อ่าน
  - ความล้มเหลวของ step ท้ายถูกยอมรับ: `emptyUserCart` ไม่สนใจ error (`_ =`) และ `sendOrderConfirmation` แค่ log warning (availability: degrade อย่างยอมรับได้)
  - `convertCurrency` ใช้ `context.TODO()` แทน `ctx` ทำให้ไม่ผูก deadline/cancellation ของ request ที่เข้ามา (availability, performance) — ข้อสังเกตจากโค้ด ไม่ใช่ส่วนของการตัดสินใจ
  - การเชื่อมต่อใช้ `insecure.NewCredentials()` (ไม่เข้ารหัสระหว่าง service) (security) — Not evidenced ว่าถูกชดเชยด้วยมาตรการอื่น

## Validation

Not evidenced in the provided files ว่าเคยมีการวัดผลหรือกำหนดเกณฑ์ **[Inferred]** วิธีตรวจสอบที่เหมาะสม: วัด latency/error rate ของ `PlaceOrder` เมื่อ inject ความล่าช้าใน dependency ทีละตัว และตรวจว่าการแก้ `demo.proto` ต้อง rebuild กี่ service

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | `demo.proto` เป็นไฟล์เดียวที่นิยาม service ของ cart, recommendation, product catalog, shipping, currency, payment, email, checkout, ad | Evidenced | `system/online-boutique/protos/demo.proto` (`service CartService` … `service AdService`) |
| 2 | ชนิดข้อมูลร่วม (`Money`, `Address`, `CartItem`) ถูกใช้ข้าม service ในไฟล์เดียวกัน | Evidenced | `demo.proto` (`Money` ใช้ใน `Product`, `ChargeRequest`, `GetQuoteResponse`; `CartItem` ใช้ใน `Cart`, `GetQuoteRequest`) |
| 3 | `checkoutservice` เรียก 6 service อื่นผ่าน gRPC client | Evidenced | `checkoutservice/main.go` (`checkoutService` struct, `mustConnGRPC` ×6, `pb.New…Client`) |
| 4 | การเรียกใน `PlaceOrder` เป็น blocking ตามลำดับ | Evidenced | `checkoutservice/main.go` `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart` |
| 5 | `frontend` เป็น HTTP server ที่เรียก service หลังบ้านผ่าน gRPC | Evidenced | `frontend/main.go` (`mux.NewRouter`, `http.ListenAndServe`, `mustConnGRPC`) |
| 6 | ไม่พบ message broker/async client ในไฟล์ที่อนุญาต | Evidenced (absence) | `checkoutservice/main.go`, `frontend/main.go` imports; ยืนยันได้เฉพาะ 3 ไฟล์ ไม่ครอบคลุมทั้งระบบ |
| 7 | "ทุก service" ใช้ gRPC ไม่ตรงทั้งหมด: shopping assistant ไม่มี gRPC connection ใน `main()` และมี route HTTP `/assistant`, `/bot` | Evidenced | `frontend/main.go` (`shoppingAssistantSvcAddr`, routes `/assistant`, `/bot`; ไม่มี service ใน `demo.proto`) โปรโตคอลปลายทาง Not evidenced |
| 8 | มี trace context propagation ข้าม gRPC | Evidenced | `checkoutservice/main.go` (`otelgrpc.NewServerHandler`, `propagation.TraceContext{}`); `frontend/main.go` (`otelgrpc.NewClientHandler`) |
| 9 | ไม่มี refund/compensation หลัง charge สำเร็จแล้ว ship ล้ม | Evidenced | `checkoutservice/main.go` `PlaceOrder` (หลัง `chargeCard` เรียก `shipOrder` แล้ว `return … codes.Unavailable`) |
| 10 | `emptyUserCart` ถูกเพิกเฉย error และ email ล้มแค่ log warning | Evidenced | `checkoutservice/main.go` (`_ = cs.emptyUserCart`, `log.Warnf("failed to send order confirmation…")`) |
| 11 | `convertCurrency` ใช้ `context.TODO()` | Evidenced | `checkoutservice/main.go` `convertCurrency` |
| 12 | การเชื่อมต่อ gRPC ไม่เข้ารหัส | Evidenced | `mustConnGRPC` ในทั้งสองไฟล์ (`insecure.NewCredentials()`) |
| 13 | เหตุผลที่เลือก sync gRPC คือ flow ต้องการผลลัพธ์ของแต่ละขั้นก่อนขั้นถัดไป | Inferred | สอดคล้องกับลำดับใน `PlaceOrder` แต่ไม่มีเอกสารระบุเหตุผลของผู้ออกแบบ |
| 14 | เหตุผลที่ใช้ proto ไฟล์เดียว คือเพื่อแชร์ชนิดข้อมูลและลดความซ้ำซ้อน | Inferred | สอดคล้องกับ claim 2 แต่ไม่มีไฟล์ระบุเจตนา |
| 15 | Decision drivers ทั้งหมดและลำดับความสำคัญ | Inferred | ไม่มีเอกสารระบุ drivers |
| 16 | ผู้ตัดสินใจและวันที่ตัดสินใจ | Not evidenced | ไม่มีในไฟล์ที่อนุญาต (มีเพียง `Copyright Google LLC`) |
| 17 | ผู้เขียนพิจารณา option 2, 3, 4 | Inferred | ไม่มีหลักฐานว่าเคยพิจารณา เป็นตัวเลือกที่สร้างขึ้นเพื่อเปรียบเทียบ |
| 18 | ข้อดี/ข้อเสียของ gRPC เทียบกับ REST (binary เร็วกว่า, ไม่มี schema บังคับ) | Assumption | ความรู้ทั่วไปของอุตสาหกรรม ไม่ได้วัดในระบบนี้ |
| 19 | ข้อดี/ข้อเสียของ async messaging (ลด temporal coupling, ต้องมี broker, eventual consistency) | Assumption | ความรู้ทั่วไปของอุตสาหกรรม |
| 20 | การแก้ไฟล์ proto ร่วมกระทบทุก service ที่ generate stub และต้อง deploy พร้อมกัน | Inferred | `checkoutservice` import `genproto` ที่ generate ภายใน service เอง (`pb ".../src/checkoutservice/genproto"`) แต่ไม่เห็น build/deploy process ในไฟล์ที่อนุญาต |
| 21 | ระบบเป็นตัวอย่างสาธิตจึงเน้นความเรียบง่าย | Assumption | `Copyright Google LLC` ไม่ได้ระบุวัตถุประสงค์ |
| 22 | วิธี validate ที่เสนอ (inject latency, นับ service ที่ต้อง rebuild) | Assumption | แนวปฏิบัติทั่วไป ไม่มีใน repository |
