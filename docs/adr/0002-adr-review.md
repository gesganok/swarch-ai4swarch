# ADR-0002: ใช้ RabbitMQ ส่งงานหลังสั่งซื้อ

- **Status:** Proposed
- **Date:** 2026-10-30
- **Deciders:** นิสิตผู้จัดทำ

## Context and problem

ปัจจุบัน `checkoutservice.PlaceOrder` เรียกบริการชำระเงินและจัดส่ง แล้วจึงเรียก `emailservice` หากการส่งอีเมลล้มเหลว คำสั่งซื้อยังสำเร็จ (`docs/scenario.md`, หัวข้อ How it works today). โจทย์ต้องการส่งอีเมลผ่าน MailFast ให้ `loyaltyservice` ที่จะสร้างใหม่มอบแต้ม และบันทึกเหตุการณ์ให้ marketing ต่อไปอาจมีงาน SMS, fraud scoring และการปรับคำแนะนำสินค้า (`docs/scenario.md`, หัวข้อ The new requirement).

โจทย์กำหนดให้ p95 ของ `PlaceOrder` ไม่เกิน 500 ms หมายถึง 95% ของคำขอใช้เวลาไม่เกินค่านี้ (ปัจจุบันประมาณ 300 ms). MailFast มี p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5% และส่งได้ไม่เกิน 50 อีเมล/วินาที ขณะที่ช่วงพีกมี 200 คำสั่งซื้อ/วินาที. แต้มต้องเข้าภายใน 1 นาทีและให้เพียงครั้งเดียวต่อคำสั่งซื้อ โดยอีเมล แต้ม หรือ analytics ขัดข้องแล้วคำสั่งซื้อต้องยังสำเร็จ (`docs/scenario.md`, Quality goals and constraints).

ร่าง AI เสนอ outbox หรือที่เก็บเหตุการณ์ไว้รอส่ง แต่โจทย์ไม่ได้ระบุว่าจะเก็บคำสั่งซื้อแบบถาวรที่ไหน (`docs/scenario.md`). Redis ใน ADR-0001 ใช้กับตะกร้าและอาจสูญเสียข้อมูลเมื่อเริ่มใหม่ จึงไม่เลือกใช้เป็นที่เก็บเหตุการณ์คำสั่งซื้อ (`docs/adr/0001-use-redis-for-cart-storage.md`, Decision และ Consequences).

## Decision drivers

1. **Performance:** รักษาเวลา p95 ของ `PlaceOrder` ไม่เกิน 500 ms ที่ช่วงพีก 200 orders/s (`docs/scenario.md`, Quality goals and constraints).
2. **Availability:** อีเมล แต้ม หรือ analytics หยุดทำงานแล้วคำสั่งซื้อต้องยังสำเร็จ (`docs/scenario.md`, Quality goals and constraints).
3. **Correctness:** ให้แต้มภายใน 1 นาทีและไม่ให้ซ้ำ (`docs/scenario.md`, Quality goals and constraints).
4. **Modifiability และ cost:** เพิ่มงานใหม่ได้โดยไม่แก้ `checkoutservice` ทุกครั้ง; ทีมมี 4 คน โดย 2 คนเคยใช้ RabbitMQ และองค์กรชอบระบบ open-source ที่ดูแลเองบน Kubernetes (`docs/scenario.md`, หัวข้อ The new requirement และ Quality goals and constraints).

## Considered options

1. ให้ `checkoutservice` เรียกอีเมล แต้ม และ analytics โดยตรง แล้วรอให้แต่ละงานเสร็จ
2. บันทึกคำสั่งซื้อกับเหตุการณ์ที่ต้องส่งพร้อมกัน แล้วให้ RabbitMQ ส่งต่อไปยังแต่ละงาน
3. ให้ `checkoutservice` ส่งงานเข้า queue (คิวงานที่รอทำ) แยกสำหรับอีเมล แต้ม และ analytics

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. เรียกตรงและรอ | ไม่ต้องมีระบบส่งข้อความเพิ่ม; ตามลำดับงานได้ง่าย | MailFast ช้ากว่าเป้า checkout และบริการปลายทางที่ล้มเหลวอาจกระทบคำสั่งซื้อ (`docs/scenario.md`, Quality goals and constraints) |
| 2. บันทึกแล้วส่งผ่าน RabbitMQ | checkout ไม่ต้องรอ MailFast; เพิ่มผู้รับงานใหม่ได้ง่าย | ต้องมีที่เก็บข้อมูลถาวรและดูแล RabbitMQ; ต้องป้องกันการทำงานซ้ำ |
| 3. แยก queue ตามงาน | checkout ไม่ต้องรอแต่ละบริการ | เมื่อเพิ่มงานใหม่ต้องแก้ `checkoutservice`; หากส่งเข้า queue ไม่สำเร็จ งานอาจหาย |

## Decision

เลือกข้อ 2 [Inferred: เป็นแบบที่จะสร้างเพิ่ม ไม่ใช่ระบบที่มีอยู่แล้ว]. เมื่อคำสั่งซื้อสำเร็จ ให้บันทึกข้อมูลคำสั่งซื้อและรายการ `OrderPlaced` ที่รอส่งลงที่เก็บข้อมูลถาวร **พร้อมกัน** ก่อนตอบลูกค้า จากนั้นให้ตัวส่งงานอ่านรายการที่รอส่งแล้วส่งเข้า RabbitMQ ซึ่งเป็นตัวกลางส่งงานให้แต่ละบริการทำงานต่อ การบันทึกพร้อมกันมีไว้ลดความเสี่ยงที่คำสั่งซื้อสำเร็จแล้วแต่งานหลังสั่งซื้อหายไป; ยังต้องเลือกที่เก็บข้อมูลและทดสอบวิธีนี้ก่อนเปิดใช้.

