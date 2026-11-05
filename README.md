# Kafeology CRM System
 
ระบบสะสมแต้มและสิทธิพิเศษสำหรับร้านกาแฟ Kafeology (หลายสาขา) ลูกค้าสมัครสมาชิกและใช้งานผ่านแอป LINE (LIFF) พนักงานหน้าร้านสะสมแต้มและสแกน QR แลกรางวัล ส่วนผู้ดูแลจัดการกติกา ระดับสมาชิก ของรางวัล แคมเปญ และแบนเนอร์ผ่าน Admin Console
 
> สรุปจาก FigJam: **Kafeology System Design** (data design และ service domain)
 
---
 
## 1. ระบบทำอะไรได้บ้าง
 
| ผู้ใช้ | ช่องทาง | ทำอะไร |
|---|---|---|
| **ลูกค้า (Member)** | แอป LINE (LIFF) | สมัครสมาชิกและผูกบัญชี LINE, ดูแต้มและวันหมดอายุ, ดูระดับสมาชิก, ดูรางวัลและสร้าง QR แลกรางวัล, ดูแบนเนอร์และแคมเปญ |
| **พนักงานหน้าร้าน (Staff)** | Staff Web (POS) | สะสมแต้มจากบิล, แนบไฟล์บิล, สแกน QR แลกรางวัล, ขอแก้ไขรายการแต้มที่ผิด |
| **ผู้จัดการสาขา (Branch Manager)** | Admin Web | ดูสมาชิกและรายการของสาขา, อนุมัติคำขอแก้ไขแต้มและคำขอสร้างระดับ |
| **ผู้ดูแลระบบ (Super Admin)** | Admin Web | ตั้งค่ากติกา, จัดการระดับ ของรางวัล แคมเปญ แบนเนอร์, จัดการพนักงาน สาขา บทบาท และสิทธิ์เมนู |
 
### ความสามารถหลัก
- **สะสมแต้ม:** ยอดบิล ÷ บาทต่อแต้ม × ตัวคูณของระดับ (+ โบนัสจากแคมเปญ)
- **แต้มหมดอายุทีละก้อน:** แต้มแต่ละครั้งที่ได้รับมีวันหมดอายุของตัวเอง ตัดแต้มแบบ FIFO
- **ระดับสมาชิก (Tier):** ขึ้นระดับเมื่อยอดสะสมในรอบถึงเกณฑ์ ถือระดับตามจำนวนเดือนที่กำหนด สร้างระดับใหม่ต้องผ่านการอนุมัติ
- **ของรางวัล:** กำหนดแต้มที่ใช้ โควตา ช่วงเวลา และระดับที่แลกได้ แลกด้วย QR ที่มีอายุเป็นนาที
- **แคมเปญแต้ม:** ตัวคูณแต้ม, แต้มโบนัสต่อบิล, ชวนเพื่อนสมัคร
- **แบนเนอร์:** ภาพสไลด์หน้าแรกของแอป LINE (2 ภาษา) ผูกกับรางวัลหรือแคมเปญได้
- **อนุมัติ 2 ชั้น:** คำขอแก้ไขแต้มและคำขอสร้างระดับ ผู้ขออนุมัติเองไม่ได้
- **ประวัติการแก้ไขทุกส่วน:** กติกา ระดับ รางวัล แคมเปญ แบนเนอร์ ผู้ใช้ และรายการแต้ม
---
 
## 2. Architecture
 
