# AIPMTW Dashboard Enhancement：Supabase 資料庫整合計畫

> 目標：將首頁所有硬編碼資料遷移至 Supabase，讓頁面自動從資料庫讀取最新狀態，不再依賴 AI 每次巡檢時改寫 index.html 並 commit push。

## 現況問題

目前 `index.html` 是一個單一靜態檔案，所有資料都以 JavaScript 物件硬編碼：

| 資料 | 目前存放方式 | 問題 |
|------|-------------|------|
| `issueData` | JS 物件（15 個 repo 的 issue 紀錄） | 每次處理 issue 都要改 HTML 並 push |
| `repoNames` | JS 物件（repo → 顯示名稱對照） | 新增/刪除 repo 需改 HTML |
| `checkResult` | JS 物件（巡檢時間、摘要、各 repo 狀態） | 每 15 分鐘巡檢都要改 HTML 並 push |
| 卡片 HTML | 硬編碼 15 張卡片 | 新增/刪除成員需手動改 HTML 結構 |
| 時間戳記 | 散落在 HTML 各處 | commit hash 遞迴更新問題 |

**核心痛點**：AI 巡檢每 15 分鐘執行一次，每次都要讀取整個 HTML → 修改資料 → commit push → 處理 commit hash 不一致問題。這既浪費 AI 算力，也產生大量無意義的 git commit。

## 目標架構

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Browser   │────▶│  Supabase    │◀────│  AI 巡檢腳本    │
│ index.html  │     │  (PostgreSQL │     │  /do-issues   │
│ 前端 JS     │     │   + REST API)│     │  /check-actions  │
└─────────────┘     └──────────────┘     └─────────────────┘
      │                    │
      │  fetch 資料        │  寫入巡檢結果
      │  即時顯示          │  更新 issue 紀錄
      ▼                    ▼
  頁面自動更新         資料即時入庫
  不需 commit          不需改 HTML
