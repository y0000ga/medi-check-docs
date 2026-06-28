# Medi Check 技術規格書
Technical Specification Document｜版本 1.1｜2025

---

## 1. 系統架構

### 1.1 技術選型

| 層級 | 技術 |
|------|------|
| 後端框架 | FastAPI（Python） |
| ORM / 資料庫 | SQLAlchemy + Alembic（PostgreSQL） |
| 前端框架 | Vue.js（WebView 嵌入行動端） |
| Auth 機制 | JWT Access Token + Refresh Token Rotation |
| Schema 驗證 | Pydantic |
| Migration 工具 | Alembic |

### 1.2 後端分層架構

| 層級 | 職責 |
|------|------|
| Routes（app/api/routes） | 接收 HTTP request，解析 path / query / body，回傳統一格式 response |
| Services（app/services） | 處理商業邏輯、權限驗證、交易流程、組裝 response DTO |
| Repositories（app/repositories） | 執行 SQLAlchemy 查詢與資料存取，不含商業邏輯 |
| Models（app/models） | 定義資料表模型 |
| Schemas（app/schemas） | 定義 request / response schema（Pydantic） |
| Dependencies（app/dependencies） | FastAPI dependency injection |
| Core（app/core） | 共用工具、例外處理、安全邏輯、validator |
| Validation（app/validation） | 欄位驗證規則定義與執行邏輯 |

### 1.3 後端資料流

1. Route 層接收請求，解析 path、query、body
2. Route 層組裝 payload schema
3. Service 層進行權限驗證與商業邏輯處理
4. Repository 層執行查詢或寫入
5. Service 層組裝 response DTO
6. Route 層透過 `success_response(...)` 包成統一格式回傳

### 1.4 Response 格式

所有成功回應使用 `ApiResponse[T]` 包裝：

```json
{
  "success": true,
  "error": null,
  "data": { ... }
}
```

錯誤回應統一格式：

```json
{
  "success": false,
  "error": {
    "statusCode": 400,
    "message": "...",
    "details": [
      { "field": "email", "message": "...", "type": "..." }
    ]
  },
  "data": null
}
```

Exception handlers（`app/core/exception_handlers.py`）統一處理以下例外：
- `AppException`：自定義例外，直接對應 status code 與 message
- `IntegrityError`：資料庫 constraint 違反（400）
- `RequestValidationError`：Pydantic schema 驗證失敗（422），附帶 field-level details
- `HTTPException`：FastAPI 原生例外
- `Exception`：未預期例外（500）

### 1.5 專案目錄結構

```
app/
  api/routes/         # HTTP 路由
  core/               # 安全、例外處理、共用工具
    enums/            # 系統列舉值（UserStatus、HistoryStatus 等）
  db/                 # SQLAlchemy Base / Session
  dependencies/       # FastAPI dependency injection
  models/             # 資料表模型
  repositories/       # DB 查詢
  schemas/            # Pydantic schema
  services/           # 商業邏輯
    jobs/             # 背景工作邏輯
    validators/       # Service 層業務驗證
    errors/           # 各 domain 的 AppException 工廠函式
  validation/         # 欄位驗證規則（rules.py）與執行器（validators.py）
migrations/           # Alembic migration
scripts/              # 輔助腳本
```

---

## 2. 資料庫設計

### 2.1 設計核心

資料模型圍繞三個核心關注點：

- 身份與登入 Session 管理
- 病患與照護關係
- 用藥排程與實際服藥紀錄

### 2.2 最重要的設計分離：Schedule 與 History

這是整個系統最關鍵的資料設計概念。

**Schedule（規則層）** 描述「藥物理論上應該何時被服用」，回答：
- 這顆藥何時該吃
- 這個規則如何重複
- 這個規則何時終止

**History（結果層）** 記錄「某次具體事件實際上有沒有發生」，回答：
- 這次事件到底有沒有吃
- 有沒有被確認為 taken 或 missed
- 當時的上下文是什麼（時間、劑量、備註）