```
                 ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
                 │ Gateway-Admin-Web    │ │ Gateway-Staff-Web    │ │ Gateway-LIFF         │
                 │ ตรวจและออก token      │ │ ตรวจและออก token      │ │ ตรวจ LINE token       │
                 │ ฝั่ง admin            │ │ ฝั่ง staff            │ │ และออก token member  │
                 └──────────┬───────────┘ └──────────┬───────────┘ └──────────┬───────────┘
                            │                        │                        │
     ┌──────────────────────┼────────────────────────┼────────────────────────┤
     │                      │                        │                        │
┌────▼──────────┐  ┌────────▼──────┐  ┌──────────────▼───┐  ┌─────────────────▼───┐  ┌──────────────────┐
│ Identity      │  │ UserService   │  │ MemberService    │  │ CRM Service         │  │ Notification     │
│ Service       │  │               │  │                  │  │                     │  │ Service          │
└───────────────┘  └───────────────┘  └──────────────────┘  └─────────────────────┘  └──────────────────┘
```
 
| Service | หน้าที่ | Domain |
|---|---|---|
| **Gateway-Admin-Web-Service** | BFF ของ Admin Console ตรวจและออก token ฝั่ง admin | ไม่มี |
| **Gateway-Staff-Web-Service** | BFF ของแอปพนักงานหน้าร้าน ตรวจและออก token ฝั่ง staff | ไม่มี |
| **Gateway-LIFF-Service** | BFF ของแอป LINE ตรวจ LINE token และออก token ของสมาชิก | ไม่มี |
| **Identity Service** | credential และการยืนยันตัวตนของพนักงาน | IdentityUser |
| **UserService** | พนักงาน สาขา บทบาท สิทธิ์เมนู | Branch, UserBranch, UserAccount, Role, Menu, Privilege, UserAccountHistory, LoginHistory |
| **MemberService** | สมาชิก บัญชีแต้ม และการผูก LINE | Member, Point, Transaction, TransactionHistory, PointRequest, BillingDocument, MemberLineLink, MemberLoginHistory, LineLinkOtp |
| **CRM Service** | กติกา ระดับ ของรางวัล แคมเปญ แบนเนอร์ | RuleSettings, Tier, Rewards, Campaigns, Banners (+ History ของแต่ละตัว) |
| **Notification Service** | ส่งข้อความหาลูกค้าผ่าน LINE | templates, notifications |
 
---
 
## 3. Data Design แยกตาม Service
 
### 3.1 UserService
 
| ตาราง | หน้าที่ |
|---|---|
| `UserAccountData` | พนักงานและผู้ดูแล: ชื่อ, username, email, role, สถานะ, PIN hash |
| `BranchData` | สาขา: รหัสสาขา (เช่น ARI, SKV39), ชื่อ, สถานะ |
| `UserBranchData` | พนักงาน 1 คนรับผิดชอบได้หลายสาขา พร้อมสาขาเริ่มต้น (`isDefaultBranch`) |
| `RoleData` | บทบาท: `super_admin`, `branch_manager`, `staff` |
| `MenuData` | เมนูใน Admin Console (มีเมนูแม่-ลูกผ่าน `ParentMenuId`) |
| `PrivilegeData` | สิทธิ์ของแต่ละบทบาทต่อแต่ละเมนู (unique คู่ RoleId + MenuId) |
| `UserAccountHistoryData` | ประวัติการแก้ไขผู้ใช้: Create, UpdateProfile, ChangeRole, ChangeBranches, Disable, Enable |
| `LoginHistory` | ประวัติเข้าสู่ระบบของพนักงาน: LoginSuccess, LoginFailure, Logout, PinFailure, Locked |
 
**ความสัมพันธ์**
- `RoleData` 1 : N `UserAccountData` (ผู้ใช้มี 1 บทบาท)
- `UserAccountData` N : N `BranchData` ผ่าน `UserBranchData`
- `RoleData` N : N `MenuData` ผ่าน `PrivilegeData`
- `UserAccountData` 1 : N `UserAccountHistoryData`, `LoginHistory`
### 3.2 MemberService
 
**สมาชิกและการผูก LINE**
 
