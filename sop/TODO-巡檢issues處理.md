# TODO：巡檢 Issues 處理 改善建議

## 高優先

- [x] **Supabase 整合**：巡檢結果寫入資料庫而非改寫 HTML，消除 commit hash 遞迴問題 ✅ 已完成，使用 supabase-helper.js（upsert-repo, upsert-issue, add-check）
- [ ] **`/approve` 自動處理**：目前管理員留言 `/approve` 後需等下次巡檢才會處理，考慮用 GitHub Actions 監聽 `/approve` 留言即時觸發處理
- [ ] **巡檢失敗通知**：若巡檢中途出錯（API 逾時、push 失敗），目前無通知機制，考慮加入 Slack/Email 告警

## 中優先

- [ ] **Issue 分類處理**：目前所有 issue 都當作程式碼修改需求處理，可加入分類邏輯（bug fix、feature request、question）給予不同回應
- [ ] **處理品質回饋**：issue 關閉後讓提交者可評分或回報問題，改善 AI 處理品質
- [ ] **平行處理**：目前逐一處理 repo，多個 repo 同時有 issue 時可用 subagent 平行處理
- [ ] **commit message 改善**：目前 commit message 較籠統，可根據 issue 內容生成更精確的 message

## 低優先

- [ ] **巡檢儀表板**：建立獨立頁面展示歷史巡檢紀錄與趨勢，而非只顯示最近一次結果
- [ ] **Issue 模板**：為各 repo 建立 Issue 模板，引導用戶填寫結構化需求，提升 AI 處理準確度
- [ ] **自動回歸測試**：處理完 issue 後自動跑 build 驗證不破壞現有功能
