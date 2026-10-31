Agent: Codex; Model: GPT-6

# ADR-0002: แยกงานหลังคำสั่งซื้อออกจาก checkout ด้วยเหตุการณ์ที่ส่งผ่าน RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-30
- **Deciders:** ยังไม่ระบุ

## Context and problem

`checkoutservice.PlaceOrder` เรียก `paymentservice` และ `shippingservice` ก่อนเรียก `emailservice.SendOrderConfirmation` ผ่าน gRPC; หากส่งอีเมลไม่สำเร็จ ระบบบันทึกคำเตือนและยังตอบผลคำสั่งซื้อสำเร็จ (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`, `chargeCard`, `shipOrder`, `sendOrderConfirmation`). ข้อกำหนดใหม่ให้ใช้ MailFast แทน logic จำลองของ `emailservice`, เพิ่ม `loyaltyservice`, บันทึกเหตุการณ์ให้ marketing และรองรับงานหลังคำสั่งซื้อเพิ่มเติมในปีหน้า (`docs/scenario.md`, R-12).

เป้าหมาย p95 ของ `PlaceOrder` คือไม่เกิน 500 ms จากปัจจุบันประมาณ 300 ms ขณะที่ MailFast มี p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5% และจำกัด 50 emails/s; ยอดสูงสุดคือ 200 orders/s (`docs/scenario.md`, Quality goals and constraints). จึงไม่ควรรอ MailFast ในเส้นทางตอบกลับคำสั่งซื้อ [Inferred]. ต้องให้แต้มภายใน 1 นาทีแบบ exactly once และคำสั่งซื้อต้องไม่ล้มเหลวเพราะระบบอีเมล แต้ม หรือ analytics ใช้งานไม่ได้ (`docs/scenario.md`, Quality goals and constraints).

## Decision drivers

1. **Performance:** รักษา p95 ของ `PlaceOrder` ≤ 500 ms ที่ 200 orders/s (`docs/scenario.md`, Quality goals and constraints).
2. **Availability และ correctness:** ระบบปลายทางที่ล้มเหลวต้องไม่ทำให้คำสั่งซื้อล้มเหลว; แต้มต้องลงภายใน 1 นาทีและไม่ซ้ำ (`docs/scenario.md`, Quality goals and constraints).
3. **Modifiability:** เพิ่ม SMS, fraud scoring และ recommendations update ได้โดยลดการแก้ `checkoutservice` (`docs/scenario.md`, R-12).
4. **Operability และ cost:** ทีมมี 4 คน ถนัด Go/gRPC และ 2 คนเคยใช้ RabbitMQ; องค์กรชอบส่วนประกอบ open-source ที่ติดตั้งเองบน Kubernetes (`docs/scenario.md`, Quality goals and constraints).

## Considered options

1. ให้ `checkoutservice` เรียก MailFast, `loyaltyservice` และ analytics แบบ synchronous โดยตรง
2. ให้ `checkoutservice` บันทึกเหตุการณ์ `OrderPlaced` ลง durable outbox แล้วส่งผ่าน RabbitMQ ให้ worker แยกงาน
3. ให้ `checkoutservice` ส่งงานเข้า durable queue แยกตามระบบปลายทางโดยตรง

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| เรียกตรงแบบ synchronous | เพิ่มโครงสร้างพื้นฐานน้อย; ลำดับการทำงานมองเห็นง่าย | MailFast p95 1.5 s เกินเป้า checkout 500 ms; ระบบปลายทางเพิ่มจุดพึ่งพาและต้องแก้ `checkoutservice` เมื่อเพิ่มงาน (`docs/scenario.md`, Quality goals and constraints; R-12) |
| outbox + RabbitMQ | แยกเวลา checkout ออกจากงานปลายทาง; กระจายเหตุการณ์ให้ผู้รับใหม่ได้; RabbitMQ สอดคล้องกับทักษะทีมและข้อกำหนดการติดตั้งเอง (`docs/scenario.md`, R-12; Quality goals and constraints) | ต้องดูแล outbox, broker และ worker; การส่งซ้ำทำให้ผู้รับต้องป้องกันผลซ้ำ; broker หรือ outbox ที่ล่าช้าทำให้แต้มเกิน 1 นาทีได้ |
| queue แยกจาก `checkoutservice` | งานปลายทางทำแบบ asynchronous และแยกความล้มเหลวได้ | `checkoutservice` ต้องรู้จักผู้รับทุกราย; การเพิ่ม SMS หรือบริการอื่นต้องแก้ฝั่ง checkout; หาก publish ไม่สำเร็จหลังสร้างคำสั่งซื้อ งานอาจหาย |

## Decision

เลือกข้อ 2: เมื่อสร้างคำสั่งซื้อสำเร็จ ให้บันทึก `OrderPlaced` พร้อม `order_id` และข้อมูลที่ผู้รับจำเป็นต้องใช้ลง durable outbox จากนั้น publisher ส่งไป RabbitMQ แบบ durable; worker อีเมลเรียก MailFast, `loyaltyservice` ให้แต้ม และ worker analytics บันทึกเหตุการณ์ของ marketing แยกกัน [Inferred: เป็นการออกแบบใหม่จาก R-12; การมี outbox, RabbitMQ และ worker ยังไม่ปรากฏในไฟล์ที่ให้]. ใช้การส่งแบบ at-least-once พร้อม retry และ dead-letter queue. `loyaltyservice` ต้องทำผลการให้แต้มแบบ exactly once ด้วย `order_id` ที่ไม่ซ้ำและการบันทึกแต้มกับเครื่องหมายประมวลผลในธุรกรรมเดียว; การส่งข้อความ exactly once โดย broker เพียงอย่างเดียวไม่เพียงพอ [Inferred].

การบันทึก outbox ต้องเชื่อมกับจุดที่ยืนยันว่าได้วางคำสั่งซื้อแล้วอย่างปลอดภัย ก่อนตอบกลับผลสำเร็จ; วิธีทำธุรกรรมนี้ยังต้องออกแบบ เพราะไฟล์ที่ให้ไม่แสดง order store หรือธุรกรรมคำสั่งซื้อ (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`; **Open questions**). การหยุดทำงานของ MailFast, `loyaltyservice` หรือ analytics จึงไม่ทำให้ `PlaceOrder` ล้มเหลว [Inferred]. จำกัดอัตรา worker อีเมลรวมไม่เกิน 50 emails/s และให้คิวรองรับยอดค้าง; หาก 200 orders/s เกิดต่อเนื่อง คิวอีเมลจะเพิ่มอย่างน้อย 150 รายการ/s [Inferred จาก `docs/scenario.md`, Quality goals and constraints].