| ตาราง | หน้าที่ |
|---|---|
| `members` | สมาชิก: รหัสสมาชิก (`KF-010231`), ชื่อ, เบอร์โทร, วันเกิด, ระดับปัจจุบัน, วันที่ขึ้นระดับ, วันหมดอายุระดับ, วันเริ่มรอบสะสม, สถานะ |
| `MemberLineLinkData` | การผูกบัญชี LINE กับสมาชิก พร้อมการยินยอม PDPA (ผูกได้หลายครั้ง active ได้ 1) |
| `LineLinkOtpData` | OTP สำหรับยืนยันการผูกบัญชี LINE |
| `MemberLoginHistoryData` | ประวัติเข้าแอป LINE: LiffLogin, LinkSuccess, LinkFailure, OtpRequested, OtpFailed, Unlink |
 
**บัญชีแต้ม**
 
| ตาราง | หน้าที่ |
|---|---|
| `point_transactions` | สมุดบัญชีแต้ม (ledger) ทุกเหตุการณ์ที่ทำให้แต้มเปลี่ยน พร้อม `balance_before` / `points` / `balance_after`, เลขบิล, ยอดบิล, snapshot ตัวคูณ, พนักงาน, สาขา |
| `point` | แต้มแต่ละก้อน: แต้มที่เหลือ, วันที่ได้, วันหมดอายุ, สถานะ |
| `BillingDocument` | ไฟล์บิลที่แนบกับรายการ (เก็บใน object storage) |
| `point_requests` | คำขอแก้ไขรายการแต้ม (แต้มและยอดบิลที่ควรเป็น) รออนุมัติ |
| `transaction_history` | ประวัติการแก้ไขรายการแต้ม: แต้ม ยอดบิล และยอดคงเหลือก่อน/หลังแก้ |
 
**ความสัมพันธ์**
- `members` 1 : N `point_transactions`, `point`, `MemberLineLinkData`, `LineLinkOtpData`, `MemberLoginHistoryData`
- `point_transactions` 1 : 0..1 `point` (รายการสะสมแต้มสร้างก้อนแต้ม 1 ก้อน)
- `point_transactions` 1 : N `BillingDocument`, `point_requests`, `transaction_history`
**สถานะรายการแต้ม:** `success` (บันทึกสำเร็จ) / `adjusted` (ถูกปรับยอดแล้ว) / `voided` (ถูกยกเลิก)
 
### 3.3 CRM Service
 
**กติกาและระดับสมาชิก (Rules / Tier)**
 
| ตาราง | หน้าที่ |
|---|---|
| `RuleSettings` | กติกาหลัก (มีแถวเดียว): บาทต่อแต้ม, อายุแต้ม (เดือน), อายุ QR (นาที) |
| `RuleSettingsHistoryData` | ประวัติการแก้กติกา |
| `TierData` | ระดับสมาชิกที่ใช้งานจริง (master): ชื่อ, สี, รูป, เกณฑ์ยอดสะสม, ตัวคูณแต้ม, อายุระดับ (เดือน), สถานะ |
| `TierRequestData` | คำขอสร้าง/แก้/ลบระดับ ต้องอนุมัติก่อน อนุมัติแล้วจึงเพิ่มเข้า `TierData` |
| `TierHistoryData` | ประวัติการแก้ไขระดับแต่ละระดับ |
 
**สถานะคำขอระดับ:** `Pending` → `Approved` (รอถึงวันที่กำหนด) → `Applied` / หรือ `Rejected`, `Cancelled`, `Failed`
 
**ของรางวัล (Rewards)**
 
| ตาราง | หน้าที่ |
|---|---|
| `rewards` | ของรางวัล (ชื่อและรายละเอียด 2 ภาษา), แต้มที่ใช้, โควตา, ช่วงเวลาแลก, ระดับที่แลกได้ (`all` / `selected`), สถานะ |
| `reward_tiers` | ระดับที่แลกได้ เมื่อ `eligibility = selected` |
| `media_reward` | รูป cover คูปอง (JPG/PNG, 4:3, ไม่เกิน 2 MB) |
| `redemption_qr` | QR แลกรางวัลที่ลูกค้าสร้าง: `pending` / `used` / `expired` / `cancelled` |
| `reward_history` | ประวัติการแก้ไขรางวัล: create, update, pause, resume, delete |
 
