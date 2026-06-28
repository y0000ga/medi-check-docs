# Medi Check Docs

所有 Medi Check 相關文件的統一來源。

## 文件清單

| 文件 | 說明 | 主要更新者 |
|------|------|-----------|
| 產品規格書.md | 功能需求、頁面狀態、欄位定義 | 前端 |
| 技術規格書.md | API 設計、資料庫、系統架構 | 後端 |
| DESIGN.md | 設計系統規範 | 設計 / 前端 |
| healing-touch-tokens.json | Figma design tokens | 設計 |

## API 文件

openapi.json 以後端專案為唯一來源，不在此 repo 維護。

- **線上 API 文件**：https://y0000ga.github.io/medi-check-backend/
- **原始 openapi.json**：https://raw.githubusercontent.com/y0000ga/medi-check-backend/main/docs/openapi.json

## 更新規則

- 功能需求改變 → 更新產品規格書
- API 設計改變 → 更新技術規格書（openapi.json 在後端專案維護）
- 設計系統改變 → 更新 DESIGN.md

## 更新文件的流程

### 修改文件

```bash
# 在 medi-check-docs 目錄裡
git add .
git commit -m "update 產品規格書: 說明改了什麼"
git push
```

### 在前端/後端專案同步最新文件

```bash
# 在 medi-check-frontend 或 medi-check-backend 目錄裡
git submodule update --remote shared-docs
git commit -m "update shared-docs to latest"
git push
```