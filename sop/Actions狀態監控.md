# SOP：Actions 狀態監控

## 目的

監控 aipmtw 組織下所有 repo 的 GitHub Actions 執行狀態及網頁可用性，確保所有成員網站正常運作。

## 觸發方式

- **手動執行**：在 Claude Code 中執行 `/check-actions`
- **CLI 模式**：`claude -p "/check-actions" --dangerously-skip-permissions`

## 流程

### 1. 取得 Repo 清單

列出所有 public repo，並與首頁卡片比對，偵測已刪除的 repo。

### 2. Actions 狀態檢查

對每個 repo 查詢最新一次 workflow 執行結果：

| 狀態 | 說明 |
|------|------|
| ✅ success | Actions 執行成功 |
| ❌ failure | Actions 執行失敗 |
| 🔄 in_progress | 正在執行中 |
| ⚪ 無紀錄 | 尚未執行過 Actions |

### 3. 網頁可用性檢查

對每個有網頁的 repo 進行 HTTP 檢查：

| 檢查項目 | 判斷標準 |
|---------|---------|
| HTTP 狀態碼 | 200 = 正常 |
| 頁面大小 | > 1000 bytes（靜態頁面） |
| SPA 部署 | HTML 中包含 `assets/` 引用 = 正確部署 |
| 部署錯誤 | 包含 `src="/src/main.jsx"` 但無 `assets/` = 未正確建置 |

### 4. 自動修復（僅限 404 錯誤）

- 檢查 Pages 設定是否正確（build_type = workflow）
- 嘗試觸發重新部署
- 等待 60 秒後驗證修復結果

### 5. 報告

寫入巡檢報告到 `2026biz/logs/` 和 `2026biz/aitech/`。

## 輸出檔案

| 檔案 | 路徑 |
|------|------|
| 完整 log | `2026biz/logs/<YYYYMMDD>/<HHMM>-actions.md` |
| 摘要 | `2026biz/aitech/<MMDD>-<HHMM>-check-actions.md` |

## 管理員操作指南

| 問題 | 處理方式 |
|------|---------|
| Actions 失敗 | 檢查 workflow 日誌，修復程式碼或設定 |
| 網頁 404 | 確認 Pages 設定為 workflow build type |
| SPA 未正確部署 | 檢查 vite.config.js 的 base 設定 |
