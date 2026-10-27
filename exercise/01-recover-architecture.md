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
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR=paymentservice:50051`) | ✅ | `src/checkoutservice/main.go` บรรทัด 116 อ่าน env `PAYMENT_SERVICE_ADDR` และบรรทัด 123 เรียก `mustConnGRPC(..., &svc.paymentSvcConn, ...)` ซึ่งเป็น gRPC client connection |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR=emailservice:5000`) | ✅ | `main.go` บรรทัด 115 และ 122 (`EMAIL_SERVICE_ADDR`, `emailSvcConn`) เป็น gRPC เช่นกัน ส่วน Service port 5000 → container 8080 ตรงกับ `emailservice.yaml` |
| `frontend` → `adservice` | `frontend.yaml` (`AD_SERVICE_ADDR=adservice:9555`) | ✅ | `src/frontend/main.go` บรรทัด 138 และ 147 (`AD_SERVICE_ADDR`, `mustConnGRPC(..., &svc.adSvcConn, ...)`) และ `handlers.go` มี `chooseAd` เรียก `fe.getAd` โดยถ้าล้มเหลวจะแค่ warn (ไม่ critical) |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550`) | ✅ | `src/recommendationservice/recommendation_server.py` บรรทัด 73 เรียก `product_catalog_stub.ListProducts(demo_pb2.Empty())` (gRPC) แต่ยังไม่ได้เปิดดูส่วนท้ายไฟล์ที่ประกาศ stub และอ่าน env var ของ address |
| `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`) | ✅ (dependency) / ❓ (ตัว Redis) | `src/cartservice/src/cartstore/RedisCartStore.cs` เก็บ/อ่านตะกร้าผ่าน `IDistributedCache` (`GetAsync`/`SetAsync` บรรทัด 40, 60) โดย `userId` เป็น key และ `Cart` ถูก serialize เป็น protobuf bytes ยืนยันว่ามี cache store จริง แต่ยังไม่ได้เปิดไฟล์ตั้งค่า (เช่น `Startup.cs`) ว่าผูกกับ Redis ผ่าน `REDIS_ADDR` |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - **ไม่พบ dependency ที่ไม่มีจริง** ในห้ารายการที่ตรวจ และ `frontend` ต่อ gRPC ไปยัง 7 service ตรงกับตาราง (`src/frontend/main.go` บรรทัด 141–147)
  - **ขาด: `frontend` → `shoppingassistantservice` (HTTP, ไม่ใช่ gRPC)** Run B ใส่ไว้ใน Gaps แต่ไม่ได้ใส่ในแผนภาพ โค้ดยืนยันว่ามี dependency นี้จริง: `handlers.go` บรรทัด 464 สร้าง `http.NewRequest(POST, "http://" + fe.shoppingAssistantSvcAddr, ...)` ใน `chatBotHandler` และ `main.go` บรรทัด 139 บังคับ `mustMapEnv` ให้ต้องมี env นี้ ทั้งที่ไม่มี manifest ของ service นี้ใน `kubernetes-manifests/` (เป็น optional component ที่ถูก comment ไว้) และ Run B ไม่ได้ระบุว่าเป็น HTTP
  - **ขาด: OpenTelemetry collector** `COLLECTOR_SERVICE_ADDR` ใช้ใน `frontend/main.go` (บรรทัด 178), `checkoutservice/main.go` (บรรทัด 164) และ `recommendation_server.py` (บรรทัด 116) ส่ง trace ผ่าน OTLP/gRPC เมื่อ `ENABLE_TRACING=1` ใน manifests ที่อ่านไม่มี env นี้ จึงมองไม่เห็นจากหลักฐานของ Run B และ Run B ก็ไม่ได้ระบุไว้ใน Gaps
  - **Container ที่น่าสงสัย: `loadgenerator`** Run B ใส่ในแผนภาพ แม้ `kustomization.yaml` comment ออกไว้ (Run B ระบุเรื่องนี้ใน Gaps แล้ว) จึงขึ้นกับว่าถือเป็นส่วนหนึ่งของระบบหรือเป็นเครื่องมือทดสอบ
  - **ภาษาที่ Run B ระบุว่า "not evidenced"** ถูกต้องตามหลักฐานที่ให้ และจากซอร์สพบว่าเป็น Go (`checkoutservice`, `frontend`), Python (`recommendationservice`) และ C# (`cartservice`)

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

A ไม่สามารถรู้ ผู้เรียก → ผู้ถูกเรียก เพราะ .proto ไม่บอกเรื่อง implementation

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

เข้าใจ architecture ผิด