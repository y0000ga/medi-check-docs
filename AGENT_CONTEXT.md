# Medi Check — Agent Context

這份文件提供給 code agent 開發參考用，說明專案架構、檔案結構、命名慣例與重要設計決策。

---

## 1. 專案概覽

**產品名稱：** Medi Check
**類型：** 藥品提醒與服藥管理 Web App，透過 WebView 嵌入行動端
**前端框架：** Nuxt 3 + Vue 3 + TypeScript
**UI Framework：** Nuxt UI（基於 Tailwind CSS）
**設計系統：** Healing Touch（詳見 shared-docs/DESIGN.md）
**後端：** FastAPI（Python），不在此專案內

---

## 2. 目錄結構

```
├── app.config.ts          # App 設定
├── app.vue                # 根元件
├── assets/
│   └── css/
│       └── tailwind.css  # Tailwind 全域樣式
├── components/
│   ├── AppFooter.vue      # 全域 Footer
│   ├── AppHeader.vue      # 全域 Header（手機版漢堡選單）
│   └── auth/
│       └── AuthShell.vue  # 登入/註冊頁面的外框元件
├── constants/
│   └── route.ts           # 路由常數定義
├── layouts/
│   ├── auth.vue           # 登入/註冊用 layout（無側欄）
│   └── default.vue        # 一般頁面 layout（含側欄）
├── pages/                 # 頁面元件（自動路由）
├── utils/
│   ├── registerValidation.ts  # 註冊表單驗證
│   └── validation.ts          # 通用表單驗證
├── nuxt.config.ts
├── tailwind.config.js
└── DESIGN.md              # 設計系統規範
```

---

## 3. 頁面路由對應

| 路由 | 檔案 | 說明 |
|------|------|------|
| `/` | `pages/index.vue` | 首頁（今日服藥事件） |
| `/login` | `pages/login.vue` | 登入頁 |
| `/register` | `pages/register.vue` | 註冊頁 |
| `/main` | `pages/main.vue` | 主頁（待確認用途） |
| `/medications` | `pages/medications/index.vue` | 藥品列表 |
| `/medications/new` | `pages/medications/new.vue` | 新增藥品（Step Flow） |
| `/medications/[id]` | `pages/medications/[id].vue` | 藥品詳細 |
| `/medications/[id]/edit` | `pages/medications/[id]/edit.vue` | 編輯藥品 |
| `/schedules` | `pages/schedules/index.vue` | 排程列表 |
| `/schedules/new` | `pages/schedules/new.vue` | 新增排程（Step Flow） |
| `/schedules/[id]` | `pages/schedules/[id].vue` | 排程詳細 |
| `/schedules/[id]/edit` | `pages/schedules/[id]/edit.vue` | 編輯排程 |
| `/histories` | `pages/histories/index.vue` | 服藥紀錄列表 |
| `/histories/[id]` | `pages/histories/[id].vue` | 服藥紀錄詳細 |
| `/histories/[id]/edit` | `pages/histories/[id]/edit.vue` | 編輯服藥紀錄 |
| `/care-relationships` | `pages/care-relationships/index.vue` | 照護關係列表 + 邀請管理（雙 Tab） |
| `/care-invitations` | `pages/care-invitations/index.vue` | 重定向至 /care-relationships?tab=invitations |
| `/care-invitations/new` | `pages/care-invitations/new.vue` | 重定向至 /care-relationships?tab=invitations&modal=new |
| `/settings` | `pages/settings/index.vue` | 設定主頁 |
| `/settings/notifications` | `pages/settings/notifications.vue` | 通知設定 |
| `/settings/profile/edit` | `pages/settings/profile/edit.vue` | 編輯個人資料 |
| `/health` | `pages/health.vue` | 健康檢查端點 |

---

## 4. Layout 說明

### `layouts/auth.vue`
用於登入、註冊頁面，無側欄、無 Header，只有置中的內容區。

使用方式（在 page 裡宣告）：
```vue
<script setup>
definePageMeta({ layout: 'auth' })
</script>
```

### `layouts/default.vue`
用於所有登入後的頁面，包含：
- 桌機版：左側固定側欄（240px）+ 右側內容區
- 平板版：左側 icon 窄側欄（64px）+ 右側內容區
- 手機版：頂部 Header（含漢堡選單）+ 內容區 + 左側滑出選單

RWD Breakpoints：
- 手機：`< 768px`
- 平板：`768px – 1024px`
- 桌機：`>= 1024px`

---

## 5. 導覽結構

側欄導覽項目（依序）：
- 首頁 → `/`
- 藥品 → `/medications`
- 排程 → `/schedules`
- 照護關係 → `/care-relationships`
- 設定 → `/settings`

---

## 6. 命名慣例

### 元件（Components）
- 檔名：PascalCase（`AppHeader.vue`、`AuthShell.vue`）
- 業務元件放 `components/` 下對應功能子目錄
- 共用元件放 `components/` 根目錄

### 頁面（Pages）
- 檔名：kebab-case 或 index.vue
- 動態路由用 `[id].vue`

### 常數（Constants）
- 路由常數統一定義在 `constants/route.ts`
- 使用時從 constants import，不要在頁面裡 hardcode 路由字串