**為什麼不直接預先展開所有 event：**
- 未來事件數量會快速膨脹，尤其是長期、每日、多時段的排程
- 規則修改時必須重新生成、刪除或同步大量未來 event
- 難以分辨「規則改了」和「某一次事件真的被確認處理」
- 還沒發生的 event 本質上只是推導結果，不值得先落地成正式資料

因此，系統選擇在查詢時由 Schedule 動態展開 event instance，只有實際發生且被確認後，才由 History 永久保存。

### 2.3 資料表說明

#### users

系統可登入、可被驗證、可持有 session 的主體。解決「誰可以登入」、「誰擁有帳號身份」、「誰可以發起或接受照護邀請」。

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID PK | |
| name | String(100) | 不可為空 |
| email | String(255) | 唯一、有 index，儲存時一律 lowercase |
| password_hash | String(255) | bcrypt 雜湊值 |
| avatar_url | String(500) | 可為空 |
| birth_date | DateTime(tz) | 可為空 |
| is_email_verified | Boolean | 預設 false，Email 認證功能待實作 |
| status | Enum(UserStatus) | active / invited / disabled，預設 active |
| created_at / updated_at | DateTime(tz) | |

`UserStatus.INVITED` 表示透過邀請機制建立但尚未完整啟用的帳號，此狀態的使用者無法登入。

#### user_sessions

管理 refresh token 的生命週期。解決 token 輪替、登出後撤銷 session、追蹤 session 是否失效、支援多裝置登入管理。

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID PK | |
| user_id | UUID FK(users.id) | |
| token_id | UUID | 唯一，對應 JWT payload 的 `tid` claim |
| refresh_token_hash | String(255) | bcrypt 雜湊值，驗證時需與原始 token 比對 |
| user_agent | String(255) | 可為空，來自 HTTP header |
| ip_address | String(45) | 可為空，支援 IPv6（最長 45 字元） |
| expires_at | DateTime(tz) | refresh token 到期時間 |
| revoked_at | DateTime(tz) | 可為空，登出或 rotation 時填入 |
| created_at / updated_at | DateTime(tz) | |

#### patients

被照護者不一定等於登入使用者。Patient 是系統真正被管理的主體，可以是無帳號的獨立資料（如寵物），也可以對應至另一個 User 帳號。

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID PK | |
| linked_user_id | UUID FK(users.id) | 可為空，唯一。對應至有帳號的使用者；用於判斷自身病患身份 |
| name | String(100) | 不可為空 |
| birth_date | DateTime(tz) | 可為空 |
| avatar_url | String(500) | 可為空 |
| note | String(1000) | 可為空 |
| created_at / updated_at | DateTime(tz) | |

#### care_invitations

照護關係不應被直接建立，需要中間的邀請流程。記錄「誰邀請誰」、「邀請哪位病患」、「要給什麼權限」，並追蹤邀請的狀態流轉。

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID PK | |
| inviter_user_id | UUID FK(users.id) | 發出邀請的使用者 |
| patient_id | UUID FK(patients.id) | 可為空。INVITE_CAREGIVER 時為被照護的 Patient |
| invitee_email | String(255) | 有 index，被邀請人的 email |
| invitee_user_id | UUID FK(users.id) | 可為空，被邀請人若有帳號則填入 |
| invitation_type | Enum(InvitationType) | INVITE_CAREGIVER（我邀請別人來照顧我）/ INVITE_PATIENT（我邀請別人成為我的被照護者） |
| permission_level | Enum(PermissionLevel) | read / write / admin，預設 read |
| status | Enum(InvitationStatus) | pending / accepted / declined / revoked，預設 pending |
| sent_at | DateTime(tz) | 可為空 |
| accepted_at | DateTime(tz) | 可為空 |
| declined_at | DateTime(tz) | 可為空 |
| revoked_at | DateTime(tz) | 可為空 |
| expired_at | DateTime(tz) | 可為空，保留欄位，expiry 功能待實作 |
| created_at / updated_at | DateTime(tz) | |

