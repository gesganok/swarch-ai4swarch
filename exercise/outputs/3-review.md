Agent: Codex (GPT-6)

## Quality-attribute scenarios

| Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|
| ลูกค้า | เรียก `checkoutservice.PlaceOrder` | ช่วงพีก 200 คำสั่งซื้อ/วินาที; MailFast มี p95 1.5 วินาที | บันทึกคำสั่งซื้อและ `OrderPlaced` แล้วตอบโดยไม่รออีเมล | p95 ไม่เกิน 500 ms (ปัจจุบันประมาณ 300 ms); `docs/scenario.md` (Quality goals and constraints), `docs/adr/0002-adr-review.md` (Decision) |
| ลูกค้า | สั่งซื้อขณะ MailFast, `loyaltyservice` หรือ analytics หยุดทำงาน | การทำงานปกติที่บริการปลายทางขัดข้อง | คำสั่งซื้อยังสำเร็จ; งานที่บันทึกไว้รอส่งซ้ำ | คำสั่งซื้อไม่ล้มเหลวเพราะบริการเหล่านี้; แต้มหนึ่งครั้งต่อคำสั่งซื้อและภายใน 1 นาทีตามข้อกำหนด; `docs/scenario.md` (Quality goals and constraints) |
| ระบบส่งงาน | ส่ง `OrderPlaced` ซ้ำหลังส่งต่อหรือรับงานล้มเหลว | RabbitMQ และผู้รับงานกลับมาทำงาน | `loyaltyservice` ตรวจรหัสคำสั่งซื้อและบันทึกผลพร้อมแต้ม | ไม่ให้แต้มซ้ำ; `docs/adr/0002-adr-review.md` (Decision, Validation), `docs/scenario.md` (Quality goals and constraints) |

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | แยก MailFast ออกจากทางตอบ `PlaceOrder` และยอมรับว่าอีเมลมาทีหลัง (`docs/adr/0002-adr-review.md`, Decision, Consequences) | MailFast p95 1.5 วินาทีสูงกว่าเป้า checkout 500 ms (`docs/scenario.md`, Quality goals and constraints) |
| Risk | ADR ยังไม่กำหนดที่เก็บถาวรและขอบเขตธุรกรรมที่บันทึกคำสั่งซื้อกับ `OrderPlaced` พร้อมกัน; การชำระเงินและจัดส่งเกิดก่อนขั้นตอนนี้ (`docs/adr/0002-adr-review.md`, Decision, Open questions; `docs/scenario.md`, How it works today) | หากบันทึกล้มเหลวหลังชำระเงินหรือจัดส่ง อาจเกิดคำสั่งซื้อที่กู้คืนไม่ได้หรือไม่มีงานหลังสั่งซื้อ; แนวทางกู้คืนยังเป็นคำถามเปิด |
| Sensitivity point | เวลาเขียนที่เก็บถาวรอยู่ในทางตอบ checkout และการส่งแต้มขึ้นกับเวลาค้างตั้งแต่ outbox ถึง `loyaltyservice` (`docs/adr/0002-adr-review.md`, Decision) | ต้องวัดทั้ง p95 ไม่เกิน 500 ms และเวลาให้แต้มไม่เกิน 1 นาที; `docs/scenario.md` (Quality goals and constraints) |
| Trade-off point | RabbitMQ และที่เก็บถาวรช่วยแยกผู้รับงาน แต่เพิ่มภาระดูแลและจุดขัดข้อง (`docs/adr/0002-adr-review.md`, Considered options, Consequences) | เพิ่ม modifiability และความทนต่อปลายทางล่ม แลกกับ cost และ operability ของทีม 4 คน (`docs/scenario.md`, Quality goals and constraints) |
| Risk | โครงสร้างการกระจายงานและการยืนยันรับงานของผู้รับแต่ละรายยังไม่ระบุ (`docs/adr/0002-adr-review.md`, Decision) | MailFast รับได้ 50 อีเมล/วินาทีเมื่อมีสูงสุด 200 คำสั่งซื้อ/วินาที; หากงานอีเมลค้างร่วมกับงานแต้ม อาจกระทบเป้า 1 นาที (`docs/scenario.md`, Quality goals and constraints) |

## Missing or weak

- ระบุที่เก็บข้อมูลถาวร ขอบเขตธุรกรรม และวิธีกู้คืนเมื่อบันทึกหลังชำระเงินหรือจัดส่งไม่สำเร็จ; ปัจจุบัน ADR ยอมรับช่องว่างนี้แต่ยังเลือกแบบที่ต้องพึ่งการบันทึกพร้อมกัน (`docs/adr/0002-adr-review.md`, Decision, Open questions).
- กำหนดการกระจาย `OrderPlaced` ไปยังผู้รับแต่ละราย การยืนยันรับงาน การส่งซ้ำ และการแยกงานอีเมลจากงานแต้ม; ขั้นตอนเหล่านี้ยังเป็น **[Inferred]** จากคำว่า RabbitMQ “ส่งงานให้แต่ละบริการ” (`docs/adr/0002-adr-review.md`, Decision).
- หาก `loyaltyservice` หยุดเกิน 1 นาที จะรับประกันเวลาให้แต้มไม่ได้; ADR ระบุปัญหาแต่ยังไม่มีขอบเขตข้อยกเว้นหรือวิธีรับมือที่ตกลงแล้ว (`docs/adr/0002-adr-review.md`, Decision, Open questions; `docs/scenario.md`, Quality goals and constraints).
- เปรียบเทียบทางเลือกส่งเข้า broker แบบ asynchronous โดยตรงพร้อม publisher confirmation และแนวทางจัดการงานที่ส่งไม่สำเร็จให้ชัดขึ้น; ข้อ 3 ระบุเพียงคิวแยกตามงาน จึงยังเทียบต้นทุนและความเสี่ยงกับ outbox ได้ไม่ครบ (`docs/adr/0002-adr-review.md`, Considered options).
- ข้อความเรื่อง Redis อ้าง `docs/adr/0001-use-redis-for-cart-storage.md` ซึ่งอยู่นอกไฟล์ที่อนุญาตให้ตรวจในงานนี้ จึง **Not evidenced in the provided files**; อย่าใช้เป็นเหตุผลที่ตรวจยืนยันแล้ว (`docs/adr/0002-adr-review.md`, Context and problem).

## Verdict

**Accept with changes.** ก่อนรับรองให้ (1) กำหนดธุรกรรมและการกู้คืนหลัง payment/shipping, (2) ระบุการกระจายงานแยกผู้รับพร้อม retry/ack และพิสูจน์ว่าแต้มเข้าภายใน 1 นาทีแม้อีเมลค้าง, (3) ทดสอบ p95 ที่ 200 orders/s รวมเวลาบันทึกถาวร พร้อมทดสอบกรณีส่งซ้ำและบริการปลายทางล่ม (`docs/adr/0002-adr-review.md`, Decision, Validation; `docs/scenario.md`, Quality goals and constraints).
