# TODO：首頁資料同步 改善建議

## 高優先（已有計畫）

- [ ] **Supabase 遷移**：將所有硬編碼資料遷移至 Supabase，前端動態讀取（見 `plan/aipmtw-dashboard-enhancement.md`）
  - 第一階段：建立資料表並匯入資料
  - 第二階段：前端改為動態 fetch
  - 第三階段：巡檢腳本改寫 DB
  - 第四階段：Realtime 訂閱（可選）

## 中優先

- [ ] **卡片自訂化**：允許成員透過 repo 內的設定檔（如 `space.json`）自訂卡片顯示名稱、顏色、描述
- [ ] **快取策略**：前端加入 Service Worker 快取，離線時顯示最近一次資料而非空白
- [ ] **分頁/排序**：成員數量增長後，加入排序選項（按更新時間、提交數、名稱）和分頁
- [ ] **commit hash 遞迴問題**：在 Supabase 方案實施前，考慮將 commit hash 改為非同步更新（先 push，下次巡檢再同步 hash）

## 低優先

- [ ] **多語系支援**：首頁加入英文切換
- [ ] **深色/淺色模式**：加入主題切換功能
- [ ] **SEO 改善**：動態產生 meta tags 和 Open Graph 資料