**InvitationType 說明：**
- `INVITE_CAREGIVER`：發起人（inviter）邀請被邀請人（invitee）來照顧自己的 Patient
- `INVITE_PATIENT`：發起人（inviter）邀請被邀請人（invitee）成為被照護的對象，接受後對方的 Patient 成為被照護者

#### care_relationships

邀請接受後建立的正式照護關係。作為穩定的權限來源，判斷特定 User 對特定 Patient 的存取等級（read / write / admin）。

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID PK | |
| caregiver_user_id | UUID FK(users.id) | 照護者（擁有存取權限的使用者） |
| created_by_user_id | UUID FK(users.id) | 建立此關係的使用者（即邀請的發起人） |
| patient_id | UUID FK(patients.id) | 被照護者 |
| invitation_id | UUID FK(care_invitations.id) | 可為空，來源邀請，可追溯關係建立路徑 |
| permission_level | Enum(PermissionLevel) | read / write / admin，預設 read |
| status | Enum(RelationshipStatus) | active / revoked，預設 active |
| revoked_at | DateTime(tz) | 可為空 |
| created_at / updated_at | DateTime(tz) | |

Unique constraint：`(caregiver_user_id, patient_id)`，每位照護者對每位病患最多一筆有效關係。

> `care_invitations` 解決「關係如何建立」；`care_relationships` 解決「關係建立後如何被使用」。

#### medications

藥物主資料，隸屬於特定 Patient。讓排程與歷史紀錄不必直接掛在病患底下，支援單一病患擁有多種藥物。

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID PK | |
| patient_id | UUID FK(patients.id) | |
| name | String(100) | 不可為空 |
| dosage_form | Enum(DosageForm) | 劑型（tablet / capsule / liquid 等，共 38 種） |
| note | String(500) | 預設空字串 |
| created_at / updated_at | DateTime(tz) | |

#### schedules

描述「理論上應該何時吃藥」的規則，包含頻率、間隔、時段與結束條件。是規則層，不是結果層。

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID PK | |
| patient_id | UUID FK(patients.id) | 反正規化欄位，從 medication 複製以便直接依病患查詢 |
| medication_id | UUID FK(medications.id) | |
| timezone | String(64) | IANA timezone 字串，例如 "Asia/Taipei" |
| start_date | Date | 排程開始日期 |
| time_slots | JSON（list[str]） | 每日時段列表，格式 "HH:MM" |
| amount | Integer | 每次劑量數量，需 > 0 |
| dose_unit | Enum(DosageUnit) | 可為空，劑量單位（mg / tablet / ml 等，共 27 種） |
| frequency_unit | Enum(FrequencyUnit) | 可為空；one-time 時為 null，重複時必填（day / week / month / year） |
| interval | Integer | 可為空；重複間隔，例如 2 表示每 2 天 / 週 |
| weekdays | JSON（list[int]） | 可為空；僅 frequency_unit=week 時使用，值為 1（週一）–7（週日） |
| end_type | Enum(EndType) | 可為空；null 表示 one-time；never / until / counts |
| until_date | Date | 可為空；end_type=until 時的截止日期 |
| occurrence_count | Integer | 可為空；end_type=counts 時的發生次數上限，需 > 1 |
| created_at / updated_at | DateTime(tz) | |

Check constraints：`amount > 0`、`interval >= 1`（或 null）、`occurrence_count >= 1`（或 null）

**Schedule 種類判斷邏輯：**
- `end_type = null`：One-time（單次排程），不允許 frequency_unit / interval / weekdays / until_date / occurrence_count
- `end_type = never`：無限重複，不允許 until_date / occurrence_count
- `end_type = until`：重複至指定日期，until_date 必填且須 > start_date
- `end_type = counts`：重複指定次數，occurrence_count 必填且須 > 1，不允許 until_date

#### histories