### 工具函式（Utils）
- 檔名：camelCase（`registerValidation.ts`）
- 表單驗證邏輯放 `utils/`，不要放在元件裡

---

## 7. UI Framework 使用規範

使用 **Nuxt UI**，以下元件直接使用，不需要自己刻：
- `UButton`、`UInput`、`USelect`、`UTextarea`
- `UModal`、`UDropdown`
- `UToggle`、`UCheckbox`、`URadio`
- `UBadge`、`UAvatar`
- `UPagination`、`UTabs`
- `UCard`、`UDivider`

需要自己建立的業務元件（放在 `components/` 下）：
- `EventCard`（首頁服藥事件卡片，三種狀態）
- `MedicationCard`（藥品列表卡片）
- `ScheduleCard`（排程列表卡片）
- `HistoryCard`（服藥紀錄卡片）
- `InvitationCard`（邀請卡片）
- `CareRelationshipCard`（照護關係卡片）
- `EmptyState`（空白狀態：插圖 + 文字 + 按鈕）
- `StepProgress`（Step Flow 進度條）
- `DateWeekPicker`（週日期選擇器）
- `AppSidebar`（側欄，含三種 RWD 變體）
- `AppHeader`（手機版頂部 Header）

---

## 8. 設計系統重要規則

詳細規範見 `shared-docs/DESIGN.md`，以下是最關鍵的幾點：

### 顏色
- 主色：`#006492`（primary）
- 背景：`#F7F9FB`（surface）
- 卡片背景：`#FFFFFF`
- 主要文字：`#191C1E`
- 次要文字：`#3F4850`
- 邊框：`#BEC7D1`

### 狀態顏色
| 狀態 | 文字色 | 背景色 |
|------|--------|--------|
| 待確認（pending） | `#006492` | `#CAE6FF` |
| 已服用（taken） | `#2D6A4F` | `#D8F3DC` |
| 已逾時（overdue） | `#BA1A1A` | `#FFDAD6` |

### 字型
- 字體：Be Vietnam Pro
- 所有字型樣式見 DESIGN.md 的 typography 區塊

### 圓角
- 卡片、按鈕：`12px`（`rounded-xl`）
- 小元件（input、chip）：`8px`（`rounded-lg`）
- 狀態標籤：pill shape（`rounded-full`）

### 間距
- 容器內距：`20px`
- 元件間距：`16px`
- 區塊間距：`32px`

---

## 9. 重要設計決策

### Schedule 與 History 分離
- `Schedule`：規則層，定義「何時應該吃藥」
- `History`：結果層，記錄「那次實際上有沒有吃」
- 前端透過 `GET /schedule-matches` 取得後端已展開的 event instance
- **前端不自己計算排程規則**，所有展開邏輯由後端處理

### 服藥紀錄狀態
- `pending`：待確認，可點擊標記已服用
- `taken`：已服用，不可再操作
- `overdue`：已逾時，仍可點擊補填（不可直接 disable）

### 密碼規則
- 6–18 個字元
- 必須包含大寫字母、特殊字元（!@#$%^&*）
- 不允許空白字元

### API 呼叫
- 所有 API 需攜帶 Bearer Token（`Authorization: Bearer <access_token>`）
- Token 過期時呼叫 `POST /auth/refresh` 更新

### 照護關係權限
- `read`：唯讀
- `write`：可新增、編輯
- `admin`：可管理照護關係（邀請、撤銷）

### 照護關係（CareRelationship）路由位置
照護關係相關頁面位於頂層路由（`/care-relationships`），不在 `/settings` 底下。
`ROUTE.careRelationships.list()` 和 `ROUTE.careInvitations.*` 是頂層常數，不在 `ROUTE.setting` 下。
`/care-invitations` 和 `/care-invitations/new` 皆重定向至 `/care-relationships` 並搭配 query params。

### 病人（Patient）資料來源
病人沒有獨立的管理頁面。
病人資料透過照護關係（CareRelationship）取得。
新增藥品、排程等 Step Flow 中的「選擇照護對象」步驟，
資料來源為當前用戶的照護關係所關聯的 Patient 列表，
透過 `GET /patients` API 取得（後端會根據照護關係過濾）。

---

## 10. 開發注意事項

1. **不要 hardcode 路由字串**，統一從 `constants/route.ts` import
2. **表單驗證邏輯**放 `utils/`，不要寫在 page 元件裡
3. **空白狀態**每個列表頁都需要，使用 `EmptyState` 元件
4. **載入狀態**每個資料請求都需要處理
5. **錯誤狀態**API 錯誤需要顯示給使用者
6. **RWD**所有頁面需支援手機（375px）、平板（768px）、桌機（1280px）

## 11. 文件路徑

所有共用文件放在 `shared-docs/` 目錄下，引用時請使用以下路徑：

| 文件 | 路徑 |
|------|------|
| 產品規格書 | `shared-docs/Medi_Check_產品規格書.md` |
| 技術規格書 | `shared-docs/Medi_Check_技術規格書.md` |
| 設計系統 | `shared-docs/DESIGN.md` |
| Design Tokens | `shared-docs/healing-touch-tokens.json` |

### API Schema
openapi.json 在後端專案維護，需要時從以下位置取得：
https://raw.githubusercontent.com/y0000ga/medi-check-backend/main/docs/openapi.json
