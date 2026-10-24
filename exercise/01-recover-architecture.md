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
| frontend → adservice | frontend.yaml (AD_SERVICE_ADDR), adservice.yaml (Service port grpc), demo.proto (AdService) | ✅ | ตั้งค่า adservice:9555 ตรงกับพอร์ต 9555 ของ Service และ proto ระบุ AdService.GetAds |
| frontend → cartservice | frontend.yaml (CART_SERVICE_ADDR), cartservice.yaml (Service port grpc), demo.proto (CartService) |  ✅ | ตั้งค่า cartservice:7070 ตรงกับพอร์ต 7070 ของ Service และ proto ระบุ CartService.AddItem |
| frontend → checkoutservice | frontend.yaml (CHECKOUT_SERVICE_ADDR), checkoutservice.yaml (Service port grpc), demo.proto (CheckoutService) | ✅ | ตั้งค่า checkoutservice:5050 ตรงกับพอร์ต 5050 ของ Service และ proto ระบุ CheckoutService.PlaceOrder |
| frontend → currencyservice | frontend.yaml (CURRENCY_SERVICE_ADDR), currencyservice.yaml (Service port grpc), demo.proto (CurrencyService) | ✅ | ตั้งค่า currencyservice:7000 ตรงกับพอร์ต 7000 ของ Service และ proto ระบุ CurrencyService.GetSupportedCurrencies |
| frontend → productcatalogservice | frontend.yaml (PRODUCT_CATALOG_SERVICE_ADDR), productcatalogservice.yaml (Service port grpc), demo.proto (ProductCatalogService) | ✅ | ตั้งค่า productcatalogservice:3550 ตรงกับพอร์ต 3550 ของ Service และ proto ระบุ ProductCatalogService.ListProducts และ GetProduct |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): …

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด 
`ตอบ` agent ไม่สามารถรู้ว่าการเรียกใช้ depenencies เกิดขึันยังไงได้เพราะ agent ไม่ได้ถูอนุญาติให้อ่านไฟล์อื่นๆได้ และ ใน `.proto` ไม่มีข้้อมูลนี้
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ `ตอบ` dependency ที่ไม่มีจริง