每次排程事件的實際結果。保留以下快照欄位，確保歷史紀錄不受後續主資料修改影響：

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | UUID PK | |
| patient_id | UUID FK(patients.id) | 反正規化欄位 |
| schedule_id | UUID FK(schedules.id) | 可為空；手動建立的 history 可能無對應排程 |
| medication_id | UUID FK(medications.id) | 可為空 |
| amount_snapshot | Integer | 建立當下的劑量快照，需 > 0 |
| dose_unit_snapshot | Enum(DosageUnit) | 可為空，建立當下的單位快照 |
| medication_name_snapshot | String(100) | 建立當下的藥品名稱快照 |
| medication_dosage_form_snapshot | Enum(DosageForm) | 可為空，建立當下的劑型快照 |
| scheduled_at_snapshot | DateTime(tz) | 本次事件理論上應發生的時間 |
| intake_at | DateTime(tz) | 可為空；實際服藥時間，quickCheck 時為 now()，手動編輯時更新並將 source 改為 manual |
| status | Enum(HistoryStatus) | pending / taken / missed |
| source | Enum(HistorySource) | quickCheck / manual / system |
| taken_amount | Integer | 可為空；實際服藥量，需 > 0 |
| note | String(500) | 可為空 |
| feeling | Integer | 可為空；1–5 的評分，check constraint 限制範圍 |
| created_at / updated_at | DateTime(tz) | |

`history.source` 標記紀錄來源：
- `quickCheck`：使用者在排程事件上快速確認
- `manual`：使用者補填或修正 intake_at
- `system`：background job 自動補出的 missed history

`history.status` 值：
- `pending`：已建立但尚未確認（目前系統流程中較少主動寫入此狀態）
- `taken`：已服藥
- `missed`：已錯過（由 background job 寫入）

### 2.4 實體關係概覽

| 實體 | 角色定位 |
|------|---------|
| User | 登入與操作主體 |
| UserSession | Session 與 token 管理層 |
| Patient | 被管理與被照護的主體 |
| Medication | 病患底下的藥物主資料 |
| Schedule | 藥物的規則層 |
| History | 規則展開後的結果層 |
| CareInvitation | 照護關係建立前的流程層 |
| CareRelationship | 照護關係建立後的權限層 |

### 2.5 Migration 管理

專案使用 Alembic 管理 schema 變更。

```bash
# 套用所有 migration
uv run alembic upgrade head

# 自動產生新的 migration
uv run alembic revision --autogenerate -m "describe change"
```

Migration 入口：
- `migrations/env.py`
- `alembic.ini`

---

## 3. Auth 流程

### 3.1 Token 模型

| Token | 說明 |
|-------|------|
| access_token | 短效 token，透過 `Authorization: Bearer <token>` 用於一般 API 呼叫 |
| refresh_token | 長效 token，僅在 refresh 與 logout 時透過 request body 傳入；以 bcrypt 雜湊值儲存於 user_sessions |

JWT payload：
- access_token：`{ "sub": "<user_id>", "exp": <timestamp> }`
- refresh_token：`{ "sub": "<user_id>", "tid": "<token_id>", "exp": <timestamp> }`

refresh 時會進行 refresh token rotation：舊 session 標記為 revoked，同時建立新 session 與新 token。

### 3.2 支援端點

- `POST /auth/sign-up`
- `POST /auth/sign-in`
- `POST /auth/refresh`
- `POST /auth/logout`
- `POST /auth/forgot-password`（stub，尚未實作）
- `POST /auth/reset-password`（stub，尚未實作）

### 3.3 註冊流程

1. 正規化 email（lowercase + strip）並驗證不重複
2. 驗證 name（必填，1–100 字）與 password（必填，6–18 字，需含大寫與特殊字元）
3. 建立 `users`（含 bcrypt password_hash）
4. 建立關聯的 `patients`（自身病人，linked_user_id = user.id）
5. 建立 `user_sessions`（含 refresh_token bcrypt hash、user-agent、ip）
6. Response body 回傳 `user_id`、`access_token`、`refresh_token`

重複 email 錯誤分兩層保護：先查詢檢查，再捕捉 `IntegrityError`。

