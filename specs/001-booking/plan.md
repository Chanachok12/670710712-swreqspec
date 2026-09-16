# แผนงานฟีเจอร์: จองคิวตรวจสุขภาพ (Booking)

## 1. สรุปแนวทาง
ฟีเจอร์นี้ช่วยให้ผู้รับบริการที่ยืนยันตัวตนแล้วเลือกแพ็กเกจ วัน และช่วงเวลาตรวจสุขภาพ ได้เห็นช่วงเวลาว่างและจำนวนที่นั่งคงเหลือ จากนั้นระบบจะตรวจสอบการจองซ้ำในวันเดียวกัน บันทึกการจองให้เป็นรายการเดียว และส่งคำขอแจ้งเตือนแบบ asynchronous โดยไม่ให้ขั้นตอนการส่งข้อความหยุดการทำงานหลัก ระบบจะออกหมายเลขคิวตามวันที่บริการและบันทึก audit log ทุกครั้งที่เข้าถึงข้อมูลการจองและประวัติผู้รับบริการ

## 2. เทคโนโลยีที่ใช้
| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| MySQL | CON-TECH-01 | ใช้เก็บข้อมูลการจอง โควตา และ audit log ตามมาตรฐานโรงพยาบาล |
| Python FastAPI | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้พัฒนา API สำหรับค้นหาช่วงเวลา วางแผนการจอง และยืนยันการจอง |
| React (Vite) | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้แสดงหน้าจอเลือกแพ็กเกจ วัน และช่วงเวลา พร้อมจำนวนที่นั่งคงเหลือ |
| Message queue / async worker | IF-NOT-01, NFR-REL-02 | จัดการส่ง SMS/LINE แบบ asynchronous และการ retry ภายใน 5 นาที |
| TLS 1.2+ | NFR-SEC-01 | ใช้ HTTPS/TLS ระหว่าง client และ API และระหว่าง service กับ notification worker |

## 3. โมเดลข้อมูล
| Entity | ฟิลด์หลัก | รองรับ FR / Constraint |
|---|---|---|
| Patient | patient_id, hn, full_name, identity_verified_at | IF-IDP-01, IF-HIS-01; ไม่เก็บเลขบัตรประชาชนในตารางการจอง |
| Booking | booking_id, patient_id, hn, package_id, service_date, slot_id, queue_no, status, created_at, confirmed_at | FR-BKG-02, FR-BKG-04, FR-BKG-05, AC-BKG-01, AC-BKG-02, AC-BKG-04 |
| TimeSlot | slot_id, service_date, start_time, end_time, capacity, package_id | FR-BKG-01, FR-BKG-03, FR-BKG-06 |
| BookingAuditLog | audit_id, actor_user_id, patient_hn, access_time, action, booking_id | DOM-PDPA-01, AC-BKG-06 |
| NotificationMessage | notification_id, booking_id, channel, payload, status, retry_count, next_retry_at | IF-NOT-01, FR-BKG-05, NFR-REL-02 |

หมายเหตุ: ตารางการจองจะเก็บ HN เป็นข้อมูลอ้างอิงภายในระบบเท่านั้นและจะไม่เก็บเลขบัตรประชาชนตาม IF-HIS-01

## 4. API / หน้าจอ
- GET /api/booking/slots?service_date=YYYY-MM-DD&package_id=... -> แสดงช่วงเวลาว่างพร้อมจำนวนที่นั่งคงเหลือ, รองรับ FR-BKG-01, FR-BKG-06
- GET /api/booking/calendar?from=YYYY-MM-DD&to=YYYY-MM-DD&package_id=... -> แสดงตารางวันและช่วงเวลา 30 วันข้างหน้า, รองรับ FR-BKG-01
- POST /api/booking/check-duplicate -> ตรวจสอบว่าผู้รับบริการมีคิวที่ยังไม่ได้ใช้ในวันเดียวกันหรือไม่, รองรับ FR-BKG-02
- POST /api/booking/confirm -> ยืนยันการจองและบันทึก booking, queue_no, ลดความจุ, ส่งข้อความแจ้งเตือนแบบ async, รองรับ FR-BKG-03, FR-BKG-04, FR-BKG-05, AC-BKG-01, AC-BKG-03, AC-BKG-04
- POST /api/notifications/retry -> worker ส่งซ้ำข้อความที่ไม่สำเร็จภายใน 5 นาที, รองรับ NFR-REL-02, FR-BKG-05
- GET /api/booking/{booking_id}/audit -> ดึง audit log สำหรับการตรวจสอบและการเข้าถึงข้อมูล, รองรับ DOM-PDPA-01, AC-BKG-06
- หน้า Booking Form: เลือกแพ็กเกจ วัน และช่วงเวลา มองเห็นจำนวนที่นั่งคงเหลือ และแสดงข้อผิดพลาด “ช่วงเวลาเต็ม” พร้อม 3 ตัวเลือกที่ใกล้ที่สุดในวันเดียวกัน, รองรับ FR-BKG-01, FR-BKG-03, FR-BKG-06

