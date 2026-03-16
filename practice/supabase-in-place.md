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

## 資料表結構

| 資料表 | 用途 |
|--------|------|
| `repos` | Repo 基本資料（名稱、URL、顏色、commit 資訊、issue 數量） |
| `issues` | Issue 處理紀錄（repo、issue 編號、作者、摘要） |
| `check_results` | 巡檢紀錄（時間、摘要） |
| `check_details` | 巡檢各 repo 詳情（open issues、結果） |

## 安全設計

- **Row Level Security (RLS)**：所有資料表啟用 RLS
- **公開讀取**：前端使用 publishable key（anon），僅有 SELECT 權限
- **寫入限制**：僅 service_role key 可寫入，存放於 AI 巡檢環境變數中

## 技術選型

- **Supabase**：免費方案足夠使用（50K 月活、500MB 資料庫）
- **supabase-js v2**：透過 ESM CDN（esm.sh）直接在前端引入，無需建置工具
- **PostgREST**：Supabase 內建 REST API，前端直接查詢

## 實作階段

1. **第一階段**：建立 Supabase 專案 + 資料表 + 匯入現有資料 ✅
2. **第二階段**：前端改造（動態卡片 + Supabase fetch） ✅
3. **第三階段**：巡檢腳本改造（寫 DB 取代改 HTML）
4. **第四階段**：Realtime 即時更新（可選）

## Supabase 專案資訊

- 專案名稱：aipm001
- URL：`https://reipdepbltfbfxnjjegy.supabase.co`
- 方案：Nano（免費）