### 3.4 登入流程

1. 正規化 email（lowercase + strip）並依 email 找出使用者
2. 驗證使用者狀態（`validate_active_user`）：
   - 使用者不存在 → 401（token 錯誤，避免 email enumeration）
   - 狀態為 INVITED → 403
   - 狀態為 DISABLED → 403
3. 驗證密碼（bcrypt verify），不符合 → 401 invalid credentials
4. 清除該使用者的所有過期/撤銷 session（`delete_inactive_sessions_by_user_id`）
5. 建立 `user_sessions`
6. Response body 回傳 `user_id`、`access_token`、`refresh_token`

### 3.5 Token 更新流程（Refresh）

1. 解析 refresh token，取出 `tid`（token_id）與 `sub`（user_id）
2. 依 `token_id` 找到對應的 `user_sessions` 資料
3. 依序驗證：
   - session 不存在 → invalid
   - `revoked_at` 不為空 → revoked
   - `expires_at <= now` → expired
   - `user_id` 不符 → invalid
   - bcrypt hash 不符 → invalid
4. 將目前 session 標記為 revoked（`revoked_at = now()`）
5. 建立新的 session 與新的 refresh token（新 token_id）
6. 回傳新的 `user_id`、`access_token`、`refresh_token`

### 3.6 登出流程

1. 解析 refresh token，取出 `tid`
2. 依 `token_id` 找到對應的 `user_sessions`
3. 若 session 不存在、已撤銷或已過期 → 直接回傳成功（避免前端卡住）
4. 若仍有效則標記為 revoked
5. 回傳成功

---

## 4. API 端點總覽

### 4.1 Auth

| Method | 端點 | 說明 |
|--------|------|------|
| POST | /auth/sign-up | 註冊 |
| POST | /auth/sign-in | 登入 |
| POST | /auth/refresh | 更新 token（Refresh Token Rotation） |
| POST | /auth/logout | 登出，撤銷 refresh token session |
| POST | /auth/forgot-password | 忘記密碼（stub，未實作） |
| POST | /auth/reset-password | 重設密碼（stub，未實作） |

### 4.2 Users

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /users/me | 取得目前登入使用者資料（含關聯 patient_id） |
| PATCH | /users/me | 更新目前登入使用者資料 |

### 4.3 Patients

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /patients | 取得所有可存取的病患列表（支援分頁、搜尋、排序） |
| GET | /patients/options | 取得病患選單列表（固定最多 1000 筆，用於前端下拉） |
| POST | /patients | 建立新病患 |
| GET | /patients/{patient_id} | 取得特定病患資料 |
| PUT | /patients/{patient_id} | 更新特定病患資料 |

### 4.4 Medications

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /medications | 取得所有可存取的藥品列表（可依 patient_ids 過濾） |
| GET | /patients/{patient_id}/medications | 取得特定病患的藥品列表 |
| POST | /patients/{patient_id}/medications | 為特定病患新增藥品 |
| GET | /medications/{medication_id} | 取得特定藥品資料 |
| PATCH | /medications/{medication_id} | 更新特定藥品資料 |
| DELETE | /medications/{medication_id} | 刪除特定藥品 |

> **PATCH 限制**：`PATCH /medications/{medication_id}` 僅接受 `name`、`dosage_form`、`note` 三個欄位（均為選填）；`patient_id` 不支援變更，亦不在 request body 中。
> **備註欄位名稱**：medications 相關端點（create / detail / edit）一律使用 `note`（單數），非 `notes`。

### 4.5 Schedules

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /schedules | 取得所有排程（可依 patient_ids 過濾） |
| GET | /schedules/{schedule_id} | 取得特定排程 |
| POST | /medications/{medication_id}/schedules | 為特定藥品建立排程 |
| PATCH | /schedules/{schedule_id} | 更新特定排程（全欄位替換，含規則驗證） |
| DELETE | /schedules/{schedule_id} | 刪除特定排程 |
| GET | /schedule-matches | 取得排程展開後的 event instance（含對應 history） |