**แคมเปญ (Campaign)**
 
| ตาราง | หน้าที่ |
|---|---|
| `campaigns` | แคมเปญแต้ม: ประเภท (ตัวคูณแต้ม / แต้มโบนัสต่อบิล / ชวนเพื่อน), ตัวคูณ, แต้มโบนัส, ยอดขั้นต่ำ, ช่วงวันที่, วันในสัปดาห์, ช่วงเวลา, สถานะ |
| `tier_campaigns` | ระดับที่ได้สิทธิ์ในแคมเปญ |
| `campaign_history` | ประวัติการแก้ไขแคมเปญ: create, update, pause, resume, cancel |
 
**แบนเนอร์ (Banner)**
 
| ตาราง | หน้าที่ |
|---|---|
| `banners` | แบนเนอร์หน้าแรกแอป LINE: ชื่อและภาพ 2 ภาษา (16:9, ไม่เกิน 5 MB), ประเภท (แสดงผลอย่างเดียว / ผูกรางวัล / ผูกแคมเปญ), ช่วงเวลา, แสดงหรือซ่อน, ลำดับ |
| `banner_history` | ประวัติการแก้ไขแบนเนอร์: create, update, show, hide, reorder, delete |
 
### 3.4 Notification Service
 
| Domain | หน้าที่ |
|---|---|
| templates | แม่แบบข้อความ (ได้แต้ม, ขึ้นระดับ, แต้มใกล้หมดอายุ, ผลการอนุมัติ ฯลฯ) |
| notifications | log การส่งข้อความผ่าน LINE และสถานะการส่ง |
 
---
 
## 4. กฎทางธุรกิจหลัก
 
**แต้ม**
- แต้ม = `floor(ยอดบิล ÷ บาทต่อแต้ม × ตัวคูณของระดับ)` โดยเก็บ snapshot อัตราและตัวคูณไว้ในรายการ
- แต้มแต่ละก้อนหมดอายุ = วันที่ได้ + อายุแต้ม (เดือน)
- ตัดแต้มจากก้อนที่หมดอายุก่อน (FIFO)
- แต้มคงเหลือ = ผลรวมแต้มที่เหลือของก้อนที่ยังไม่หมดอายุ
- รายการแต้มแก้ไขตรง ๆ ไม่ได้ ต้องสร้างรายการใหม่มาหักล้าง (`adjust` / `reversal`) และบันทึก `transaction_history`
**ระดับสมาชิก**
- ขึ้นระดับทันทีเมื่อยอดสะสมในรอบถึงเกณฑ์ของระดับที่สูงกว่า
- ถือระดับตาม `Tier_term_months` นับจากวันที่ขึ้นระดับ ครบกำหนดแล้วกลับระดับเริ่มต้นและเริ่มนับรอบใหม่
- สร้างระดับใหม่ต้องให้ Manager อนุมัติ เลือกได้ว่ามีผลหลังอนุมัติหรือตามวันที่กำหนด
**ของรางวัลและ QR**
- แลกได้เมื่อ: รางวัลเปิดอยู่, อยู่ในช่วงเวลา, โควตาเหลือ, ระดับตรงตามที่กำหนด, แต้มพอ และ QR ยังไม่หมดอายุ
- ตัดแต้มและเพิ่มโควตาที่ใช้ใน transaction เดียวกัน
**แคมเปญ**
- บิลที่เข้าได้หลายแคมเปญ ใช้แคมเปญที่ให้แต้มสูงสุดเพียงอันเดียว
- แต้มจากแคมเปญไม่นับเป็นยอดใช้จ่ายสำหรับระดับ
**การอนุมัติ**
- ผู้ขออนุมัติคำขอของตัวเองไม่ได้ (ใช้กับคำขอแก้ไขแต้มและคำขอระดับ)
- ปฏิเสธต้องระบุเหตุผล
**ข้อมูลและความปลอดภัย**
- ลบข้อมูลแบบ soft delete และ anonymize ข้อมูลส่วนตัวของสมาชิกตาม PDPA เมื่อลบบัญชี
- ตารางประวัติเพิ่มได้อย่างเดียว ห้ามแก้หรือลบ
- ไม่เก็บรหัสผ่าน PIN และ OTP เป็นข้อความจริง เก็บเป็น hash เท่านั้น
- เวลาเก็บเป็น UTC และแสดงผลเป็นเวลาไทย (พ.ศ.)
---
 
