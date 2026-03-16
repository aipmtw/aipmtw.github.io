# Supabase 就地整合實作紀錄

## 概述

將 AIPMTW Dashboard（aipmtw.github.io）從靜態硬編碼架構遷移至 Supabase 資料庫驅動架構，讓 AI 巡檢不再需要每次修改 HTML 並 commit push。

## 核心變更

### 遷移前
- 所有資料（repo 清單、issue 紀錄、巡檢結果）硬編碼在 `index.html` 的 JavaScript 物件中
- AI 巡檢每 15 分鐘修改 HTML → commit → push，產生大量無意義 commit
- 新增/刪除 repo 需手動修改 HTML 結構

### 遷移後
- 資料存放於 Supabase PostgreSQL 資料庫
- 前端透過 Supabase JS Client 即時讀取資料，動態產生卡片
- AI 巡檢改為呼叫 API 寫入資料庫，不再修改任何檔案
- 新增/刪除 repo 只需操作資料庫

## 資料表完整規格

### 1. `repos` — Repo 基本資料

存放所有成員空間的 repo 資訊，前端據此動態產生卡片。

| 欄位 | 型別 | 說明 | 範例 |
|------|------|------|------|
| `id` | TEXT (PK) | Repo 名稱，也是唯一識別 | `sherry`, `swim-meet` |
| `display_name` | TEXT (NOT NULL) | 卡片顯示名稱 | `Sherry`, `Robert (Swim Meet)` |
| `description` | TEXT | Repo 說明（可選） | `null` |
| `url` | TEXT (NOT NULL) | 網頁 URL | `https://aipmtw.github.io/sherry/` |
| `github_url` | TEXT (NOT NULL) | GitHub repo URL | `https://github.com/aipmtw/sherry` |
| `card_color_from` | TEXT | 卡片漸層起始色 | `pink-500`, `indigo-500` |
| `card_color_to` | TEXT | 卡片漸層結束色 | `rose-500`, `blue-500` |
| `avatar_letter` | CHAR(1) | 卡片頭像字母 | `S`, `R` |
| `last_commit_hash` | TEXT | 最新 commit 短 hash（7 碼） | `6e1a511` |
| `last_commit_time` | TIMESTAMPTZ | 最新 commit 時間 | `2026-03-15T12:55:38+00:00` |
| `valid_issue_count` | INT | 有效提交數（Collaborator 提交的已關閉 issues 總數） | `4` |
| `is_active` | BOOLEAN | 是否啟用（刪除 repo 時設為 false） | `true` |
| `created_at` | TIMESTAMPTZ | 建立時間 | 自動 |
| `updated_at` | TIMESTAMPTZ | 更新時間 | 自動 |

**使用方式**：

| 操作者 | 操作 | 方式 |
|--------|------|------|
| 前端 | 讀取所有啟用的 repo | `supabase.from('repos').select('*').eq('is_active', true)` |
| 巡檢腳本 | 同步 commit 資訊 | `supabase-helper.js upsert-repo '{...}'` |
| 巡檢腳本 | 新增 repo | `supabase-helper.js upsert-repo '{全部欄位}'` |
| 巡檢腳本 | 停用已刪除 repo | `supabase-helper.js deactivate-repo '<id>'` |

---

### 2. `issues` — Issue 處理紀錄

記錄各 repo 已完成的 issue，前端點擊「有效提交數」按鈕時顯示。

| 欄位 | 型別 | 說明 | 範例 |
|------|------|------|------|
| `id` | SERIAL (PK) | 自動遞增 ID | `1` |
| `repo_id` | TEXT (FK → repos.id) | 所屬 repo | `sherry` |
| `issue_number` | INT | GitHub issue 編號 | `4` |
| `author` | TEXT | 提交者 GitHub 帳號 | `aipmtw` |
| `title` | TEXT | Issue 標題（可選） | `null` |
| `summary` | TEXT (NOT NULL) | 非技術用語繁體中文簡述 | `頁面加入需求提交按鈕` |
| `status` | TEXT | 狀態 | `closed` |
| `completed_at` | TIMESTAMPTZ | 完成時間 | `2026-03-15T12:56:17+00:00` |
| `created_at` | TIMESTAMPTZ | 記錄建立時間 | 自動 |

**使用方式**：

| 操作者 | 操作 | 方式 |
|--------|------|------|
| 前端 | 讀取 repo 最近 3 筆 issue | `supabase.from('issues').select('*').eq('repo_id', id).order('completed_at', {ascending: false}).limit(3)` |
| 巡檢腳本 | 同步 issue 紀錄 | `supabase-helper.js upsert-issue '{...}'` |

---

### 3. `check_results` — 巡檢紀錄

每次巡檢新增一筆，記錄巡檢時間和摘要。前端用於顯示「最後巡檢時間」和「巡檢記錄」彈窗。

| 欄位 | 型別 | 說明 | 範例 |
|------|------|------|------|
| `id` | SERIAL (PK) | 自動遞增 ID | `8` |
| `check_time` | TIMESTAMPTZ (NOT NULL) | 巡檢執行時間 | `2026-03-16T04:05:56+00:00` |
| `summary` | TEXT (NOT NULL) | 巡檢摘要 | `掃描 15 個 repo，全部正常，無待處理 issues` |
| `created_at` | TIMESTAMPTZ | 記錄建立時間 | 自動 |

**使用方式**：

| 操作者 | 操作 | 方式 |
|--------|------|------|
| 前端 | 讀取最新巡檢（顯示巡檢時間按鈕） | `supabase.from('check_results').select('*, check_details(*)').order('check_time', {ascending: false}).limit(1).single()` |
| 前端 | 讀取最近 7 次巡檢（巡檢記錄彈窗） | `supabase.from('check_results').select('*, check_details(*)').order('check_time', {ascending: false}).limit(7)` |
| 巡檢腳本 | 新增巡檢紀錄 | `supabase-helper.js add-check '{check_time, summary, details: [...]}'` |