> `/schedule-matches` 由後端負責展開排程規則並合併對應 history，前端不需自行實作 recurrence 邏輯。
> 查詢範圍限制：`from_date` 到 `to_date` 最多 **7 天**，超過回傳 400。
> **PATCH 限制**：`PATCH /schedules/{schedule_id}` 為全欄位替換，Required 欄位：`timezone`、`start_date`、`time_slots`、`amount`；Optional 欄位：`dose_unit`、重複規則欄位群。`medication_id` 與 `patient_id` 不在 request body 中，**不支援透過編輯變更所屬藥品或病人**。

### 4.6 Histories

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /histories | 取得服藥歷史列表（可依 patient_ids、medication_id、status、日期範圍過濾） |
| GET | /histories/{history_id} | 取得特定歷史紀錄（包含 note、feeling） |
| POST | /histories/quick-check | 快速確認服藥（quickCheck），source 自動設為 quickCheck |
| PATCH | /histories/{history_id} | 更新歷史紀錄；若修改 intake_at 則自動將 source 改為 manual |

`GET /histories` 回應包含：分頁資料 + `intaken_size`（taken 筆數）+ `missed_size`（missed 筆數）聚合統計。

### 4.7 Care Invitations

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /care-invitations | 取得照護邀請列表（可依 direction、status 過濾） |
| POST | /care-invitations/me/caregiver | 邀請別人成為自己的照護者（INVITE_CAREGIVER） |
| POST | /care-invitations/me/patient | 邀請別人成為自己的被照護者（INVITE_PATIENT） |
| POST | /care-invitations/{invitation_id}/revoke | 撤回邀請（僅邀請發起人可操作） |
| POST | /care-invitations/{invitation_id}/decline | 拒絕邀請（僅被邀請人可操作） |
| POST | /care-invitations/{invitation_id}/accept | 接受邀請（僅被邀請人可操作），接受後自動建立 CareRelationship |

### 4.8 Care Relationships

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /care-relationships | 取得照護關係列表（可依 permission_level、direction 過濾） |

### 4.9 App Config

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /app-config/validation | 取得前端欄位驗證規則的 metadata（供前端同步驗證邏輯） |

### 4.10 Health Check

| Method | 端點 | 說明 |
|--------|------|------|
| GET | /health | 應用程式存活確認（回傳 `{ "status": "ok" }`） |
| GET | /health/ready | 就緒確認，額外 ping DB 並回傳 db 狀態 |

---

## 5. 核心設計決策

### 5.1 權限驗證集中在 Service 層

Route 層只處理 request mapping，真正的權限判斷集中在 Service 層的 validator 函式：

- `validate_patient_access(db, user_id, patient_id) → PatientAccess`：
  - 若 `patient.linked_user_id == user_id`，視為自身病患，給予 WRITE 權限
  - 否則查 care_relationships，取得對應的 permission_level
  - 找不到或無關係 → 403
- `validate_medication_access(db, user_id, medication_id) → MedicationAccess`：
  - 先確認 medication 存在，再透過 `validate_patient_access` 取得 patient 的權限層級
- `ensure_can_read(permission_level)`
- `ensure_can_write(permission_level)`
- `ensure_can_admin(permission_level)`

Permission level 排序：READ(1) < WRITE(2) < ADMIN(3)。

### 5.2 Frontend 不自行實作排程規則

前端透過 `GET /schedule-matches` 取得後端已展開的 event instance，後端同時附上對應的 history 狀態。前端因此不需要自行處理：

- Recurrence 規則計算
- 合法事件判斷
- History merge 邏輯

`schedule-matches` 查詢範圍**最多 7 天**（`MAX_SCHEDULE_EVENT_RANGE_LENGTH_DAYS = 7`），超過限制回傳 400。History 查詢範圍多擴展 ±1 天（`from_date - 1` 到 `to_date + 1`）以確保邊界事件的 history 能被正確抓到。

### 5.3 History 快照設計

