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
| 1 | `demo.proto` ไฟล์เดียวนิยาม service ทั้ง 9 ตัว (Claim 1) | Evidenced | Evidenced | `system/online-boutique/protos/demo.proto` มี `service CartService` (บรรทัด 23) ถึง `service AdService` (บรรทัด 243) ครบ 9 service ในไฟล์เดียว |
| 2 | `PlaceOrder` เรียก service อื่นแบบ blocking ตามลำดับ (Claim 4) | Evidenced | Evidenced | `src/checkoutservice/main.go` บรรทัด 252–274: `chargeCard` → `shipOrder` → `emptyUserCart` → `sendOrderConfirmation` เรียกต่อกันใน goroutine เดียว ขั้นถัดไปรอผลขั้นก่อนหน้า ไม่มี `go` หรือ channel |
| 3 | เหตุผลที่เลือก sync gRPC คือ flow ต้องการผลของแต่ละขั้นก่อนขั้นถัดไป (Claim 13) | Inferred | Inferred | โค้ดแสดงเพียงว่า flow เป็นลำดับ (`main.go` บรรทัด 252–274) แต่ไม่มีเอกสาร/คอมเมนต์ในไฟล์ที่ให้อ่านที่ระบุเจตนาผู้ออกแบบ จึงเป็นการอนุมานจากพฤติกรรม |
| 4 | เหตุผลที่ใช้ proto ไฟล์เดียวคือเพื่อแชร์ชนิดข้อมูล (Claim 14) | Inferred | Inferred | `Money` ถูกใช้ใน `Product`, `GetQuoteResponse`, `CurrencyService.Convert`, `ChargeRequest` (`demo.proto` บรรทัด 82, 118, 142, 189) สอดคล้องกับเหตุผลนี้ แต่ไม่มีไฟล์ใดระบุเจตนา |
| 5 | การแก้ proto ร่วมกระทบทุก service ที่ generate stub และต้อง deploy พร้อมกัน (Claim 20) | Inferred | Inferred | `main.go` บรรทัด 34 import `.../src/checkoutservice/genproto` แต่ `src/checkoutservice/` ไม่มีโฟลเดอร์ `genproto` (สร้างตอน build) และไม่เห็นกระบวนการ build/deploy ในไฟล์ที่ให้อ่าน ส่วน "ต้อง deploy พร้อมกัน" เป็นการอนุมานล้วน |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า *Evidenced* กี่ข้อ 12 (Claim 1–12) ตามที่นิสิตตรวจ 12

## สะท้อนคิด

1. agent เคยระบุการเดาว่าเป็น *Evidenced* หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี

    ไม่พบกรณีที่ agent ระบุการเดาว่าเป็น Evidenced ตรวจสอบโดยครวจ Claim 1–12 ทีละข้อกับไฟล์ต้นทาง ด้วย grep บรรทัดที่ agent อ้าง เช่น mustConnGRPC, _ = cs.emptyUserCart, context.TODO(), insecure.NewCredentials() และอ่านลำดับใน PlaceOrder (บรรทัด 252–274) ทั้ง 12 ข้อมีหลักฐานตรงตามที่อ้าง

2. หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด
   1. คิดว่านี่คือการตัดสินใจจริงของทีมออกแบบ
      - ADR ระบุ Status: Reconstructed แต่ส่วน Decision เขียนว่า "เลือกตัวเลือก 1" และมีรายการ Considered options 4 ข้อ
      - ผู้อ่านอาจเข้าใจว่าทีมเคยเปรียบเทียบ REST และ async messaging จริง
   2. คิดว่านี่คือการตัดสินใจจริงของทีมออกแบบ
      - ADR ระบุ Status: Reconstructed แต่ส่วน Decision เขียนว่า "เลือกตัวเลือก 1" และมีรายการ Considered options 4 ข้อ
      - ผู้อ่านอาจเข้าใจว่าทีมเคยเปรียบเทียบ REST และ async messaging จริง
      - ความจริงคือ Claim 17 ระบุว่าตัวเลือก 2–4 agent สร้างขึ้นเพื่อเปรียบเทียบเอง ไม่มีหลักฐานว่าเคยพิจารณา
   3. คิดว่า "เหตุผล" และ "decision drivers" มีหลักฐานรองรับ
      - Drivers ทั้งหมดและลำดับความสำคัญเป็น Inferred (Claim 15) และเหตุผลหลักของ sync gRPC กับ proto เดียวก็เป็น Inferred (Claim 13, 14)
      - ผู้อ่านที่ไม่เปิดตาราง claim check จะเห็นเป็นเหตุผลของผู้ออกแบบ
      - ผู้ตัดสินใจและวันที่ตัดสินใจก็ไม่ทราบ
   4. คิดว่า Pros/Cons ของ gRPC เทียบกับ REST และ async เป็นข้อมูลที่วัดในระบบนี้
      - ข้อความอย่าง "binary เร็วกว่า" และ "ลด temporal coupling" เป็นความรู้ทั่วไปของอุตสาหกรรม ไม่ได้วัดจาก Online Boutique (Claim 18, 19)
   5. คิดว่าทุก service ใน repo สื่อสารผ่าน gRPC
      - ADR ครอบคลุมเฉพาะ service ใน demo.proto โดยอ่านจากโค้ดเพียง 3 ไฟล์
      - frontend รับ HTTP จากผู้ใช้ และมี shopping assistant (/assistant, /bot) ที่ไม่อยู่ใน demo.proto โปรโตคอลปลายทางไม่ทราบ
      - ข้อสรุป "ไม่มี message broker" ยืนยันได้เฉพาะ 2 ไฟล์ (Claim 6)
      - service อื่น เช่น adservice, cartservice, recommendationservice ไม่ได้ถูกอ่านเลย
   6. เข้าใจกลไกการสร้างและ deploy ผิด
      - ข้อความ "แก้ proto แล้วต้อง regenerate และ deploy พร้อมกัน" ไม่มีหลักฐานเรื่อง build/deploy ในไฟล์ที่อ่าน (Claim 20)
      - genproto ไม่อยู่ใน repo แต่ถูกสร้างตอน build
   7. ตัวเลขผิดเรื่องจำนวน connection
      - ADR บรรทัด 17 เขียนว่า frontend เชื่อมต่อ "6 service" แต่ในโค้ดเป็น 7 (main.go บรรทัด 141–147)
   8. คิดว่าข้อสังเกตด้านคุณภาพโค้ดเป็นส่วนหนึ่งของการตัดสินใจ
      - ADR ชี้ว่าไม่มี refund หลัง chargeCard สำเร็จแล้ว shipOrder ล้ม (ยกเว้น context.TODO() ซึ่ง ADR ระบุเองว่าไม่ใช่ส่วนของการตัดสินใจ) และใช้ insecure.NewCredentials() ข้อเท็จจริงเหล่านี้ตรงกับโค้ด แต่ไม่ได้แปลว่าเป็นผลโดยตรงของการเลือก sync gRPC และเป็นข้อบกพร่องที่ตั้งใจหรือเป็นเพียงข้อจำกัดของระบบตัวอย่างก็ไม่ทราบ

3. AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต

    AI กู้คืนจากโค้ดได้ (ข้อเท็จจริงเชิงโครงสร้างและพฤติกรรม):
    - demo.proto ไฟล์เดียวนิยาม service ทั้ง 9 ตัวและชนิดข้อมูลร่วม (Money, Address, CartItem) (Claim 1, 2)
    - checkoutservice เรียก 6 service ผ่าน gRPC และ PlaceOrder เรียกแบบ blocking ตามลำดับ (Claim 3, 4)
    - มี trace propagation ผ่าน otelgrpc (Claim 8)
    - พฤติกรรมที่เห็นได้ เช่น ไม่มี refund หลัง charge สำเร็จแล้ว ship ล้ม (Claim 9), emptyUserCart ถูกเพิกเฉย error (Claim 10), การเชื่อมต่อไม่เข้ารหัส (Claim 12)

    มีเพียงสถาปนิกเดิมที่รู้ (เจตนาและบริบทของการตัดสินใจ):
    - ทำไมเลือก sync gRPC แทน REST หรือ async
    - ทำไมใช้ proto ไฟล์เดียว
    - ตัวเลือกที่เคยพิจารณาแล้วปัดทิ้ง
    - ลำดับความสำคัญของ quality attributes
    - ใครตัดสินใจและเมื่อไร
    - การไม่มี compensation หลัง charge เป็นการยอมรับความเสี่ยงโดยเจตนา หรือเป็นเพียงความเรียบง่ายของระบบตัวอย่าง

    ตัวอย่างจาก ADR ของนิสิต: การใช้ demo.proto ไฟล์เดียว
    - กู้คืนได้จากโค้ด: Money ถูกใช้ข้าม service ใน Product, GetQuoteResponse, CurrencyService.Convert และ ChargeRequest (Claim 2) และ service ทุกตัวอ้างอิงไฟล์เดียวกัน
    - กู้คืนไม่ได้: เหตุผลว่า ทำไม ทีมเลือกแบบนี้ ADR ระบุว่า "เพื่อแชร์ชนิดข้อมูลและลดความซ้ำซ้อน" แต่ Claim 14 จัดเป็น Inferred เพราะไม่มีไฟล์ใดระบุเจตนา และ ADR ยอมรับเองว่า "ส่วนเหตุผลที่ผู้ออกแบบจริงเลือก ไม่มีหลักฐานในไฟล์ที่อนุญาต" เจตนาจริงอาจเป็นอย่างอื่น เช่น ความสะดวกในการสาธิต ข้อจำกัดของเครื่องมือ generate stub หรือแค่ความเคยชิน โค้ดแยกสมมติฐานเหล่านี้ออกจากกันไม่ได้