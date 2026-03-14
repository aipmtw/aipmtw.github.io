# SOP：刪除個人空間（Delete Repo）

## 權限說明

- **只有組織管理員（aipmtw）可以刪除 repo**
- 一般 Collaborator（push 權限）無法刪除 repo
- 刪除需透過管理員操作或提交 Issue 申請

## 刪除方式

### 方式一：成員提交 Issue 申請刪除

成員在 aipmtw.github.io repo 建立 Issue，標題格式：`[刪除空間] {空間名稱}`
管理員確認後執行刪除。

### 方式二：管理員直接刪除

```bash
# 1. 刪除 repo
gh repo delete aipmtw/{name} --yes

# 2. 刪除本地目錄（如有）
rm -rf C:/2026aipm-projects/{name}

# 3. 下次 /check-issues 會自動從首頁移除卡片
```

## 自動偵測與清理

### /check-actions 偵測

每次巡檢時比對首頁卡片與實際 repo：
- `gh api repos/aipmtw/{name}` 回傳 404 → 報告中標記 `🗑️ repo 已刪除`

### /check-issues 自動清理

每次巡檢卡片同步時：
- 若 repo 已不存在（API 回傳 404）→ 自動執行：
  1. 移除 index.html 中對應的卡片區塊
  2. 移除 `issueData` / `repoNames` 中該 repo 的資料
  3. 移除 `checkResult.details` 中該 repo
  4. Commit push 更新首頁
  5. 巡檢報告中記錄「🗑️ {name} repo 已刪除，卡片已移除」

## 刪除後效果

- 該用戶的空間數 -1，可再次申請新空間
- 該空間名稱重新開放，其他人可申請
- GitHub Pages 自動停止
- aipm.com.tw/{name}/ 將返回 404
- repo 刪除後無法恢復，所有 commit 歷史消失