`histories` 表不只存外鍵，也保留建立當下的關鍵資訊快照：

- `amount_snapshot`：當時的劑量
- `dose_unit_snapshot`：當時的劑量單位
- `medication_name_snapshot`：當時的藥品名稱
- `medication_dosage_form_snapshot`：當時的劑型
- `scheduled_at_snapshot`：當時應服藥的時間點
- `source`：紀錄產生來源（quickCheck / manual / system）

即使後續主資料被修改，歷史紀錄仍保留當時的真實內容。

### 5.4 Token 傳遞方式

Web 與 mobile 共用相同的 token 合約，透過 response / request body 欄位傳遞，不依賴 cookie。Mobile client 應將 `access_token` 與 `refresh_token` 存放於安全儲存區。

> `app/dependencies/auth.py` 中存有 cookie 版本的輔助函式（`get_refresh_token_from_cookie`、`set_refresh_token_cookie`、`clear_refresh_token_cookie`），目前未被任何路由使用，是預留的 cookie auth 基礎設施。

### 5.5 Background Job：Missed Histories

系統有一個 CLI 背景工作（`app/services/jobs/runner.py`），負責自動補出「錯過但未確認」的 history 紀錄：

- 執行指令：`uv run python -m app missed-histories`
- 邏輯：對所有 active 排程，找出指定時間範圍內應發生但尚無 history 的事件，寫入 `status=missed`、`source=system`
- **Grace period（寬限期）**：預設 **2 小時**。`to_datetime - 2h` 才是實際比對的截止點，確保使用者還有 2 小時確認後不會被提前標記 missed
- 觸發方式：目前為手動或外部 cron 觸發，不是 FastAPI 內部的定時任務

### 5.6 Schedule 業務規則驗證

`app/services/validators/schedule.py` 中有完整的排程規則互斥驗證，在新增與修改排程時均執行（`validate_create_schedule_rules`）：

| 排程種類 | end_type | 必填 | 禁填 |
|----------|----------|------|------|
| One-time | null | time_slots、start_date | frequency_unit、interval、weekdays、until_date、occurrence_count |
| 每日/週/月/年重複，無限 | never | time_slots、frequency_unit、interval、start_date | until_date、occurrence_count |
| 重複至指定日期 | until | 同 never + until_date（須 > start_date） | occurrence_count |
| 重複指定次數 | counts | 同 never + occurrence_count（須 > 1） | until_date |

`frequency_unit=week` 時 `weekdays` 必填（1–7，不可重複）；其他 frequency_unit 禁填 weekdays。

### 5.7 欄位驗證規則集中管理

`app/validation/rules.py` 定義全域欄位規則（`PASSWORD_RULE`、`NAME_RULE`、`EMAIL_RULE` 等），同時供：
- Pydantic schema 使用（max_length 等靜態屬性）
- Service 層 `validate_required_string_field` / `validate_optional_string_field` 使用（runtime 驗證）
- `GET /app-config/validation` 輸出至前端（同步驗證規則）

Password 規則：6–18 字，必須包含大寫字母與特殊字元（`!@#$%^&*`），不允許空白字元。

### 5.8 Schedule 展開時的時區處理

Schedule 儲存 IANA timezone 字串（如 `"Asia/Taipei"`）。展開排程事件時：
- `time_slots` 中的時間（`"HH:MM"`）視為 schedule.timezone 下的當地時間
- 所有 scheduled_at 最終轉換為 UTC 後儲存於 history.scheduled_at_snapshot
- `GET /schedule-matches` 中，history 查找使用 UTC 時間做精確比對

Quick-check 時，client 傳入的 `scheduled_at` 會先轉換為排程時區後，才與 schedule 規則做 occurrence 驗證。

### 5.9 Patient denormalization on Schedule / History

`schedules` 與 `histories` 兩張表都直接儲存 `patient_id`（FK to patients），即使理論上可透過 medication → patient 推導。這是刻意的反正規化設計，讓「依病患查詢排程 / 歷史」的 query 不需要 JOIN medications 表。
