# แบบจำลองข้อมูลและพจนานุกรมข้อมูล
ฉบับ 0.1 · ใช้ UUID เป็นรหัสธุรกิจ ใช้ timestamptz สำหรับเหตุการณ์ และ date สำหรับวันทำงานใน Asia/Bangkok

| ตาราง | ฟิลด์หลัก | เงื่อนไขและความสัมพันธ์ |
|---|---|---|
| employee | id, email_normalized, auth_subject, status, hire_date | email และ subject ไม่ซ้ำ subject ว่างได้ก่อนเข้าใช้ครั้งแรก |
| team / membership | team_id, employee_id, valid_from, valid_to | ห้ามช่วงทีมหลักซ้อนกัน เก็บทีม ณ วันทำงานสำหรับรายงานย้อนหลัง |
| employee_role | employee_id, role | คีย์ผสม บทบาทประกอบกันได้ |
| approval_route | employee_id, primary_id, valid_from, valid_to | มีผู้อนุมัติหลักคนเดียวต่อช่วง ห้ามชี้ตัวเอง |
| delegation | primary_id, delegate_id, starts_on, ends_on | ผู้แทน active ช่วงทดแทนของหัวหน้าคนเดียวกันห้ามซ้อน |
| policy_version / leave_type | version_id, type_id, effective_from, rules, approved_by | กติกาที่ประกาศใช้แล้วเป็นฉบับอ่านอย่างเดียว |
| work_day | employee_id, local_date, shifts, break_intervals, calendar_version | ระบุช่วงงานจริงและวันหยุด เก็บรุ่นเมื่อคำนวณ |
| entitlement_period | id, employee_id, type_id, starts_on, ends_on, unit | หนึ่งบัญชีต่อคน ประเภท และรอบ |
| leave_request | id, employee_id, type_id, status, reason_private, revision, created_at | revision เพิ่มเมื่อเปลี่ยนสถานะ เก็บเหตุผลแยกจากข้อมูลรายงาน |
| leave_segment | request_id, date, starts_at, ends_at, minutes, day_fraction, period_id, policy_version, team_id | หนึ่งคำขอมีหลายช่วง ตารางเบิกใช้จริงอ้างช่วงเหล่านี้ |
| balance_entry | id, period_id, segment_id, kind, signed_quantity, source_id, reversal_of | รายการจัดสรร ยกมา ใช้ คืน ปรับ และหมดอายุ คุมรายการจากต้นทางไม่ให้ซ้ำ |
| attendance_event | id, employee_id, work_date, kind, start_at, end_at, scheduled_at, actual_at, revision, team_id | แบบจำลอง MVP หนึ่งรายการต่อคน วัน และชนิด รองรับหลายช่วงด้วยรายละเอียดลูก หากมีหลายกะให้ HR ยืนยันหน่วยนับ |
| attendance_adjustment | event_id, request_id, delta_minutes, reason, actor_id | เชื่อมหลักฐานแก้ขาดเป็นลาและตรวจย้อนผลสุทธิ |
| audit_event / idempotency | actor_id, action, entity_id, time, request_id / actor_id, operation, key, payload_hash, result_id | audit เพิ่มอย่างเดียว คีย์ซ้ำและ payload ตรงกันให้คืนผลเดิม |

## การป้องกันข้อมูลขัดกัน
อนุมัติหนึ่งครั้งต้องล็อกคำขอและบัญชีสิทธิ์ที่เกี่ยวตามลำดับเดียวกัน ตรวจ revision สถานะ เส้นทาง และยอด สร้างรายการตัดสิทธิ์ เปลี่ยนสถานะ และบันทึก audit ภายในธุรกรรมเดียว เมื่อมีการส่งคำขอพร้อมกันให้ล็อกพนักงานหรือใช้ข้อจำกัดช่วงเวลาเพื่อกันช่วงลา PENDING/APPROVED ที่ซ้อนกัน หากธุรกรรมล้มเหลวให้ย้อนทั้งหมด

## ดัชนีและรายงาน
สร้างดัชนีตาม employee_id + work_date, status + employee_id และ period_id ตรวจแผน query ด้วยข้อมูลจำลอง รายงานลาตามวันใน leave_segment และสถานะอนุมัติ ไม่รวมทั้งคำขอซ้ำเมื่อเลือกช่วงเวลาบางส่วน นับสายจากเหตุการณ์ที่ยังมีผล ยอดขาดรวมสัดส่วนวันหลังหักการปรับ ไม่รวมคนซ้ำในจำนวนพนักงานที่ขาด ยอดสิทธิ์คงเหลือเป็นค่าตามรอบ ณ วันที่เลือก ไม่บวก snapshot รายวันเข้าด้วยกัน