ปรับ `emailservice` เดิมให้รับงานแล้วส่งอีเมลผ่าน MailFast แทน logic จำลอง สร้าง `loyaltyservice` ให้แต้ม และให้ผู้รับอีกตัวบันทึกเหตุการณ์สำหรับ marketing (`docs/scenario.md`, หัวข้อ The new requirement). หากได้รับงานเดิมซ้ำ `loyaltyservice` ต้องตรวจรหัสคำสั่งซื้อก่อนให้แต้ม และบันทึกผลการตรวจพร้อมกับแต้ม เพื่อให้แต้มเพียงครั้งเดียว [Inferred: วิธีออกแบบตาม `docs/scenario.md`, Quality goals and constraints].

จำกัดการส่งอีเมลรวมไม่เกิน 50 ฉบับต่อวินาที หากช่วงพีก 200 คำสั่งซื้อต่อวินาทีเกิดต่อเนื่องและต้องส่งหนึ่งอีเมลต่อคำสั่งซื้อ งานอีเมลจะค้างเพิ่มประมาณ 150 รายการต่อวินาที จึงต้องติดตามคิวและตกลงเวลาส่งอีเมลที่ยอมรับได้ หรือขอเพิ่มโควตา MailFast (`docs/scenario.md`, หัวข้อ The new requirement และ Quality goals and constraints). แม้จะเก็บงานไว้ในคิว แต่ถ้า `loyaltyservice` หยุดนานกว่า 1 นาที ก็ยังรับประกันเวลาให้แต้มไม่ได้ ต้องตกลงวิธีรับมือกับเจ้าของข้อกำหนด (`docs/scenario.md`, Quality goals and constraints).

## Consequences

- **Positive:** checkout ไม่ต้องรอ MailFast จึงมีโอกาสรักษาเวลาเป้าหมายได้ (performance); งานอีเมลหรือแต้มที่ล้มเหลวชั่วคราวไม่ต้องทำให้คำสั่งซื้อล้มเหลว หากบันทึกงานไว้สำเร็จ (availability).
- **Positive:** เพิ่มงานหลังสั่งซื้อโดยเพิ่มผู้รับงาน ไม่ต้องเพิ่มการเรียกตรงใน `checkoutservice` (modifiability); ตรวจรหัสคำสั่งซื้อเพื่อป้องกันแต้มซ้ำ (correctness).
- **Negative:** ต้องดูแลที่เก็บข้อมูลถาวรและ RabbitMQ เพิ่ม (cost, operability); อีเมลอาจมาหลังคำสั่งซื้อ และคิวอาจยาวในช่วงพีก (latency).
- **Risks and mitigations:** ขั้นตอนปัจจุบันชำระเงินและสั่งจัดส่งก่อนส่งอีเมล หากเพิ่มการบันทึกข้อมูลถาวรหลังจากนั้นแล้วบันทึกไม่สำเร็จ ต้องมีวิธีกู้คืนก่อนเปิดใช้ (`docs/scenario.md`, หัวข้อ How it works today; correctness, reliability). หาก `loyaltyservice` หยุดเกิน 1 นาที จะผิดเป้าเวลาให้แต้ม จึงต้องแจ้งเตือนงานค้างและตกลงขอบเขตของข้อกำหนด (availability, correctness).

## Validation

- ทดสอบที่ 200 orders/s แล้ววัดว่า p95 ของ `PlaceOrder` ไม่เกิน 500 ms (`docs/scenario.md`, Quality goals and constraints).
- จำลอง MailFast, `loyaltyservice` และ analytics หยุดทำงาน ตรวจว่าคำสั่งซื้อยังสำเร็จและงานที่บันทึกไว้ไม่หาย; วัดว่าแต้มเข้าภายใน 1 นาทีในเงื่อนไขที่ตกลงกัน (`docs/scenario.md`, Quality goals and constraints).
- ส่งงาน `OrderPlaced` ซ้ำและตรวจว่าแต่ละรหัสคำสั่งซื้อได้แต้มครั้งเดียว; ทดสอบกรณีที่ส่งต่อไป RabbitMQ ไม่สำเร็จ และกรณีบันทึกข้อมูลล้มเหลวหลังชำระเงินหรือจัดส่ง.
- วัดว่าอีเมลส่งไม่เกิน 50 ฉบับต่อวินาที และติดตามจำนวนงานที่รอส่ง (`docs/scenario.md`, Quality goals and constraints).

## Open questions

- จะใช้ที่เก็บข้อมูลถาวรใด และจะกู้คืนอย่างไรหากชำระเงินหรือจัดส่งแล้วแต่บันทึกคำสั่งซื้อไม่สำเร็จ? Not evidenced in the provided files (`docs/scenario.md`).
- หาก `loyaltyservice` หยุดนานกว่า 1 นาที จะใช้แผนสำรองใด หรือเจ้าของข้อกำหนดยอมรับข้อยกเว้นแบบใด? Not evidenced in the provided files (`docs/scenario.md`, Quality goals and constraints).
- MailFast เพิ่มโควตาได้หรือไม่ และอีเมลยืนยันต้องส่งภายในเวลาเท่าใด? Not evidenced in the provided files (`docs/scenario.md`, Quality goals and constraints).