---

### 4. `check_details` — 巡檢各 repo 詳情

每次巡檢對每個 repo 新增一筆詳情，記錄該 repo 的 open issues 數和處理結果。

| 欄位 | 型別 | 說明 | 範例 |
|------|------|------|------|
| `id` | SERIAL (PK) | 自動遞增 ID | `91` |
| `check_id` | INT (FK → check_results.id) | 所屬巡檢 | `8` |
| `repo_id` | TEXT (FK → repos.id) | Repo 名稱 | `raymondliu` |
| `open_issues` | INT | 該次巡檢偵測到的 open issues 數 | `0` |
| `result` | TEXT (NOT NULL) | 處理結果 | `無待處理`, `✅ #2 已處理（AI 使用筆記）` |

**使用方式**：

| 操作者 | 操作 | 方式 |
|--------|------|------|
| 前端 | 透過 check_results 的 join 讀取 | `check_details(*)` nested select |
| 巡檢腳本 | 隨 check_results 一同寫入 | `supabase-helper.js add-check` 的 details 陣列 |

---

### 5. `changenotes` — 重大改版記要

記錄平台的重大改版紀錄，前端「重大改版記要」彈窗顯示。

| 欄位 | 型別 | 說明 | 範例 |
|------|------|------|------|
| `id` | SERIAL (PK) | 自動遞增 ID | `1` |
| `date` | DATE | 改版日期 | `2026-03-16` |
| `title` | TEXT | 改版標題 | `首頁升級為即時資料驅動` |
| `body` | TEXT | 改版詳細說明（支援換行和列表） | `首頁不再使用固定寫死的資料...` |
| `created_at` | TIMESTAMPTZ | 記錄建立時間 | 自動 |

**使用方式**：

| 操作者 | 操作 | 方式 |
|--------|------|------|
| 前端 | 讀取所有改版記要 | `supabase.from('changenotes').select('date, title, body').order('date', {ascending: false})` |
| 管理員 | 新增改版記要 | Supabase Dashboard 手動新增，或透過 API |

---

## 資料表關聯

```
repos (id)
  ├── issues (repo_id → repos.id)
  ├── check_details (repo_id → repos.id)
  └── [獨立] changenotes

check_results (id)
  └── check_details (check_id → check_results.id)
```

## 安全設計

- **Row Level Security (RLS)**：所有資料表啟用 RLS
- **公開讀取**：前端使用 publishable key（anon），僅有 SELECT 權限
- **寫入限制**：僅 service_role key 可寫入，存放於 `$BASE_DIR/2026biz/.env.supabase`

### RLS 政策摘要

| 資料表 | SELECT | INSERT/UPDATE/DELETE |
|--------|--------|---------------------|
| `repos` | anon ✅ | service_role only |
| `issues` | anon ✅ | service_role only |
| `check_results` | anon ✅ | service_role only |
| `check_details` | anon ✅ | service_role only |
| `changenotes` | anon ✅ | service_role only |

## 前端讀取方式

```javascript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

// 載入 repo 卡片
const { data: repos } = await supabase.from('repos').select('*').eq('is_active', true)

// 載入 issue 紀錄（點擊卡片時）
const { data: issues } = await supabase.from('issues').select('*').eq('repo_id', id).order('completed_at', {ascending: false}).limit(3)

// 載入最新巡檢（頁面初始化）
const { data } = await supabase.from('check_results').select('*, check_details(*)').order('check_time', {ascending: false}).limit(1).single()

// 載入巡檢記錄（點擊「巡檢記錄」按鈕）
const { data: logs } = await supabase.from('check_results').select('*, check_details(*)').order('check_time', {ascending: false}).limit(7)

// 載入改版記要（點擊「重大改版記要」按鈕）
const { data: notes } = await supabase.from('changenotes').select('date, title, body').order('date', {ascending: false})
```

## 巡檢腳本寫入方式

使用 `supabase-helper.js`（位於 `$BASE_DIR/2026biz/`）：

```bash
# 寫入巡檢結果 + 各 repo 詳情
node supabase-helper.js add-check '{"check_time":"...","summary":"...","details":[{"repo_id":"...","open_issues":0,"result":"..."},...]}'

# 同步 repo commit 資訊
node supabase-helper.js upsert-repo '{"id":"...","last_commit_hash":"...","last_commit_time":"..."}'

# 同步 issue 紀錄
node supabase-helper.js upsert-issue '{"repo_id":"...","issue_number":1,"author":"...","summary":"...","status":"closed","completed_at":"..."}'

# 列出啟用中的 repo
node supabase-helper.js list-active-repos

# 停用已刪除的 repo
node supabase-helper.js deactivate-repo '<repo_id>'
```

## 技術選型

- **Supabase**：免費方案足夠使用（50K 月活、500MB 資料庫）
- **supabase-js v2**：透過 ESM CDN（esm.sh）直接在前端引入，無需建置工具
- **PostgREST**：Supabase 內建 REST API，前端直接查詢

## Supabase 專案資訊

- 專案名稱：aipm001
- URL：`https://reipdepbltfbfxnjjegy.supabase.co`
- 方案：Nano（免費）
- 認證檔位置：`$BASE_DIR/2026biz/.env.supabase`

## 實作階段

1. **第一階段**：建立 Supabase 專案 + 資料表 + 匯入現有資料 ✅
2. **第二階段**：前端改造（動態卡片 + Supabase fetch） ✅
3. **第三階段**：巡檢腳本改造（寫 DB 取代改 HTML） ✅
4. **第四階段**：Realtime 即時更新（可選）
