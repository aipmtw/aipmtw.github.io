# SOP：刪除個人空間（Delete Repo）

## 權限說明

- **只有組織管理員（aipmtw）可以刪除 repo**
- 一般 Collaborator（push 權限）無法直接刪除 repo
- 刪除必須透過 Issue 申請 + 管理員 `/approve`

## 刪除方式

### 方式一：成員透過 Issue 申請刪除（標準流程）

1. 成員在 aipmtw.github.io repo 建立 Issue
   - 標題格式：`[刪除空間] {空間名稱} - {GitHub Account}`
   - 內文需包含簽章（Sign），由系統或手動計算
   - Sign = SHA-256(`aipmtw-2026-delete|{name}|{account}`) 前 16 碼
2. 管理員確認後在 Issue 留言 `/approve`
3. GitHub Actions `delete-space.yml` 自動執行：
   - 驗證簽章（防竄改）
   - 驗證標題與內文一致性
   - 刪除 repo
   - 留言通知並關閉 Issue

### 方式二：管理員直接刪除（手動）

```bash
# 1. 刪除 repo
gh repo delete aipmtw/{name} --yes

# 2. 刪除本地目錄（如有）
rm -rf C:/2026aipm-projects/{name}

# 3. 下次 /do-issues 會自動從首頁移除卡片
```

## 自動偵測與清理

### /check-actions 偵測

每次巡檢時比對首頁卡片與實際 repo：
- `gh api repos/aipmtw/{name}` 回傳 404 → 報告中標記 `🗑️ repo 已刪除`

### /do-issues 自動清理

每次巡檢卡片同步時：
- 若 repo 已不存在（API 回傳 404）→ 自動移除卡片、issueData、repoNames 等
- 巡檢報告記錄「🗑️ {name} repo 已刪除，卡片已移除」

## 刪除後效果

- 該用戶的空間數 -1，可再次申請新空間
- 該空間名稱重新開放
- GitHub Pages 自動停止
- repo 刪除後無法恢復

## 簽章驗證

刪除申請同樣採用 SHA-256 簽章驗證：
- Salt: `aipmtw-2026-delete`
- 格式: `SHA-256(aipmtw-2026-delete|{name}|{account})` 取前 16 碼
- 簽章不符或缺失 → 拒絕刪除並留言說明
