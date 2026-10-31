Agent: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`)

> **การประชุมนี้เป็นการจำลองโดย AI ไม่ได้ประชุมจริง** AI เล่นทั้ง 4 บทบาท และ**ไม่ได้อ่านการ์ดบทบาทใน `docs/review-board/`** (AGENTS.md ห้าม) ข้อกังวลของ PO และ SRE จึงสร้างจาก `exercise/outputs/2b-adr.md` และ `docs/scenario.th.md` เท่านั้น ไม่ใช่ข้อมูลลับจากการ์ด ส่วนที่เป็นการคาดเดาติดป้าย **[Inferred]** และค่าที่ไม่มีในไฟล์เขียนว่า `<รอจากการ์ด/PO>`

# ส่วน B: บันทึกคณะกรรมการรีวิวสถาปัตยกรรม (จำลอง)

## 1. คณะกรรมการ

**ADR ที่รีวิว:** `exercise/outputs/2b-adr.md` (ADR-0002) โดย … (ยังไม่มีไฟล์ `docs/adr/0002-*.md`)
**ผลรีวิว AI:** `exercise/outputs/3-review.md`, verdict **Accept with changes**

| บทบาท | ผู้เล่น | ที่มาของจุดยืน |
|---|---|---|
| สถาปนิก | AI (จำลอง) | ADR ส่วน Decision, Consequences, Open questions |
| AI analyst | AI (จำลอง) | อ่าน `3-review.md` เท่านั้น |
| Product owner | AI (จำลอง) | `docs/scenario.th.md`, ADR Open questions; **ไม่มีการ์ด P** |
| SRE | AI (จำลอง) | ADR Consequences, Validation; **ไม่มีการ์ด S** |

## 2. การประชุม

- **สถาปนิก:** เลือก option 3 (RabbitMQ, 3 nodes, quorum queues) เพราะ MailFast p95 1.5 s ใส่ใน `PlaceOrder` ไม่ได้ และ 200 orders/s สูงกว่า limit 50 emails/s ยอมรับว่าถ้า publish ล้ม order ยังสำเร็จแต่ event หาย (ADR, Decision)
- **AI analyst:** อ่าน 3 Risk: event หายโดยไม่มี order store / cluster เดียวกันทำให้ backlog อีเมลบล็อก publisher / rate limiter ไม่รวมทุก replica พร้อมคำตัดสิน Accept with changes
- **PO:** "ถ้าอีเมลค้างเป็นชั่วโมง ลูกค้าจะคิดว่าสั่งไม่สำเร็จไหม" และ "แต้มหายยอมได้แค่ไหน เรื่องนี้ธุรกิจต้องตัดสิน ไม่ใช่ปล่อยเป็น log error"
- **SRE:** "ADR ลงวันที่ 2026-10-25 เหลือประมาณ 17 วันถึง 11.11 ใครเป็น on-call ของ RabbitMQ และจะซ้อมกู้ cluster เมื่อไร" และคัดค้านการเพิ่ม cluster ที่สองตามที่ AI เสนอ
- **สถาปนิก ตอบ:** รับ outbox แบบเล็กที่สุด และชี้ว่า `OrderPlaced` ส่ง `email` ให้ทุก consumer ซึ่งเป็น Open question ด้าน Privacy ที่ผลรีวิว AI ไม่ได้พูดถึง
- **ถาม AI analyst:** "AI เห็นระยะเวลาระบาย backlog ไหม" ตอบ: ไม่เห็น มีแค่ QS3 ที่ระบุว่ายังไม่มีเป้าหมาย delay / "AI เห็นเรื่องกำหนดการไหม" ตอบ: ไม่เห็น

## 3. บันทึกผลการตัดสินใจ

**สมาชิกและบทบาท:** สถาปนิก: … / AI analyst: … / PO: … / SRE: … (จำลองโดย AI)

**มติ:** **อนุมัติโดยมีเงื่อนไข**

**สิ่งที่ต้องแก้ไข:**
1. **ปิดช่อง event หาย:** เพิ่ม outbox ขนาดเล็กที่เขียนเฉพาะตอน publish ล้ม พร้อม job ส่งซ้ำ และเกณฑ์ทดสอบ "ปิดทั้ง cluster แล้ว event หาย = 0" ถ้าทำไม่ทัน 11.11 PO ต้องลงนามยอมรับความเสี่ยงแต้มหายเป็นลายลักษณ์อักษร
2. **กำหนดขนาด RabbitMQ และเป้าหมายอีเมล:** PO ให้ระยะเวลา peak `<รอจากการ์ด/PO>` และเป้าหมายการส่งอีเมล `<รอจากการ์ด/PO>` แล้วกำหนด disk ตามนั้น ทดสอบ memory/disk alarm จริง ใช้ cluster เดียวสำหรับ 11.11 และทบทวนการแยก cluster หลังเทศกาล
3. **แก้ email consumer และ claim ใน ADR:** ใช้ global rate limiter, ตั้ง concurrency จากการวัดจริง, retry ด้วย DLX และ TTL, ใช้ idempotency key ถ้า MailFast รองรับ และแก้ claim "RabbitMQ มี rate limiting ในตัว"
4. **ความพร้อมปฏิบัติการและข้อมูลส่วนบุคคล:** ระบุ on-call และ runbook ของ RabbitMQ ก่อนขึ้นระบบ และตัด `email` ออกจาก payload ที่ consumer ไม่ต้องใช้ หรือแยกการเข้าถึงต่อ consumer

**ใครยกประเด็นใด**

| # | ข้อกังวล | ผู้ยกประเด็น | อยู่ในผลรีวิวของ AI หรือไม่ | AI มีทางรู้ได้หรือไม่ เพราะเหตุใด |
|---|---|---|---|---|
| 1 | publish ล้มแล้ว event หาย loyalty ได้แต้ม 0 ครั้ง ไม่มี order store | ผลรีวิว AI | ใช่ (Risk 1) | รู้ได้: ADR Decision ขัดกับ `docs/scenario.th.md` แต่ระดับที่ธุรกิจยอมรับได้ AI ไม่รู้ |
| 2 | backlog อีเมลใช้ cluster เดียวกับ loyalty จน alarm บล็อก publisher | ผลรีวิว AI | ใช่ (Risk 2) | รู้ได้: คำนวณจาก 150 msg/s และความรู้ทั่วไปเรื่อง RabbitMQ |
| 3 | rate limiter ไม่รวมทุก replica ยิงเกิน 50 emails/s ได้ | ผลรีวิว AI | ใช่ (Risk 3) | รู้ได้: ADR ไม่ได้ระบุกลไก global |
| 4 | peak 1 ชั่วโมงสร้าง backlog 540,000 ฉบับ ระบายที่ 50/s ใช้อย่างน้อย 3 ชั่วโมง (540,000 / 50 = 10,800 s) แม้ไม่มี order ใหม่ ลูกค้าอาจติดต่อฝ่ายบริการ **[Inferred]** | PO | บางส่วน (QS3 บอกว่าไม่มีเป้าหมาย แต่ไม่คำนวณเวลาระบาย) | เวลาระบายรู้ได้จากตัวเลขที่มี แต่ระดับที่ลูกค้ายอมรับได้ไม่อยู่ในไฟล์ |
| 5 | ระยะ peak จริงและเป้าหมายส่งอีเมล | PO | ไม่ (AI สมมติ "ถ้า peak ยาว 1 ชั่วโมง") | ไม่ได้: Not evidenced in the provided files และเป็น Open question ของ ADR |
| 6 | เหลือประมาณ 17 วันถึง 11.11 จากวันที่ใน ADR เงื่อนไขของ AI อาจทำไม่ทัน | SRE | ไม่ | บางส่วน: วันที่อยู่ใน ADR แต่กำลังคนและแผน release ไม่อยู่ในไฟล์ |
| 7 | ใครเป็น on-call ของ RabbitMQ และทีมมีผู้มีประสบการณ์เพียง 2 คน การเพิ่ม cluster ที่สองจึงเพิ่มภาระ | SRE | บางส่วน (เพียงนับ operability เป็นข้อดีของ RabbitMQ) | บางส่วน: ทักษะทีมอยู่ใน scenario แต่ตาราง on-call ไม่อยู่ |
| 8 | `OrderPlaced` ส่ง `email` ให้ทุก consumer และส่งข้อมูลไป MailFast | สถาปนิก | ไม่ | รู้ได้: ADR ระบุ payload และมี Open question เรื่อง Privacy แต่ผลรีวิวไม่ได้หยิบมา |

**ข้อค้นพบที่ดีที่สุดของ AI** (ที่มนุษย์มองข้ามไป): claim "RabbitMQ มี rate limiting, retry และ DLQ ต่อ message ในตัว" ไม่ถูกต้อง claim นี้เป็นเหตุผลข้อหนึ่งที่เลือก RabbitMQ แทน Kafka และสถาปนิกไม่ได้ตรวจเอง

**ข้อค้นพบที่อ่อนที่สุดของ AI** (กว้างเกินไป ผิด หรือไม่เกี่ยวข้อง):
- **ข้อเสนอแยกด้วย vhost:** ไม่ตอบ Risk 2 ที่ AI ยกเอง เพราะ memory/disk alarm ทำงานระดับ node จึงบล็อก publisher ข้าม vhost **[Inferred จากความรู้ทั่วไปเรื่อง RabbitMQ, ไม่มีหลักฐานในไฟล์]**
- **การใช้ Little's law:** แทนค่า p95 และ tail 8 s ลงในสูตรที่ต้องใช้ค่าเฉลี่ย ตัวเลข 75 และ 400 จึงไม่ใช่ค่าสำหรับตั้งระบบจริง
