> Agent: Claude Code (VS Code extension) · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, context: none)

# ADR-0002: Use a publish-subscribe "OrderPlaced" event for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not evidenced in the provided files (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานต่อเนื่อง (post-order actions) เพิ่มเติม 3 อย่าง ตาม requirement:

| # | งานหลังสั่งซื้อ | ผู้รับผิดชอบ / ปลายทาง | หมายเหตุ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อ | **MailFast** (external email provider ใหม่) | มาแทน logic จำลอง (mock) ใน `emailservice` |
| 2 | ให้คะแนนสะสม (loyalty points) | `loyaltyservice` (ต้องสร้างใหม่) | |
| 3 | บันทึก analytics event "order placed" | ทีม marketing | ปลายทาง/ระบบ analytics ไม่ระบุ |

- Marketing คาดว่า **ปีหน้าจะมี post-order actions เพิ่ม** (SMS, fraud scoring, การอัปเดต recommendations) ดังนั้นจำนวนผู้ที่ต้องรู้ว่า "มีคำสั่งซื้อเกิดขึ้น" จะเพิ่มขึ้นเรื่อย ๆ
- ปัญหาที่ต้องตัดสินใจ: **service ที่สร้างคำสั่งซื้อควรแจ้ง/สั่งงาน post-order actions เหล่านี้อย่างไร** โดยไม่ทำให้การสั่งซื้อช้าลงหรือล้มเหลวเพราะ dependency ปลายทาง และเพิ่ม action ใหม่ได้ง่าย
- ชื่อ service ที่สร้างคำสั่งซื้อ, protocol ระหว่าง service ปัจจุบัน, และ infrastructure ที่มีอยู่ (เช่น มี message broker หรือไม่): Not evidenced in the provided files. **[Inferred]** มี service หนึ่งที่ทำหน้าที่ place order และปัจจุบันเรียก `emailservice` โดยตรง
- ตัวเลขด้าน latency, ปริมาณคำสั่งซื้อ, SLA ของ MailFast: Not evidenced in the provided files

## Decision drivers

เรียงจากสำคัญมากไปน้อย (**[Inferred]** จาก requirement เพราะไม่มี quality goals ที่ระบุไว้):

- **Modifiability / extensibility:** เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ service ที่สร้างคำสั่งซื้อ
- **Availability ของ checkout:** การสั่งซื้อต้องสำเร็จได้แม้ MailFast, `loyaltyservice` หรือระบบ analytics ล่มชั่วคราว
- **Performance (latency ของ checkout):** ลูกค้าไม่ควรต้องรอ external call ไปยัง MailFast
- **Reliability:** แต่ละ action ต้องไม่สูญหาย (ไม่ลืมส่งอีเมลหรือให้คะแนน) และทำซ้ำได้อย่างปลอดภัย (idempotent) ถ้ามีการ retry
- **Cost / operational complexity:** จำนวนชิ้นส่วนใหม่ที่ทีมต้องดูแล

## Considered options