## 5. การทำงานข้าม Service ที่ต้องระวัง
 
- **สะสมแต้ม (MemberService) ต้องใช้ข้อมูลจาก CRM Service:** อัตราแต้ม, ตัวคูณของระดับ และแคมเปญที่ใช้ได้ ควร cache กติกาไว้ใน MemberService หรือเรียกผ่าน API แล้วเก็บ snapshot ลงในรายการ
- **แลกรางวัล:** ตัดแต้ม (MemberService) กับเพิ่มโควตาและปิด QR (CRM Service) อยู่คนละ service ต้องออกแบบ flow ให้ถ้าขั้นใดล้มเหลว อีกขั้นถูกยกเลิกด้วย (เช่น saga หรือ reserve → confirm)
- **ส่ง event ไป Notification และประวัติ:** ใช้ message broker + outbox pattern เพื่อไม่ให้ event หายเมื่อ service ปลายทางล่ม
- **อ้างอิงข้าม service:** เช่น `user_id`, `branch_id` ใน `point_transactions` ไม่มี FK จริง ควรเก็บ snapshot ชื่อไว้ด้วย
---
 
## 6. สิ่งที่ยังต้องเก็บใน Design
 
| จุด | รายละเอียด |
|---|---|
| ตารางตัดแต้มจากก้อน | ยังไม่มีตาราง `lot_consumptions` (รายการไหนตัดแต้มจากก้อนไหน) ทำให้ยกเลิกการแลกแล้วคืนแต้มเข้าก้อนเดิมไม่ได้ |
| `point_transactions` | ยังไม่มี `ref_transaction_id` สำหรับรายการ adjust / reversal และค่า `type` ยังไม่ครบ (earn, redeem, adjust, expire, reversal, forfeit) |
| `point` | ยังไม่มี `points_earned` และสถานะยังไม่ครบ (active, depleted, expired, voided) |
| `members` | `line_user_id` และ `line_display_name` ซ้ำกับ `MemberLineLinkData` และ `tier_id` type ยังไม่ถูก |
| `rewards` | มี `tier_id` ซ้ำกับ `reward_tiers`, field `start_at` ซ้ำ 2 แถว (แถวที่สองควรเป็น `end_at`) และมีตาราง `rewards` ซ้ำ 2 ตาราง |
| `TierData` / `TierRequestData` | `Tier_term_months` เป็น string ควรเป็น int และควรตัดสินใจว่าอายุระดับเก็บที่ระดับแต่ละระดับ หรือที่ `RuleSettings` ที่เดียว |
| Type ของวันที่ | หลายตารางใช้ `date` ควรเป็น `timestamptz` / `DateTime` |
| ชื่อ field | ปน snake_case กับ PascalCase ควรเลือกแบบเดียวทั้งระบบ |
| Mandatory | หลายตารางยังไม่ได้ระบุ |
| Identity Service | ยังไม่ได้ระบุ field ของ IdentityUser และ `PinHash` ยังอยู่ใน `UserAccountData` |
| Notification Service | ยังไม่ได้ออกแบบตาราง templates และ notifications |
 
