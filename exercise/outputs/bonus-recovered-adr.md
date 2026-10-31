Agent: Codex (GPT-6)

# ADR: ใช้ gRPC แบบรอผลและสัญญาบริการร่วมใน `demo.proto`

- **Status:** Reconstructed
- **Date:** Not evidenced in the provided files
- **Deciders:** Not evidenced in the provided files

## Context and problem

`demo.proto` นิยาม RPC ของ `CartService`, `CheckoutService` และบริการอื่นในไฟล์เดียว รวมถึงชนิดข้อมูลร่วมอย่าง `CartItem`, `Money` และ `OrderResult` (`system/online-boutique/protos/demo.proto`: `service CartService`, `service CheckoutService`, `message Money`, `message OrderResult`) ส่วน `checkoutservice` เรียกบริการปลายทางตามลำดับภายใน `PlaceOrder` แล้วรอคำตอบเพื่อนำไปคำนวณยอด ชำระเงิน และจัดส่ง (`system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `chargeCard`, `shipOrder`) [Evidenced]

ขอบเขตของคำว่า “ทุกบริการ” พิสูจน์ไม่ได้จากไฟล์ที่ให้มา: `frontend` เปิด HTTP routes และ HTTP server สำหรับผู้ใช้ (`system/online-boutique/src/frontend/main.go`: `r.HandleFunc`, `http.ListenAndServe`) ขณะที่ไฟล์นี้แสดงเพียงการสร้าง gRPC clients ไปยังบริการบางตัว (`frontendServer`, `mustConnGRPC`) [Evidenced]

## Decision drivers

- [Inferred] ต้องให้ `PlaceOrder` ได้ข้อมูลรถเข็น ราคา ค่าจัดส่ง และผลการชำระเงินก่อนตอบผู้ใช้ (`system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`).
- [Inferred] ต้องให้ชนิดข้อมูลที่ใช้ข้ามบริการตรงกัน เช่น `CartItem`, `Money` และ `OrderResult` (`system/online-boutique/protos/demo.proto`: ข้อความชื่อเดียวกัน).
- [Assumption] ต้องรักษาความง่ายในการพัฒนาและติดตามเส้นทางคำขอ โดยไม่มีหลักฐานว่าผู้ตัดสินใจให้ความสำคัญกับข้อนี้.

## Considered options

1. ใช้ gRPC แบบรอผลและสัญญาบริการร่วมใน `demo.proto` ตามรูปแบบที่พบ [Evidenced] (`system/online-boutique/protos/demo.proto`: `service CheckoutService`; `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`, `mustConnGRPC`).
2. ใช้ gRPC แบบรอผล แต่แยกไฟล์ `.proto` ตามเจ้าของบริการ [Inferred] (ทางเลือกที่ไม่พบในหลักฐาน).
3. ใช้ข้อความหรือเหตุการณ์แบบ asynchronous สำหรับขั้นตอนที่ไม่จำเป็นต้องตอบทันที และแยกสัญญาตามบริการ [Inferred] (ทางเลือกที่ไม่พบในหลักฐาน).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1 | [Inferred] เรียกและรับผลในลำดับเดียว; ใช้ชนิดข้อมูลร่วมได้จากไฟล์เดียว | [Inferred] เวลาตอบและความสำเร็จของ `PlaceOrder` ขึ้นกับปลายทางหลายแห่ง; การเปลี่ยนไฟล์ร่วมอาจกระทบหลายบริการ |
| 2 | [Inferred] แยกความเป็นเจ้าของสัญญาและลดขอบเขตการเปลี่ยนแปลง | [Inferred] ต้องจัดการชนิดข้อมูลร่วมและเวอร์ชันของหลายไฟล์; ยังรอปลายทางระหว่างคำขอ |
| 3 | [Inferred] ลดการรอสำหรับงานที่เลื่อนทำได้ และรองรับการรับคำขอเมื่อปลายทางชั่วคราวไม่พร้อม | [Inferred] ต้องจัดการสถานะ ผลลัพธ์ย้อนหลัง และความล้มเหลวแบบ asynchronous; ขั้นตอนชำระเงินและจัดส่งอาจต้องออกแบบใหม่ |

## Decision

[Inferred] รูปแบบที่กู้คืนได้คือใช้การเรียก gRPC แบบรอผลระหว่าง `frontend` และบริการที่สร้าง client ไว้ รวมถึงระหว่าง `checkoutservice` กับบริการที่เรียกใน `PlaceOrder`; ใช้ `demo.proto` เป็นสัญญาร่วมของ RPC ที่นิยามในไฟล์นี้ (`system/online-boutique/src/frontend/main.go`: `mustConnGRPC`; `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`, `pb.NewCartServiceClient`, `pb.NewPaymentServiceClient`; `system/online-boutique/protos/demo.proto`: `service CartService`, `service PaymentService`). เหตุผลดั้งเดิมที่เลือกแนวทางนี้: Not evidenced in the provided files.

## Consequences

- **Positive:** [Inferred] การส่ง `CartItem` และ `Money` ระหว่างขั้นตอนสั่งซื้อมีสัญญาชนิดข้อมูลร่วม (quality attribute: modifiability, correctness; `system/online-boutique/protos/demo.proto`: `CartItem`, `Money`; `system/online-boutique/src/checkoutservice/main.go`: `prepOrderItems`, `convertCurrency`).
- **Negative:** [Inferred] การรอผลตามลำดับอาจเพิ่มเวลา `PlaceOrder` และความล้มเหลวของการชำระเงินหรือจัดส่งทำให้คำขอล้มเหลว (quality attribute: performance, availability; `system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`, `chargeCard`, `shipOrder`).
- **Risks and mitigations:** [Evidenced] `PlaceOrder` ละเลยข้อผิดพลาดจาก `emptyUserCart` และบันทึกคำเตือนเมื่อส่งอีเมลไม่สำเร็จ (`system/online-boutique/src/checkoutservice/main.go`: `PlaceOrder`); [Inferred] ควรตรวจสอบผลกระทบต่อความถูกต้องของคำสั่งซื้อและกำหนดวิธีจัดการหรือ retry ที่เหมาะสม (quality attribute: correctness, reliability).

## Validation

[Inferred] วัดเวลาและอัตราความล้มเหลวของ `PlaceOrder` เมื่อบริการปลายทางช้า/ไม่พร้อม และทดสอบผลลัพธ์เมื่อ `EmptyCart` หรือ `SendOrderConfirmation` ล้มเหลว; เกณฑ์ยอมรับเชิงตัวเลข: Not evidenced in the provided files.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | `demo.proto` รวม RPC หลายบริการและชนิดข้อมูลร่วม | Evidenced | `system/online-boutique/protos/demo.proto` (`service CartService`, `service CheckoutService`, `message Money`, `message OrderResult`) |
| 2 | `checkoutservice` เรียกบริการต่อเนื่องและรอผลใน `PlaceOrder` | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `chargeCard`, `shipOrder`) |
| 3 | `frontend` เปิด HTTP routes และสร้าง gRPC clients บางตัว จึงยืนยัน “ทุกบริการ” ไม่ได้ | Evidenced | `system/online-boutique/src/frontend/main.go` (`r.HandleFunc`, `http.ListenAndServe`, `mustConnGRPC`) |
| 4 | ต้องได้ข้อมูลและผลลัพธ์ก่อนตอบ `PlaceOrder` | Inferred | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`) |
| 5 | ต้องให้ชนิดข้อมูลข้ามบริการตรงกัน | Inferred | `system/online-boutique/protos/demo.proto` (`CartItem`, `Money`, `OrderResult`) |
| 6 | ความง่ายในการพัฒนา/ติดตามคำขอเป็นแรงขับ | Assumption | Not evidenced in the provided files |
| 7 | ทางเลือก 1 คือรูปแบบ gRPC และไฟล์สัญญาร่วมที่พบ | Evidenced | `system/online-boutique/protos/demo.proto` (`service CheckoutService`); `system/online-boutique/src/checkoutservice/main.go` (`mustConnGRPC`, `PlaceOrder`) |
| 8 | ทางเลือก 2 แยกไฟล์ `.proto` ตามบริการ | Inferred | Not evidenced in the provided files |
| 9 | ทางเลือก 3 ใช้ asynchronous สำหรับงานที่เลื่อนทำได้ | Inferred | Not evidenced in the provided files |
| 10 | ข้อดีข้อเสียของทั้งสามทางเลือกด้านการรอผล การแยกสัญญา และความซับซ้อน | Inferred | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`); `system/online-boutique/protos/demo.proto` (`CartItem`, `Money`); ไม่พบการประเมินทางเลือกในไฟล์หลักฐาน |
| 11 | รูปแบบการตัดสินใจที่กู้คืนได้ใช้ gRPC และ `demo.proto`; ไม่ทราบเหตุผลดั้งเดิม | Inferred | `system/online-boutique/src/frontend/main.go` (`mustConnGRPC`); `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`); `system/online-boutique/protos/demo.proto` (`service CartService`) |
| 12 | ผลดีด้านสัญญาข้อมูลร่วมและผลเสียด้านเวลา/ความพร้อมใช้งาน | Inferred | `system/online-boutique/protos/demo.proto` (`CartItem`, `Money`); `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`, `chargeCard`, `shipOrder`) |
| 13 | ละเลยข้อผิดพลาด `emptyUserCart` และเตือนเมื่อส่งอีเมลไม่สำเร็จ | Evidenced | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`) |
| 14 | ควรตรวจสอบความถูกต้องและกำหนดการจัดการความล้มเหลว | Inferred | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`) |
| 15 | ควรวัดเวลา/ความล้มเหลวและทดสอบปลายทางล้มเหลว | Inferred | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`); เกณฑ์ตัวเลขไม่ปรากฏในหลักฐาน |