## Consequences

- **Positive:** งานปลายทางไม่เพิ่มเวลารอ MailFast ใน `PlaceOrder` (performance); ความล้มเหลวของผู้รับแยกจากผลคำสั่งซื้อ (availability); เพิ่มผู้รับเหตุการณ์ได้โดยไม่เพิ่มการเรียก synchronous ใน checkout (modifiability) [Inferred จาก `docs/scenario.md`, R-12; Quality goals and constraints].
- **Positive:** `order_id` ที่ไม่ซ้ำและการบันทึกแต้มแบบ atomic รองรับผลลัพธ์ exactly once แม้มีการส่งซ้ำ (correctness) [Inferred].
- **Negative:** ต้องดูแล durable outbox, RabbitMQ, worker และพื้นที่เก็บข้อความค้าง (cost, operability); อีเมลและ analytics อาจมาหลังผลคำสั่งซื้อ (consistency) [Inferred].
- **Risks and mitigations:** หาก outbox หรือ broker ค้าง แต้มอาจเกิน 1 นาที; ติดตามอายุข้อความและแจ้งเตือนก่อนครบกำหนด พร้อม retry และ dead-letter queue (availability, correctness). MailFast จำกัด 50 emails/s เทียบกับยอดสูงสุด 200 orders/s จึงต้องยอมรับคิวอีเมลหรือเจรจาเพิ่มโควตา (latency, capacity; `docs/scenario.md`, Quality goals and constraints). การส่งซ้ำอาจทำให้อีเมลซ้ำ; ใช้ idempotency key หาก MailFast รองรับ และตรวจสอบเงื่อนไขนี้ก่อนใช้งานจริง (correctness).

## Validation

ทดสอบโหลด 200 orders/s และตรวจ p95 `PlaceOrder` ≤ 500 ms; จำลอง MailFast ตอบช้า 1.5–8 s หรือหยุดทำงาน รวมถึง `loyaltyservice` และ analytics หยุดทำงาน แล้วตรวจว่าคำสั่งซื้อยังสำเร็จ (`docs/scenario.md`, Quality goals and constraints). ทดสอบการส่งซ้ำและการเริ่ม worker ใหม่เพื่อยืนยันว่าแต่ละ `order_id` ได้แต้มครั้งเดียว; วัดเวลาให้แต้มจากคำสั่งซื้อจนเสร็จว่า ≤ 1 นาที. วัด backlog และอายุข้อความของทุกคิว รวมถึงตรวจว่าอีเมลไม่เกิน 50 emails/s (`docs/scenario.md`, Quality goals and constraints).

## Open questions

- จะเก็บ outbox ที่ใด และจะผูกการบันทึก `OrderPlaced` กับการยืนยันคำสั่งซื้ออย่างไร? **Not evidenced in the provided files** (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`).
- MailFast รองรับ idempotency key หรือการเพิ่มโควตาหรือไม่ และอีเมลต้องส่งภายในเวลาเท่าใด? **Not evidenced in the provided files** (`docs/scenario.md`, Quality goals and constraints).
- ระบบ analytics ปลายทาง รูปแบบข้อมูล `OrderPlaced`, ระยะเวลาการเก็บข้อมูล และนโยบายข้อมูลส่วนบุคคลคืออะไร? **Not evidenced in the provided files** (`docs/scenario.md`, R-12).
- ใครเป็นผู้ตัดสินใจและดูแล broker/outbox ใน production? **Not evidenced in the provided files** (`docs/scenario.md`, Quality goals and constraints).