1. **Synchronous orchestration** – service ที่สร้างคำสั่งซื้อเรียก MailFast adapter, `loyaltyservice` และ analytics ทีละตัวโดยตรง (เช่น gRPC/HTTP) ภายใน request ของการสั่งซื้อ
2. **Asynchronous point-to-point commands** – service ที่สร้างคำสั่งซื้อส่ง message แยกเข้า queue ของแต่ละ action (email queue, loyalty queue, analytics queue)
3. **Publish-subscribe domain event** – service ที่สร้างคำสั่งซื้อ publish event `OrderPlaced` เพียงครั้งเดียวไปยัง message broker; แต่ละ consumer (`emailservice` ที่เชื่อม MailFast, `loyaltyservice`, analytics) subscribe แยกกันเอง
4. **Central workflow orchestrator** – ใช้ workflow engine (เช่น saga/orchestration service) เป็นผู้ควบคุมลำดับขั้นของ post-order actions

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous orchestration | เรียบง่าย ไม่ต้องมี infrastructure ใหม่; รู้ผลทันที; debug ง่าย | latency ของ checkout = ผลรวมของทุก call (รวม external MailFast); ถ้าปลายทางตัวใดล่ม checkout ล้มหรือต้องเขียน error handling ซับซ้อน; ทุก action ใหม่ต้องแก้ service ที่สร้างคำสั่งซื้อ (coupling สูง) |
| 2. Async point-to-point commands | checkout ไม่ต้องรอปลายทาง; แต่ละ queue retry ได้อิสระ | service ที่สร้างคำสั่งซื้อยังต้องรู้จักทุก action และต้องแก้โค้ดทุกครั้งที่เพิ่ม action; ต้องมี broker |
| 3. Pub-sub `OrderPlaced` event | producer ไม่รู้จัก consumer → เพิ่ม SMS/fraud/recommendations ได้แค่เพิ่ม subscriber; checkout ไม่ต้องรอ; consumer ล่มไม่กระทบ checkout และประมวลผลย้อนหลังได้เมื่อกลับมา; ทีม marketing ได้ event เดียวกันเป็นแหล่งข้อมูล analytics | ต้องมี message broker ใหม่ (ดูแล, monitor, cost); eventual consistency (อีเมล/คะแนนมาช้ากว่าการสั่งซื้อเล็กน้อย); ต้องจัดการ duplicate delivery, schema versioning ของ event และ dead-letter; มองเห็น flow ทั้งหมดยากขึ้น ต้องมี tracing |
| 4. Central workflow orchestrator | เห็นสถานะทุกขั้นตอนในที่เดียว; รองรับขั้นตอนที่มีลำดับ/compensation (เช่น fraud scoring ที่อาจยกเลิกคำสั่งซื้อ) | เพิ่มชิ้นส่วนที่ซับซ้อนที่สุด; orchestrator ต้องรู้จักทุก action (ต้องแก้ทุกครั้งที่เพิ่ม); เกินความจำเป็นสำหรับ actions ปัจจุบันที่เป็นอิสระต่อกัน |

## Decision

เลือก **Option 3: Publish-subscribe `OrderPlaced` event** ผ่าน message broker

- เหตุผลหลัก: driver อันดับแรกคือ modifiability — marketing คาดว่าจะมี action เพิ่มอีกอย่างน้อย 3 อย่างในปีหน้า; pub-sub ให้เพิ่ม consumer ใหม่ได้โดย **ไม่แตะ** service ที่สร้างคำสั่งซื้อ ซึ่ง Option 1, 2, 4 ทำไม่ได้
- ทั้ง 3 actions ใน requirement เป็นอิสระต่อกันและไม่ต้องตอบผลกลับให้ลูกค้าระหว่าง checkout **[Inferred]** จึงทำแบบ asynchronous ได้ ช่วยทั้ง availability และ latency ของ checkout
- แนวทางย่อยที่เสนอ:
  - publish event ด้วย **transactional outbox** (หรือกลไกเทียบเท่า) เพื่อไม่ให้ event หายถ้า service ล่มหลังบันทึกคำสั่งซื้อ — ขึ้นกับว่าการสั่งซื้อถูกบันทึกลง data store หรือไม่ (Not evidenced in the provided files)
  - `emailservice` เปลี่ยนจาก mock เป็น consumer ที่เรียก MailFast; `loyaltyservice` และ analytics เป็น consumer แยก แต่ละตัวมี consumer group/queue ของตัวเอง
  - event payload มี `order_id` เป็น idempotency key และมี schema version
- ยังไม่เลือกเทคโนโลยี broker (เช่น RabbitMQ, Kafka, หรือ managed pub/sub) — ดู Open questions
- ถ้าในอนาคต fraud scoring ต้อง **บล็อก** คำสั่งซื้อก่อนยืนยัน ต้องทบทวนใหม่ (อาจต้องเป็น synchronous step หรือ Option 4 สำหรับขั้นนั้น)

## Consequences

