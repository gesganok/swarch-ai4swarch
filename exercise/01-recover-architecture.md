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
| `checkoutservice` → `cartservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (`CART_SERVICE_ADDR`) | ✅ | พบ `CART_SERVICE_ADDR` กำหนดค่าเป็น `cartservice:7070` จึงยืนยันว่า `checkoutservice` ถูกตั้งค่าให้ติดต่อ `cartservice` |
| `checkoutservice` → `productcatalogservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | พบ `PRODUCT_CATALOG_SERVICE_ADDR` กำหนดค่าเป็น `productcatalogservice:3550` จึงยืนยันว่า `checkoutservice` ถูกตั้งค่าให้ติดต่อ `productcatalogservice` |
| `checkoutservice` → `shippingservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (`SHIPPING_SERVICE_ADDR`) | ✅ | พบ `SHIPPING_SERVICE_ADDR` กำหนดค่าเป็น `shippingservice:50051` จึงยืนยันว่า `checkoutservice` ถูกตั้งค่าให้ติดต่อ `shippingservice` |
| `checkoutservice` → `paymentservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | พบ `PAYMENT_SERVICE_ADDR` กำหนดค่าเป็น `paymentservice:50051` จึงยืนยันว่า `checkoutservice` ถูกตั้งค่าให้ติดต่อ `paymentservice` |
| `checkoutservice` → `currencyservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (`CURRENCY_SERVICE_ADDR`) | ✅ | พบ `CURRENCY_SERVICE_ADDR` กำหนดค่าเป็น `currencyservice:7000` จึงยืนยันว่า `checkoutservice` ถูกตั้งค่าให้ติดต่อ `currencyservice` |
| `checkoutservice` → `emailservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | พบ `EMAIL_SERVICE_ADDR` กำหนดค่าเป็น `emailservice:5000` จึงยืนยันว่า `checkoutservice` ถูกตั้งค่าให้ติดต่อ `emailservice` |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): ไม่พบ dependency ที่ผิดจากรายการที่ตรวจ

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   Run A อ่านเพียง `demo.proto` จึงรู้ว่าแต่ละ service มี RPC อะไร แต่ยังระบุไม่ได้ว่า service ใดเรียก service ใด หรือมี container และ data store ใดถูก deploy จริง  ส่วนRun B อ่านไฟล์ใน `system/online-boutique/kubernetes-manifests/` เพิ่ม จึงระบุ dependency ที่ตั้งค่าไว้จาก `*_SERVICE_ADDR` และพบ `redis-cart` (`exercise/outputs/1B-architecture.md`). แม้ทั้งสอง run จะใช้ model เดียวกัน และ `prompts/1-recover-architecture.md` เหมือนกัน หลักฐานที่ต่างกันก็ทำให้รายละเอียดสถาปัตยกรรมที่กู้คืนได้ต่างกัน.

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   จุดที่อาจทำให้สมาชิกใหม่เข้าใจผิดมากที่สุดคือการอ่านลูกศร 
   ภาพ `Run B` มีลูกศรหลายเส้น สมาชิกใหม่อาจเข้าใจว่าบริการเหล่านี้เรียกกันทุกครั้งที่มีการใช้งาน แต่ไฟล์ตั้งค่าบอกเพียงว่าแต่ละบริการถูกตั้งค่าให้ติดต่อบริการใดได้ ยังไม่ได้บอกว่าเรียกกันเมื่อไร
