# TODO：成員授權管理 改善建議

## 高優先

- [ ] **Collaborator 管理介面**：在首頁新增管理員專區，可直接查看/新增/移除各 repo 的 Collaborator，取代命令列操作
- [ ] **`/approve` 自動加 Collaborator**：管理員對非 Collaborator 的 issue 留言 `/approve` 後，自動將該用戶加為 Collaborator 並處理 issue

## 中優先

- [ ] **權限分級**：目前所有 Collaborator 都是 push 權限，可考慮區分「只能提 issue」和「可以 push 程式碼」兩種角色
- [ ] **成員儀表板**：讓每位成員可看到自己的空間狀態、issue 處理歷史、空間用量
- [ ] **邀請連結**：產生一次性邀請連結，用戶點擊後自動加為指定 repo 的 Collaborator

## 低優先

- [ ] **活躍度追蹤**：記錄各 Collaborator 的最後活動時間，長期不活躍的自動標記
- [ ] **批次管理**：支援一次操作多個 repo 的 Collaborator（如將某用戶加到所有 repo）
