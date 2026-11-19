# สัญญา API สำหรับเว็บและระบบหลังบ้าน
ฉบับ 0.1 · REST JSON ที่เสนอ · base path /v1 บน Edge Function · ทุกเส้นทางต้องส่ง Authorization: Bearer <access_token>

## รูปแบบร่วม
UUID สำหรับ id, YYYY-MM-DD สำหรับวันที่, ISO 8601 พร้อม offset สำหรับเวลา endAt เป็นปลายช่วงแบบไม่รวม จุดอ่านแบบรายการใช้ limit (ค่าเริ่ม 25 สูงสุด 100) และ cursor ผลตอบกลับ {items,nextCursor} คำสั่งสร้างหรือทำธุรกรรม POST รับ Idempotency-Key ต่อผู้ใช้และ operation คีย์เดิมกับข้อมูลเดิมคืนผลเดิม คีย์เดิมกับข้อมูลต่างคืน 409 คำสั่งแก้สถานะส่ง revision เพื่อป้องกันเขียนทับ

| Method และ path | ผู้ใช้และข้อมูลเข้า | ผลเมื่อสำเร็จ |
|---|---|---|
| GET /me | ผู้มีบัญชี active | 200 employeeId, roles, team, permissions |
| GET /me/balances?asOf= | เจ้าของ | 200 รายประเภทและรอบ พร้อม unit, allocated, used, pending, remaining, projected |
| POST /leave/preview | เจ้าของ ส่ง typeId, intervals[] | 200 segments[], quantity, unit, projected, warnings[] |
| POST /leave/requests | เจ้าของ ส่งข้อมูลแบบ preview และ reason | 201 id, status, revision, quantity ตรวจสูตรซ้ำบน server |
| GET /leave/requests?scope=mine หรือ inbox | เจ้าของหรือผู้อนุมัติปัจจุบัน | 200 รายการตามสิทธิ์และสถานะที่กรอง |
| GET /leave/requests/{id} | เจ้าของ ผู้ตัดสินที่เกี่ยวข้อง หรือ HR ตามสิทธิ์ | 200 รายละเอียด ช่วงลา ผลตัดสิน และประวัติที่เปิดเผยได้ |
| POST /leave/requests/{id}/decision | ผู้อนุมัติปัจจุบัน ส่ง action, revision, comment | 200 สถานะใหม่และ revision ต้องให้เหตุผลเมื่อ reject |
| POST /leave/requests/{id}/withdraw | เจ้าของ ส่ง revision | 200 WITHDRAWN |
| POST /leave/requests/{id}/reverse | HR ที่ได้รับสิทธิ์แก้ย้อนหลัง ส่ง revision, reason | 200 REVERSED และเลขอ้างอิงคืนสิทธิ์ |
| POST /attendance | HR หรือหัวหน้าตามทีม ส่ง employeeId, workDate, kind และช่วงขาดหรือเวลามาสาย | 201 id, revision มีผลกับรายงานทันที |
| GET /attendance | HR หรือหัวหน้าตามทีม ส่ง from, to, teamId หรือ employeeId | 200 รายการในขอบเขต |
| PATCH /attendance/{id} | ผู้มีสิทธิ์แก้ ส่ง revision, changes, reason | 200 ข้อมูลล่าสุดพร้อม audit |
| GET /reports/executive | CEO ส่ง from, to, teamId?, employeeId?, typeId? | 200 leaveDays, absentDays, lateCount, lateMinutes, pendingCount, asOf และ breakdown |
| GET, POST /admin/employees | Admin อ่านหรือสร้างด้วย email, teamId, roles, primaryApproverId | 200 รายการ หรือ 201 บัญชีใหม่ |
| PATCH /admin/employees/{id} | Admin ส่ง revision กับ status, teamId หรือ roles | 200 ค่าปัจจุบัน |
| PUT /admin/routes/{employeeId} | Admin ส่ง primaryId, effectiveFrom | 200 เส้นทางพร้อมวันที่มีผล |
| GET, POST /admin/delegations | Admin อ่านหรือสร้างด้วย primaryId, delegateId, startsOn, endsOn | 200 รายการ หรือ 201 การมอบหมาย |
| PATCH /admin/delegations/{id} | Admin ส่ง revision, endsOn หรือสถานะยกเลิก | 200 ช่วงที่มีผล ใช้กับงานค้างทันที |
| POST /admin/policy-versions | Admin ส่ง effectiveFrom, rules, HR approval reference | 201 ฉบับใหม่ ไม่เขียนทับฉบับเก่า |
| PUT /admin/calendar/{employeeId}/{date} | Admin ส่ง shifts, breaks, reason | 200 รุ่นปฏิทิน |
| POST /admin/balance-entries | Admin ส่ง periodId, kind, signedQuantity, reason, sourceId | 201 รายการจัดสรรหรือปรับยอดพร้อม audit |

## ตัวอย่างการส่งลา
POST /leave/requests โดยระบบหาผู้ยื่นจาก token ข้อมูลตัวอย่างเป็นสมมติฐานสำหรับทดสอบ
```json
{"typeId":"f112daea-7d13-44fa-b355-4340ff903e81","intervals":[{"startAt":"2026-12-01T09:00:00+07:00","endAt":"2026-12-01T12:00:00+07:00"}],"reason":"ธุระส่วนตัว"}
```
ผล 201 ตัวอย่าง
```json
{"id":"69494cde-c8cc-4010-a768-1bdf1f305625","status":"PENDING","revision":1,"quantity":180,"unit":"minute"}
```
POST /leave/requests/{id}/decision รับ {"action":"approve","revision":1} และคืน {"status":"APPROVED","revision":2} เมื่อ commit สำเร็จ

## ข้อผิดพลาดที่หน้าเว็บต้องแยก
รูปแบบ {"error":{"code":"STALE_REVISION","message":"ข้อมูลเปลี่ยนแล้ว กรุณาโหลดใหม่","requestId":"…"}} ใช้ 400 เมื่อรูปแบบผิด, 401 เมื่อ session ใช้ไม่ได้, 403 เมื่อบัญชีปิดหรือหน้าที่ไม่อนุญาต, 404 เมื่อหาไม่พบหรือผู้ใช้อยู่นอกขอบเขตข้อมูล, 409 สำหรับ STALE_REVISION, OVERLAP หรือ ALREADY_DECIDED, 422 สำหรับ INSUFFICIENT_BALANCE, POLICY_MISSING หรือ NO_APPROVER เมื่อพบ 409 ให้โหลดสถานะล่าสุดก่อนแสดงทางเลือก เมื่อเครือข่ายขาดหลังส่ง ใช้คีย์เดิมสอบทานผล