- **Positive:** เพิ่ม SMS, fraud scoring, recommendations update ได้โดยเพิ่ม subscriber ใหม่เท่านั้น (quality attribute: modifiability)
- **Positive:** การสั่งซื้อสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่มชั่วคราว; message รอใน broker จนกว่า consumer จะกลับมา (quality attribute: availability)
- **Positive:** checkout ไม่ต้องรอ external call ไปยัง MailFast (quality attribute: performance — latency)
- **Positive:** แต่ละ consumer scale และ deploy แยกกันได้ (quality attribute: scalability, deployability)
- **Positive:** การเปลี่ยน email provider ในอนาคตจำกัดผลกระทบไว้ที่ `emailservice` (quality attribute: modifiability)
- **Negative:** ต้องติดตั้งและดูแล message broker ใหม่ รวมถึง monitoring (quality attribute: cost, operability)
- **Negative:** eventual consistency — อีเมลยืนยันและคะแนนสะสมอาจมาถึงช้ากว่าการสั่งซื้อ (quality attribute: usability / data consistency)
- **Negative:** flow กระจายตัว debug ยากกว่าการเรียกตรง (quality attribute: testability, observability)
- **Risks and mitigations:**
  - *Broker กลายเป็น single point of failure* → ใช้ broker แบบ HA/managed; outbox ทำให้ checkout ยังทำงานได้ถ้า broker ล่มชั่วคราว
  - *Duplicate delivery ทำให้ส่งอีเมลซ้ำหรือให้คะแนนซ้ำ* → consumer ทุกตัวต้อง idempotent โดยใช้ `order_id`
  - *Message ที่ประมวลผลไม่ได้ (poison message)* → dead-letter queue + alert
  - *MailFast ล่มหรือ rate limit* → retry แบบ exponential backoff ใน `emailservice`
  - *Schema ของ event เปลี่ยนแล้วทำให้ consumer พัง* → versioned schema และเปลี่ยนแบบ backward-compatible เท่านั้น
  - *ข้อมูลส่วนบุคคล (อีเมลลูกค้า) อยู่ใน event ที่ทีม marketing อ่าน* → จำกัดฟิลด์ใน payload หรือแยก topic/สิทธิ์การเข้าถึง

## Validation

- **Load test:** วัด latency ของ checkout ก่อน/หลังการเปลี่ยนแปลง; ต้องไม่เพิ่มขึ้นจาก baseline (ตัวเลขเป้าหมาย: ดู Open questions)
- **Fault injection:** ปิด MailFast stub, `loyaltyservice` และ broker ทีละตัว → การสั่งซื้อยังสำเร็จ และ action ถูกประมวลผลครบเมื่อระบบกลับมา
- **Metrics:** consumer lag, จำนวนใน dead-letter queue, อัตราการส่งอีเมลสำเร็จ, เวลาตั้งแต่สั่งซื้อจนถึงส่งอีเมล (end-to-end delay)
- **Modifiability check:** เพิ่ม consumer ตัวอย่าง (เช่น SMS stub) โดยไม่แก้ service ที่สร้างคำสั่งซื้อ
- **Review date:** ทบทวน ADR นี้เมื่อเริ่มออกแบบ fraud scoring (อาจต้องเป็น blocking step)

## Open questions

- service ใดเป็นผู้สร้างคำสั่งซื้อ และปัจจุบันเรียก `emailservice` ด้วย protocol อะไร? (Not evidenced in the provided files)
- ระบบมี message broker หรือ data store สำหรับคำสั่งซื้ออยู่แล้วหรือไม่? (มีผลต่อการใช้ transactional outbox)
- ใครเป็น deciders ของ ADR นี้?
- ปริมาณคำสั่งซื้อ (orders/sec, peak) และเป้าหมาย latency ของ checkout คือเท่าใด?
- MailFast มี SLA, rate limit, API (HTTP/SMTP), และรองรับ idempotency key หรือไม่?
- อีเมลยืนยันต้องถึงลูกค้าภายในกี่วินาที/นาที? คะแนนสะสมต้องแสดงทันทีหลังสั่งซื้อหรือไม่?
- ทีม marketing ใช้ระบบ analytics อะไร และต้องการข้อมูลใดใน event?
- ทีมมีประสบการณ์กับ broker ตัวใด (RabbitMQ, Kafka, managed pub/sub)? มีข้อจำกัดด้าน cost หรือ cloud platform หรือไม่?
- fraud scoring ในอนาคตต้องบล็อกคำสั่งซื้อก่อนยืนยันหรือไม่?
- มีข้อกำหนดด้าน privacy/PDPA สำหรับข้อมูลลูกค้าใน event หรือไม่?
