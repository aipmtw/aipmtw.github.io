# SOP：技術注意事項（Tech Notices）

## ORG_ADMIN_TOKEN 權限需求

`ORG_ADMIN_TOKEN`（Personal Access Token）需要以下 scope：

| Scope | 用途 |
|-------|------|
| `repo` | 建立 repo、推送程式碼、管理 collaborators |
| `admin:org` | 管理組織設定 |
| `delete_repo` | 刪除 repo（delete-space workflow 需要） |

**注意**：若缺少 `delete_repo` scope，`delete-space.yml` workflow 會在 "Delete repo" 步驟失敗（HTTP 403）。

### 如何更新 Token

1. 前往 https://github.com/settings/tokens
2. 編輯現有 Token 或建立新 Token
3. 確認勾選 `repo`、`admin:org`、`delete_repo`
4. 複製新 Token
5. 前往 https://github.com/aipmtw/aipmtw.github.io/settings/secrets/actions
6. 更新 `ORG_ADMIN_TOKEN` 的值

## Git Clone 後不可再 git init + remote add

- `git clone` 後 origin remote 已存在
- **錯誤**：`git init` + `git remote add origin` → `error: remote origin already exists`
- **正確**：clone 後直接 `git add` → `git commit` → `git push`
- 若需更換 URL：用 `git remote set-url origin <url>`

## GitHub Template Repo 注意事項

- `gh repo create --template` 有時不會立即複製檔案（repo 顯示 size=0）
- **解決方案**：workflow 中 clone 後檢查是否有檔案，若為空則手動從 template 複製
- 目前 workflow 採用 clone → sed 替換的方式，若 template 為空則會失敗

## Workflow 觸發注意事項

- `issue_comment` 事件會觸發 repo 中**所有**監聽該事件的 workflow
- `create-space.yml` 和 `delete-space.yml` 都監聯 `issue_comment`
- 每次 `/approve` 留言會同時觸發兩個 workflow，由 `if` 條件篩選
- `if` 條件使用 `contains()` 而非 `==` 來比對 comment body
- 確保 Issue 標題前綴正確：`[申請空間]` 或 `[刪除空間]`

## 巡檢中的 [申請空間] / [刪除空間] Issue

- 這類 Issue 是申請流程的一部分，不應被當作一般 Issue 靜默關閉
- 提交者通常不是 aipmtw.github.io 的 Collaborator（他們是新申請者）
- 巡檢時應跳過這類 Issue，標記為「待管理員審批」