## 5. ตารางตรวจ Constraints
| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| CON-TECH-01 | MySQL เป็น datastore ของ booking, time slot และ audit log | ใช้แล้ว |
| DOM-PDPA-01 | BookingAuditLog, tracing actor_user_id + access_time + patient_hn, เก็บไม่น้อยกว่า 1 ปี | ใช้แล้ว |
| IF-IDP-01 | Precondition ในผังการจองและ API check/confirm ให้ตรวจสอบผลยืนยันตัวตนก่อนเข้าถึงข้อมูลผู้รับบริการ | ใช้แล้ว |
| IF-HIS-01 | Patient entity ใช้ HN จาก HIS และไม่เก็บเลขบัตรประชาชนใน Booking | ใช้แล้ว |
| IF-NOT-01 | NotificationMessage + async worker เพื่อส่ง SMS/LINE โดยไม่ให้หยุดการจอง | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria
| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-BKG-01 | test_AC_BKG_01_booking_success_reduces_capacity | ตั้ง service_date=2026-09-16, slot 09:00 capacity=1, ยืนยันจองแล้ว ตรวจสอบการบันทึก booking, queue_no, และ capacity จาก 1 เป็น 0 |
| AC-BKG-02 | test_AC_BKG_02_duplicate_active_booking_blocked | สร้างคิวที่ยังไม่ได้ใช้ในวันเดียวกันให้ผู้รับบริการคนเดิม แล้วลองจองอีกครั้ง ตรวจสอบการปฏิเสธและแสดงหมายเลขคิวเดิม |
| AC-BKG-03 | test_AC_BKG_03_slot_full_offers_three_alternatives | ตั้ง slot ที่เลือกมี capacity=0 และมีผู้ใช้อีกคนยืนยันก่อนแล้ว ให้ยืนยันอีกคน ตรวจสอบข้อความ “ช่วงเวลาเต็ม” และมี 3 ตัวเลือกในวันเดียวกัน |
| AC-BKG-04 | test_AC_BKG_04_notification_failure_keeps_booking | จำลอง notification fail แล้วยืนยันจอง ตรวจสอบว่าการจองยังถูกบันทึก แสดง queue_no และมี notification retry queued ภายใน 5 นาที |
| AC-BKG-05 | test_AC_BKG_05_slot_lookup_p95_under_2s | พรอ้่มจำลอง 200 คนเรียก GET /api/booking/calendar และวัด p95 < 2 วินาที |
| AC-BKG-06 | test_AC_BKG_06_audit_log_recorded | เรียก API read booking หรือเข้าถึงข้อมูลผู้รับบริการ ตรวจสอบว่าเกิด AuditLog อย่างน้อย 1 รายการพร้อม actor_user_id, access_time, patient_hn |

## 7. ลำดับงาน
1. สร้าง schema สำหรับ Patient, Booking, TimeSlot, BookingAuditLog, NotificationMessage และ index ที่จำเป็น, รองรับ FR-BKG-01, FR-BKG-02, DOM-PDPA-01
2. สร้าง API ดึงช่วงเวลาว่าง 30 วันพร้อมจำนวนที่นั่งคงเหลือ, รองรับ FR-BKG-01, NFR-PERF-01
3. สร้าง logic ตรวจสอบคิวที่ยังไม่ได้ใช้ในวันเดียวกัน และแสดงหมายเลขคิวเดิม, รองรับ FR-BKG-02, AC-BKG-02
4. สร้าง flow ยืนยันจองแบบ atomic: ตรวจ capacity -> บันทึก booking -> ลดที่นั่ง -> สร้าง queue_no -> ส่ง notification request, รองรับ FR-BKG-03, FR-BKG-04, AC-BKG-01, AC-BKG-03
5. สร้าง async worker สำหรับ SMS/LINE retry และ queue resend ภายใน 5 นาที, รองรับ FR-BKG-05, NFR-REL-02, AC-BKG-04
6. สร้างหน้า UI สำหรับเลือกแพ็กเกจ วัน และช่วงเวลา พร้อมข้อความ “ช่วงเวลาเต็ม” และตัวเลือก 3 ช่วงเวลาใกล้เคียง, รองรับ FR-BKG-03, FR-BKG-06, AC-BKG-03
7. สร้าง audit logging และ endpoint สำหรับตรวจสอบ log, รองรับ DOM-PDPA-01, AC-BKG-06
8. ทดสอบสคริปต์และ performance validation ครบตาม AC-BKG-01 ถึง AC-BKG-06

## 8. สิ่งที่ยังไม่ทำ
- Q-01: “คิวที่ยังไม่ได้ใช้” หมายถึงสถานะใดบ้าง (เช่น ยังไม่ได้เข้ารับบริการ / ยังไม่ยืนยัน / เรียกคืนหรือไม่) -> ส่วนที่เกี่ยวข้องกับข้อนี้จะยังไม่สร้างจนกว่าจะได้คำตอบ

