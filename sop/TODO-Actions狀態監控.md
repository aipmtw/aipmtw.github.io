# TODO：Actions 狀態監控 改善建議

## 高優先

- [ ] **自動修復擴展**：目前僅自動修復 404 錯誤，考慮擴展至 Actions 失敗的常見原因（npm install 失敗、build error）
- [ ] **異常告警**：連續 2 次以上 Actions 失敗時，自動在 repo 建立 issue 通知 Collaborator

## 中優先

- [ ] **效能監控**：記錄各網頁的回應時間和頁面大小，偵測效能劣化
- [ ] **SSL 憑證檢查**：監控 HTTPS 憑證到期時間（GitHub Pages 自動管理，但自訂域名需注意）
- [ ] **部署歷史追蹤**：記錄每次 Actions 成功部署的時間和 commit，建立部署時間線

## 低優先

- [ ] **視覺化報告**：將 Actions 狀態整合至首頁，用綠/紅燈顯示各 repo 健康狀態
- [ ] **Webhook 整合**：接收 GitHub Actions 完成的 webhook，即時更新狀態而非定期輪詢
