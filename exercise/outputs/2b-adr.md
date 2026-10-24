Agent: AI assistant using Copilot SDK in VS Code; Model: ไม่ระบุ

# ADR-0002: ใช้ RabbitMQ สำหรับงานหลังวางคำสั่งซื้อ

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** ยังไม่ระบุ

## Context and problem

ข้อกำหนด R-12 ระบุว่าเมื่อวางคำสั่งซื้อแล้ว ระบบต้องส่งอีเมลยืนยันผ่าน MailFast ซึ่งแทนที่ mock `emailservice`, มอบ loyalty points ผ่าน `loyaltyservice` ใหม่ และบันทึก analytics event สำหรับทีมการตลาด อีกทั้งคาดว่าจะมีงานหลังสั่งซื้อเพิ่ม เช่น SMS, fraud scoring และ recommendations update ( `docs/scenario.md` — The new requirement (R-12)).

ปัจจุบัน `checkoutservice.PlaceOrder` เรียก `paymentservice` และ `shippingservice` ก่อนส่งอีเมล และการส่งอีเมลที่ล้มเหลวจะถูกบันทึกเป็น warning โดยคำสั่งซื้อยังสำเร็จ (`docs/scenario.md` — How it works today; `system/online-boutique/src/checkoutservice/main.go` — `PlaceOrder`). แนวทางเรียกงานหลังสั่งซื้อแบบ synchronous ไม่เหมาะกับ MailFast ที่มี p95 1.5 วินาทีและ p99 8 วินาที ขณะที่ p95 ของ `PlaceOrder` ต้องไม่เกิน 500 ms (ปัจจุบันประมาณ 300 ms) นอกจากนี้ peak load คือ 200 orders/s แต่ MailFast จำกัดไว้ 50 emails/s (`docs/scenario.md` — Quality goals and constraints).

## Decision drivers

- **Availability:** คำสั่งซื้อต้องไม่ล้มเหลวเพราะ email, loyalty หรือ analytics ใช้งานไม่ได้ (`docs/scenario.md` — Quality goals and constraints).
- **Performance:** รักษา p95 ของ `PlaceOrder` ไม่เกิน 500 ms (`docs/scenario.md` — Quality goals and constraints).
- **Correctness:** ให้ loyalty points ภายใน 1 นาทีและไม่มอบซ้ำ แม้มีการส่งข้อความซ้ำหรือ retry (`docs/scenario.md` — Quality goals and constraints).
- **Scalability and modifiability:** รองรับ 200 orders/s และเพิ่มงานหลังสั่งซื้อในอนาคตได้โดยไม่ต้องเพิ่มการเรียกแบบ synchronous ใน checkout (`docs/scenario.md` — Quality goals and constraints; The new requirement (R-12)).
- **Operations and team fit:** ใช้ส่วนประกอบแบบ open-source ที่ self-host บน Kubernetes ได้ และคำนึงว่าทีมมีนักพัฒนาสองคนที่เคยใช้ RabbitMQ (`docs/scenario.md` — Quality goals and constraints).

## Considered options

1. เรียกบริการหลังสั่งซื้อแบบ synchronous จาก `checkoutservice`
2. เผยแพร่ event ผ่าน RabbitMQ แบบ durable แล้วให้ consumer แยกกันประมวลผลแต่ละงาน
3. บันทึกงานลง persistent work queue ที่สร้างและดูแลเอง แล้วให้ worker ดึงไปประมวลผล

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| เรียกแบบ synchronous | รูปแบบใกล้เคียงกับการเรียก gRPC ที่มีอยู่ใน `checkoutservice` (`system/online-boutique/src/checkoutservice/main.go` — `PlaceOrder`) และติดตามผลการเรียกได้ทันที | MailFast ที่ช้าและจำกัดอัตราการส่งอาจทำให้ latency ของ checkout เกินเป้าหมาย; การผูกความพร้อมใช้งานของงานปลายทางกับ checkout ขัดกับข้อกำหนดว่า checkout ต้องไม่ล้มเหลวเมื่อบริการเหล่านั้นขัดข้อง |
| RabbitMQ แบบ durable และ consumers แยกตามงาน | แยก checkout ออกจากเวลาตอบสนองและความพร้อมใช้งานของ consumer; คิวช่วยรองรับช่วงที่คำสั่งซื้อเข้ามาเร็วกว่าขีดจำกัดของ MailFast; เพิ่ม consumer สำหรับงานใหม่ได้; เป็น open-source และมีผู้มีประสบการณ์ในทีม | เพิ่ม broker และภาระการปฏิบัติการ; การส่งซ้ำเกิดขึ้นได้ จึงต้องทำ consumer ให้ idempotent; ต้องกำหนดการจัดเก็บ event ที่ทนทานและแนวทางกู้คืนเมื่อ broker ใช้งานไม่ได้ |
| Persistent work queue ที่สร้างเอง | ควบคุมรูปแบบการจัดเก็บ, retry และการทำงานได้โดยตรง; ไม่ต้องเพิ่ม broker | ต้องสร้างและดูแลกลไกคิว, การแข่งขันระหว่าง worker, retry และการกู้คืนเอง; เพิ่มภาระการพัฒนาและปฏิบัติการ และไม่ใช้ประโยชน์จากประสบการณ์ RabbitMQ ที่มีอยู่ |

## Decision

เลือก **RabbitMQ แบบ self-hosted บน Kubernetes** เป็นช่องทางส่งงานหลังวางคำสั่งซื้อ โดย `checkoutservice` เผยแพร่ event ที่มี `order_id` หลังคำสั่งซื้อสำเร็จ แล้วให้ consumer แยกกันจัดการการส่งอีเมลผ่าน MailFast, การให้ points โดย `loyaltyservice` และ analytics event การแยก consumer ช่วยให้เพิ่ม SMS, fraud scoring และ recommendations update ได้โดยไม่เพิ่มการเรียกงานเหล่านี้ในเส้นทาง synchronous ของ checkout (ข้อกำหนด R-12 ใน `docs/scenario.md`).

