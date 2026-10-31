**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ: เกศกนก อัศวโยธิน / รหัสนิสิต: 6870020021**
**agent ที่ใช้:** Codex
**โมเดล:** GPT-6-Sol Medium
**แพ็กเกจที่ใช้:** GPT Plus

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A |Codex/GPT-6-Sol Medium | `prompts/1-…` run A |`prompts/1-recover-architecture.md`<br>`system/online-boutique/protos/demo.proto`|✅|3| |
| 2 | 1-B | Codex/GPT-6-Sol Medium | `prompts/1-…` run B |`prompts/1-recover-architecture.md`<br>`system/online-boutique/protos/demo.proto`<br>`system/online-boutique/kubernetes-manifests/` |✅ |4| |
| 3 | 2-a | Codex/GPT-6-Sol Medium | `prompts/2-write-adr.md` run a, `context: none`, ความต้องการหลังสั่งซื้อ | `prompts/2-write-adr.md`<br>`docs/adr/template.md` | ✅ | 3 | ร่าง ADR โดยไม่มีไฟล์บริบท จึงยังไม่มีตัวเลขใน scenario |
| 4 | 2-b | Codex/GPT-6-Sol Medium | `prompts/2-write-adr.md` run b, ใช้ scenario และไฟล์บริบท | `prompts/2-write-adr.md`<br>`docs/adr/template.md`<br>`docs/scenario.md`<br>`docs/adr/0001-use-redis-for-cart-storage.md`<br>`system/online-boutique/src/checkoutservice/main.go` | ✅ | 5 | มีข้อมูลพอระบุข้อจำกัดและเปรียบเทียบทางเลือก |
| 5 | 3 | Codex/GPT-6-Sol Medium | `prompts/3-review-adr.md` รีวิว ADR | `prompts/3-review-adr.md`<br>`docs/adr/0002-adr-review.md`<br>`docs/scenario.md` | ✅ | 5 | ชี้ความเสี่ยงเรื่องงานค้างและเวลาให้แต้ม |
| 6 | โบนัส | Codex/GPT-6-Sol Medium | `prompts/bonus-recover-decision.md` กู้คืนเหตุผลการใช้ gRPC และ `demo.proto` | `prompts/bonus-recover-decision.md`<br>`docs/adr/template.md`<br>`system/online-boutique/protos/demo.proto`<br>`system/online-boutique/src/checkoutservice/main.go`<br>`system/online-boutique/src/frontend/main.go` | ✅ | 5 | แยกสิ่งที่เห็นในไฟล์กับเหตุผลที่ยังเป็นการอนุมาน |
| 7 | | | | | | | |
