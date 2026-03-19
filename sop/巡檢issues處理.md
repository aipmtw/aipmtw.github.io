# SOP：巡檢 Issues 處理

## 目的

定期掃描 aipmtw 組織下所有 public repo 的 open issues，根據授權政策自動處理或留言提示。

## 觸發方式

- **自動排程**：每 15 分鐘執行一次（:05, :20, :35, :50）
- **手動執行**：在 Claude Code 中執行 `/do-issues`
- **CLI 模式**：`claude -p "/do-issues" --dangerously-skip-permissions`

## 流程

### 1. 掃描階段

1. 列出所有 public repo（排除 private repo 如 2026biz、biz-core）
2. 額外加入指定的 private repo（如 `ovb3web`）到巡檢清單
3. 對每個 repo 查詢 open issues

**注意**：private repo 僅巡檢 issues，不寫入 Supabase repos 表（避免顯示在首頁）

### 2. 授權檢查

對每個 open issue 依序檢查：

| 優先序 | 條件 | 動作 |
|--------|------|------|
| 1 | 標題以 `[申請空間]` 或 `[刪除空間]` 開頭 | 跳過，等待管理員 `/approve` |
| 2 | 標題以 `[申請日記功能]` 開頭 | Collaborator/Owner 提交 → 直接處理（無需 /approve） |
| 3 | 提交者是該 repo 的 Owner | 正常處理 |
| 4 | 提交者是該 repo 的 Collaborator | 正常處理 |
| 5 | 提交者非授權用戶 | 留言提示需 `/approve`，不關閉 |

**重要**：
- 每次巡檢都必須即時查詢 collaborator 清單（`gh api repos/aipmtw/<repo>/collaborators`），不得使用快取或歷史記錄
- Owner 檢查：`gh api repos/aipmtw/<repo> --jq '.owner.login'`，owner 不在 collaborator API 回傳中

### 3. 處理授權 Issue

1. 確認本地目錄存在（不存在則 clone）
2. `git pull` 取得最新程式碼
3. 讀取相關程式碼，分析 issue 內容
4. 判斷處理方式：
   - **可修改** → 修改程式碼 → commit push → 留言說明 → 關閉 issue
   - **已完成** → 留言說明 → 關閉 issue
   - **需更多資訊** → 留言分析，保持 open

### 4. 非授權 Issue 留言

留言內容（僅留言一次，避免重複）：

> 🤖 /do-issues 巡檢提示：此 issue 的提交者 @{author} 目前不是本 repo 的 Collaborator。管理員請留言 /approve 以核准處理此需求。

管理員看到後可選擇：
- 留言 `/approve` → 下次巡檢會處理
- 將該用戶加為 Collaborator → 下次巡檢自動識別並處理
- 不理會 → issue 保持 open

### 5. 報告與同步

1. 寫入巡檢報告（logs/ 和 aitech/）
2. 使用 `supabase-helper.js` 同步資料到 Supabase 資料庫：
   - `upsert-repo`：各 repo 資訊（commit hash、時間、有效提交數）
   - `upsert-issue`：各 repo 最近已完成 issue 紀錄
   - `add-check`：巡檢結果摘要與各 repo 詳情
   - 新增 repo 偵測（自動 INSERT）
   - 已刪除 repo 清理（`deactivate-repo`）
3. commit push 巡檢 log 到 2026biz repo
4. **不再編輯 index.html**，前端自動從 Supabase 讀取最新資料

## 輸出檔案

| 檔案 | 路徑 | 內容 |
|------|------|------|
| 完整巡檢 log | `2026biz/logs/<YYYYMMDD>/<HHMM>-summary.md` | 各 repo 詳細處理記錄 |
| 巡檢摘要 | `2026biz/aitech/<MMDD>-<HHMM>-do-issues.md` | 簡短結果摘要 |

## 管理員操作指南

| 場景 | 操作方式 |
|------|---------|
| 核准非 Collaborator 的 issue | 在 issue 留言 `/approve` |
| 永久授權某用戶 | 將用戶加為 repo Collaborator |
| 暫停巡檢 | 取消 cron 排程或關閉 Claude Code session |