กำหนดให้ข้อความและคิวมีความทนทาน พร้อม retry และ dead-letter handling ส่วน consumer ต้องทำงานซ้ำได้อย่างปลอดภัย: `loyaltyservice` ใช้ `order_id` ป้องกันการให้ points ซ้ำ และ MailFast consumer จำกัดอัตราการส่งไม่เกิน 50 emails/s. การบันทึก event จาก checkout ต้องใช้รูปแบบที่ไม่ทำให้คำสั่งซื้อสำเร็จแต่ event สูญหาย เช่น transactional outbox; อย่างไรก็ตาม context ที่ให้มายังไม่ระบุ order database หรือกลไก persistence จึงต้องยืนยันรายละเอียดนี้ก่อนนำไปใช้จริง.

## Consequences

- **Positive:** งานหลังสั่งซื้อไม่ต้องรอ MailFast หรือ consumer ทำงานเสร็จ จึงช่วยควบคุม latency ของ checkout (performance); คิวช่วยรองรับความต่างระหว่าง peak 200 orders/s กับ MailFast ที่จำกัด 50 emails/s (scalability).
- **Positive:** การขัดข้องชั่วคราวของ consumer สามารถจัดการด้วยคิวและ retry โดยไม่ทำให้ checkout ต้องเรียกบริการเหล่านั้นโดยตรง (availability); เพิ่ม consumer สำหรับงานในอนาคตได้โดยลดการเปลี่ยนแปลงใน checkout (modifiability).
- **Positive:** ใช้ RabbitMQ แบบ self-hosted ซึ่งสอดคล้องกับความต้องการ open-source และประสบการณ์บางส่วนในทีม (team fit, operational fit).
- **Negative:** ต้องดูแล RabbitMQ, การเฝ้าระวังคิว และการกู้คืนเมื่อ broker หรือ consumer ขัดข้อง (operability, cost).
- **Negative:** งานหลังสั่งซื้อจะเสร็จภายหลังการตอบกลับ checkout ไม่ใช่ทันที; ต้องเฝ้าระวังให้ loyalty สำเร็จภายใน 1 นาที (timeliness).
- **Risks and mitigations:** RabbitMQ อาจส่งข้อความซ้ำ; ใช้ idempotency key จาก `order_id` และบันทึกสถานะการให้ points อย่าง atomic กับการเปลี่ยนยอด points (correctness). MailFast มีอัตรารับสายต่ำกว่า peak orders; จำกัดอัตราการส่ง, เฝ้าระวัง backlog และกำหนดการแจ้งเตือนเมื่อคิวเพิ่มหรืออายุงานค้างเสี่ยงเกิน SLA (performance, timeliness).
- **Risks and mitigations:** หากบันทึกคำสั่งซื้อกับ event ไม่ได้อย่างทนทาน คำสั่งซื้ออาจสำเร็จแต่ไม่มีงานให้ consumer; ออกแบบ transactional outbox หรือกลไกที่ให้ความทนทานเทียบเท่าก่อนเปิดใช้งาน (reliability). วิธีผูก persistence นี้ยังไม่ยืนยันจาก context ที่ให้มา.

## Validation

- ทดสอบ load ที่ 200 orders/s และยืนยันว่า p95 ของ `PlaceOrder` ไม่เกิน 500 ms.
- จำลอง MailFast ใช้เวลาตอบสนองตาม p95 1.5 วินาที/p99 8 วินาที, ล้มเหลว 0.5% และจำกัด 50 emails/s; ยืนยันว่า checkout ไม่รอ MailFast, อัตราส่งไม่เกิน limit และ backlog ถูกระบายได้.
- จำลองการส่ง event ซ้ำและ retry; ยืนยันว่าแต่ละ `order_id` ได้ loyalty points เพียงครั้งเดียว และได้รับภายใน 1 นาที.
- ปิดหรือทำให้ consumer และ broker ขัดข้องชั่วคราว; ยืนยันว่า email, loyalty และ analytics ที่ขัดข้องไม่ทำให้คำสั่งซื้อถูกปฏิเสธ และงานที่บันทึกไว้อย่างทนทานกลับมาประมวลผลได้.
- ตรวจสอบ metrics/alerts สำหรับความลึกและอายุของคิว, dead-letter messages, อัตราการส่ง MailFast และเวลาจน loyalty สำเร็จ.

## Open questions

- ปัจจุบันจะจัดเก็บคำสั่งซื้อและ transactional outbox ที่ใด และจะรับประกันการบันทึกคำสั่งซื้อกับ event อย่างไร? (`system/online-boutique/src/checkoutservice/main.go` — `PlaceOrder` แสดงการสร้าง order result และเรียกบริการ แต่ context ที่ให้มาไม่ได้แสดง order database หรือ persistence สำหรับ outbox.)
- analytics event ต้องมี schema, retention และปลายทางใดบ้าง?
- เมื่อ MailFast หรือ `loyaltyservice` ใช้งานไม่ได้ต่อเนื่อง จะกำหนด retry policy, dead-letter recovery และเจ้าของการตรวจสอบงานค้างอย่างไร?
- มี SLA สำหรับการส่งอีเมลและการบันทึก analytics หรือไม่? ข้อกำหนดระบุ SLA หนึ่งนาทีเฉพาะ loyalty points.
- ทีมจะกำหนดเป้าหมาย availability, replication และทรัพยากร Kubernetes สำหรับ RabbitMQ อย่างไร?
