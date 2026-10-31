Agent: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`)

# Architecture review: ADR-0002 (`exercise/outputs/2b-adr.md`) แบบ ATAM

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QS1 Performance | ลูกค้า | เรียก `PlaceOrder` | ช่วง 11.11 ที่ 200 orders/s, RabbitMQ ปกติ | ตัดบัตร, ship, publish `OrderPlaced` พร้อม publisher confirm แล้วตอบลูกค้า | p95 ≤ 500 ms (การ publish ต้องไม่เกิน headroom ประมาณ 200 ms) |
| QS2 Availability / correctness | RabbitMQ node ล่ม 1 ใน 3 หรือ broker เข้าสู่ memory/disk alarm | `PlaceOrder` publish event | ช่วง peak | order สำเร็จ และ event ไม่หาย | order ล้ม 0 ครั้ง, loyalty ได้แต้ม 1 ครั้งพอดีภายใน 60 s |
| QS3 Throughput / reliability | ลูกค้า 200 คน/s | สร้างงานอีเมล 200 ฉบับ/s | MailFast จำกัด 50 emails/s, ล้ม 0.5 %, p99 8 s | queue เก็บ backlog แล้วทยอยส่ง | เรียก MailFast ≤ 50/s, อีเมลหาย 0 ฉบับ, delay ≤ (ยังไม่มีเป้าหมายใน `docs/scenario.md`) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | เมื่อ publish ล้ม checkout จะ "log แล้วคืน order สำเร็จ" (ADR, Decision) และไม่มี order store ให้ reconcile | event ที่หายทำให้ loyalty ได้แต้ม **0 ครั้ง** ซึ่งละเมิด exactly once ใน `docs/scenario.md` ADR บอกว่าเป็น "พฤติกรรมเดียวกับ email ตอนนี้" แต่ email ไม่มีข้อกำหนด exactly once ส่วน loyalty มี |
| **Risk** | ใช้ RabbitMQ cluster เดียวกันสำหรับ backlog อีเมลและ queue loyalty | backlog เพิ่ม 150 msg/s (9,000/นาที) ถ้า peak ยาว 1 ชั่วโมงจะค้างประมาณ 540,000 ข้อความ ถ้า quorum queue ใช้ disk หรือ memory จนถึง watermark RabbitMQ จะ **block publisher ทุกตัว** ทำให้ checkout publish timeout ทั้งระบบ และ event ของ loyalty หายไปด้วย |
| **Risk** | การจำกัด 50 emails/s ทำ "ด้วย prefetch และ rate limiter" แต่ไม่ได้บอกว่าจำกัดรวมทุก replica อย่างไร | prefetch จำกัดแค่จำนวนข้อความที่ค้างอยู่ ไม่ได้จำกัด rate ถ้า scale `emailservice` เป็น N replica ที่แต่ละตัวมี limiter ของตัวเอง จะยิงเกิน rate limit ของ MailFast |
| **Sensitivity point** | timeout ของ publish และ publisher confirm กับ quorum queue | confirm ต้องรอให้ข้อมูลถูก replicate ไปถึง majority ขณะเลือก leader ใหม่ (QS2) tail latency อาจเกิน 200 ms ค่า timeout ที่เลือกเป็นตัวตัดสินว่าจะผิด p95 หรือทำให้ event หาย |
| **Sensitivity point** | concurrency หรือ prefetch ของ email consumer | ตาม Little's law ถ้าส่ง 50/s และ latency p95 1.5 s ต้องมี in-flight ประมาณ 75 ข้อความ และช่วง tail 8 s ต้องมีถึงประมาณ 400 ถ้าตั้งต่ำไป จะไม่ได้ถึง 50/s และ backlog จะไม่ลดลง |
| **Sensitivity point** | throughput ของ consumer ใน `loyaltyservice` | ต้อง ≥ 200 msg/s อย่างต่อเนื่อง ถ้าต่ำกว่านี้ lag จะสะสมและเกิน 60 s ภายในไม่กี่วินาทีของ peak |
| **Trade-off point** | นโยบายเมื่อ publish ล้ม: ให้ order สำเร็จ หรือให้ order ล้ม | availability ของ order ได้ แต่ correctness ของ loyalty เสีย ADR ยอมรับ trade-off นี้แต่ยังไม่ได้ตัดสิน |
| **Trade-off point** | RabbitMQ แทน Kafka | ได้เรื่อง operability และทักษะทีม แต่ไม่มี replay ซึ่ง analytics และ recommendations update ปีหน้าอาจต้องใช้ (modifiability) |
| **Non-risk** | เอา MailFast ออกจาก critical path | MailFast p95 1.5 s อย่างเดียวก็เกิน 500 ms แล้ว จึงเรียกแบบ synchronous ไม่ได้ เหตุผลถูกต้องและอิงตัวเลขจริง |
| **Non-risk** | topic exchange ที่แต่ละ action มี queue ของตัวเอง | เพิ่ม SMS หรือ recommendations ได้โดยไม่ต้องแก้ checkout และ consumer ที่ล่มไม่กระทบ queue ของ consumer ตัวอื่น |
| **Non-risk** | idempotency ด้วย unique key บน `order_id` | `order_id` เป็น UUID ที่สร้างครั้งเดียวต่อ order ทำให้ redelivery ไม่ให้แต้มซ้ำ (ใช้ได้ถ้าบันทึกแต้มกับ key ใน transaction เดียวกัน) |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** (a) เพิ่ม storage ขนาดเล็กให้ checkout เพื่อทำ transactional outbox หรือ durable spool ADR ปัดตกด้วยเหตุผลว่า "ไม่มี DB" ทั้งที่เพิ่ม DB ได้ (b) แยก email backlog ไปไว้ vhost หรือ cluster อื่นจาก loyalty (c) RabbitMQ Streams สำหรับ analytics ที่ต้อง replay (d) ขอเพิ่ม rate limit หรือใช้ batch API ของ MailFast
- **Claim ที่ไม่มีหลักฐานหรือน่าจะผิด:** "RabbitMQ มี rate limiting, retry และ DLQ ต่อ message ในตัว" (Decision, เหตุผล) RabbitMQ ไม่มี rate limiting ในตัว และ exponential backoff ต้องสร้างเองจาก DLX ร่วมกับ TTL หรือใช้ plugin claim นี้เป็นเหตุผลหลักที่เลือก RabbitMQ แทน Kafka จึงต้องแก้
- **Claim ที่ขัดกันเอง:** "option 3 ผ่าน driver 1–4 พร้อมกัน" แต่ส่วน Risks ยอมรับเองว่า event หายได้เมื่อ broker ล่ม จึงยังไม่ผ่าน driver 3
- **Consequences ที่ไม่ได้ระบุ:** ถ้า loyaltyservice ล่มเกิน 60 s จะผิด SLA 1 นาทีแน่นอน ADR ไม่ได้บอกว่ายอมรับได้หรือไม่ / at-least-once ทำให้อีเมลซ้ำได้ ถ้า MailFast ไม่มี idempotency key / retry 0.5 % กิน rate budget 50/s ด้วย
- **Validation อ่อน:** การทดสอบ "ปิดทั้ง cluster แล้วนับ event ที่หาย" ไม่มีเกณฑ์ผ่านหรือไม่ผ่าน และไม่มีการทดสอบกรณี memory/disk alarm

## 4. Verdict: **Accept with changes**

1. **ปิดช่องที่ event หาย:** เพิ่ม outbox หรือ durable spool ให้ checkout (หรือมี order store สำหรับ reconcile) แล้วกำหนดเกณฑ์ชัดเจนว่า "event หาย = 0" ในการทดสอบ broker failure
2. **แยก backlog อีเมลออกจาก path ของ loyalty:** ใช้ vhost หรือ cluster แยก กำหนดขนาด disk สำหรับ backlog ประมาณ 540k ข้อความต่อชั่วโมงของ peak และทดสอบกรณี memory/disk alarm ร่วมกับ publish timeout
3. **ออกแบบ email consumer ให้ชัด:** ใช้ global rate limiter สำหรับทุก replica (หรือ consumer เดียว) ตั้ง concurrency ตาม Little's law ทำ retry ด้วย DLX และ TTL ใช้ idempotency key กับ MailFast และแก้ claim เรื่อง "rate limiting ในตัว"