```

## Supabase 資料表設計

### 1. `repos` — Repo 基本資料

```sql
CREATE TABLE repos (
  id TEXT PRIMARY KEY,              -- e.g. 'sherry', 'swim-meet'
  display_name TEXT NOT NULL,       -- e.g. 'Sherry', 'Robert (Swim Meet)'
  description TEXT,                 -- repo 說明
  url TEXT NOT NULL,                -- e.g. 'https://aipmtw.github.io/sherry/'
  github_url TEXT NOT NULL,         -- e.g. 'https://github.com/aipmtw/sherry'
  card_color_from TEXT DEFAULT 'indigo-500',  -- 卡片漸層起始色
  card_color_to TEXT DEFAULT 'blue-500',      -- 卡片漸層結束色
  avatar_letter CHAR(1),            -- 卡片頭像字母
  last_commit_hash TEXT,            -- 最新 commit 短 hash
  last_commit_time TIMESTAMPTZ,     -- 最新 commit 時間
  valid_issue_count INT DEFAULT 0,  -- 有效提交數
  is_active BOOLEAN DEFAULT true,   -- repo 是否存在
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 2. `issues` — Issue 處理紀錄

```sql
CREATE TABLE issues (
  id SERIAL PRIMARY KEY,
  repo_id TEXT REFERENCES repos(id) ON DELETE CASCADE,
  issue_number INT NOT NULL,
  author TEXT NOT NULL,              -- GitHub username
  title TEXT,
  summary TEXT NOT NULL,             -- 非技術用語繁體中文簡述
  status TEXT DEFAULT 'closed',      -- 'open', 'closed', 'pending_approve'
  completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(repo_id, issue_number)
);
```

### 3. `check_results` — 巡檢紀錄

```sql
CREATE TABLE check_results (
  id SERIAL PRIMARY KEY,
  check_time TIMESTAMPTZ NOT NULL,
  summary TEXT NOT NULL,             -- 巡檢摘要
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 4. `check_details` — 巡檢各 repo 詳情

```sql
CREATE TABLE check_details (
  id SERIAL PRIMARY KEY,
  check_id INT REFERENCES check_results(id) ON DELETE CASCADE,
  repo_id TEXT REFERENCES repos(id) ON DELETE CASCADE,
  open_issues INT DEFAULT 0,
  result TEXT NOT NULL               -- e.g. '✅ #2 已處理', '無待處理'
);
```

### 5. `collaborators` — Collaborator 快取（可選）

```sql
CREATE TABLE collaborators (
  repo_id TEXT REFERENCES repos(id) ON DELETE CASCADE,
  username TEXT NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY(repo_id, username)
);
```

## 實作階段

### 第一階段：建立 Supabase 專案與資料表

1. 建立 Supabase 專案
2. 執行上述 SQL 建立資料表
3. 設定 Row Level Security (RLS)：
   - `repos`, `issues`, `check_results`, `check_details` → 公開讀取（anon key）
   - 寫入限制為 service_role key（僅 AI 巡檢腳本使用）
4. 匯入現有資料（從目前 index.html 的 JS 物件轉入）

### 第二階段：改造前端（index.html）

將硬編碼資料改為從 Supabase 即時讀取：

```javascript
// 引入 Supabase client
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

// 載入 repo 資料並動態產生卡片
async function loadRepos() {
  const { data: repos } = await supabase
    .from('repos')
    .select('*')
    .eq('is_active', true)
    .order('display_name')

  renderCards(repos)
}

// 載入 issue 紀錄
async function loadIssues(repoId) {
  const { data: issues } = await supabase
    .from('issues')
    .select('*')
    .eq('repo_id', repoId)
    .order('completed_at', { ascending: false })
    .limit(3)

  renderIssueModal(issues)
}

// 載入最新巡檢結果
async function loadCheckResult() {
  const { data } = await supabase
    .from('check_results')
    .select('*, check_details(*)')
    .order('check_time', { ascending: false })
    .limit(1)
    .single()

  renderCheckModal(data)
}
```

**卡片動態產生**：不再硬編碼 15 張卡片 HTML，改為用 JS 根據 `repos` 表的資料動態建立 DOM。新增/刪除 repo 只需操作資料庫，前端自動反映。

### 第三階段：改造 AI 巡檢腳本（/do-issues）

巡檢腳本改為寫入 Supabase，不再修改 index.html：

```bash
# 巡檢流程變更
# 舊：讀 HTML → 改 JS 物件 → commit push
# 新：查 GitHub API → 寫入 Supabase → 完成

# 使用 curl 或 supabase CLI 寫入
curl -X POST "$SUPABASE_URL/rest/v1/check_results" \
  -H "apikey: $SUPABASE_SERVICE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"check_time": "...", "summary": "..."}'
```

**巡檢腳本需要的環境變數**：
- `SUPABASE_URL`：Supabase 專案 URL
- `SUPABASE_SERVICE_KEY`：Service Role Key（有寫入權限）

### 第四階段：即時更新（可選）

使用 Supabase Realtime 訂閱，讓已開啟的頁面自動更新：

```javascript
supabase
  .channel('repos-changes')
  .on('postgres_changes', { event: '*', schema: 'public', table: 'repos' },
    () => loadRepos()
  )
  .subscribe()
```

## 遷移前後對比

| 面向 | 遷移前 | 遷移後 |
|------|--------|--------|
| 資料存放 | 硬編碼在 index.html | Supabase 資料庫 |
| 巡檢更新 | 改 HTML → commit → push | API 寫入資料庫 |
| 新增 repo | 手動加卡片 HTML + JS 資料 | INSERT 一筆到 repos 表 |
| 刪除 repo | 手動刪卡片 HTML + JS 資料 | UPDATE is_active = false |
| Git commit | 每次巡檢 1-3 個 commit | 僅程式碼變更才 commit |
| 頁面載入 | 靜態，資料可能過時 | 動態 fetch，永遠最新 |
| AI 算力 | 讀寫大量 HTML | 只做 API 呼叫 |
| commit hash 問題 | 遞迴更新無法避免 | 完全消除 |

## 風險與注意事項

1. **Supabase 免費方案限制**：50,000 月活躍用戶、500MB 資料庫、1GB 檔案儲存，對本專案綽綽有餘
2. **API Rate Limit**：Supabase anon key 預設無嚴格限制，但建議前端加入快取（例如 5 分鐘內不重複 fetch）
3. **離線降級**：若 Supabase 無法連線，前端應顯示「資料載入中」提示而非空白頁面，可考慮保留最後一次靜態快照作為 fallback
4. **安全性**：Service Role Key 僅存於 AI 巡檢環境（環境變數），不暴露在前端程式碼中
5. **遷移期間**：可先讓前端同時支援靜態資料與 Supabase 資料，確認穩定後再移除靜態資料

## 時程建議

| 階段 | 工作項目 | 預估工作量 |
|------|---------|-----------|
| 第一階段 | Supabase 設定 + 資料表 + 匯入資料 | 1 次對話 |
| 第二階段 | 前端改造（動態卡片 + fetch 資料） | 1-2 次對話 |
| 第三階段 | 巡檢腳本改造（寫 DB 取代改 HTML） | 1 次對話 |
| 第四階段 | Realtime 訂閱（可選） | 1 次對話 |

## 結論

透過 Supabase 整合，AI 巡檢腳本將從「讀寫 HTML 檔案」轉變為「呼叫 API 寫入資料庫」，大幅降低複雜度與錯誤率。前端頁面則從靜態展示進化為動態資料驅動，新增或移除 repo 只需操作資料庫，無需任何程式碼變更。
