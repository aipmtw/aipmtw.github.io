# SOP：建立個人空間（Create Repo）

## 觸發方式

### 方式一：透過首頁申請表單（自助）

1. 申請人在 https://aipm.com.tw/ 底部填寫：
   - **空間名稱**（小寫英文+數字+連字號，2-32 字元）
   - **GitHub Account**
2. 系統即時驗證：
   - 空間名稱是否可用（不重複、不在待審批中）
   - GitHub 帳號是否存在（顯示頭像確認）
   - 該用戶是否已達上限（每人最多 2 個空間）
3. 兩個都通過後點擊「提交申請」
4. **跳轉至 GitHub Issue 預填頁面，由申請人自己登入 GitHub 帳號送出 Issue**
   - 這樣可以確保 Issue 的作者就是實際申請人
   - Issue 標題格式：`[申請空間] {名稱} - {帳號}`
5. 管理員收到通知，在 Issue 留言 `/approve` 核准

### 方式二：管理員直接建立（手動）

管理員在 Claude Code 中直接執行建立流程。

## 重要原則

- **Issue 必須由申請人本人建立**，不可由系統代建，以確保能追溯實際申請者身份
- Issue 作者 = 實際申請人，這是授權驗證的基礎

## 模板 Repo

所有新空間基於 `aipmtw/space-template` 模板建立：
- **Repo**: https://github.com/aipmtw/space-template
- **內容**: Vite + React 19 + Tailwind CSS 4 + deploy.yml + .gitignore + README.md
- **佔位符**: 所有 `SPACE_NAME` 會在建立時自動替換為實際空間名稱
- **Favicon**: `S` 字母會替換為空間名稱首字母大寫
- **README.md**: H1 標題為空間名稱

## 建立流程

### 自動化（GitHub Actions `create-space.yml`）

管理員留言 `/approve` → Actions 自動執行：

1. 驗證 SHA-256 簽章
2. `gh repo create aipmtw/{name} --template aipmtw/space-template`（從模板建立）
3. Clone 新 repo，將所有 `SPACE_NAME` 替換為實際名稱
4. 替換 favicon 首字母
5. `npm install` 生成 `package-lock.json`
6. `git push` 到新 repo
7. 加入協作者：`gh api repos/aipmtw/{name}/collaborators/{account} -X PUT`
8. 設定 GitHub Pages：`gh api repos/aipmtw/{name}/pages -X POST -f build_type=workflow`
9. 關閉 Issue，留言通知申請人

### 手動建立

```bash
# 1. 從模板建立 repo
gh repo create aipmtw/{name} --public --template aipmtw/space-template

# 2. Clone 並替換名稱
git clone https://github.com/aipmtw/{name}.git
cd {name}
find . -type f -not -path './.git/*' | xargs sed -i 's/SPACE_NAME/{name}/g'

# 3. npm install 並 push
npm install
git add -A && git commit -m "初始建立" && git push

# 4. 加入協作者
gh api repos/aipmtw/{name}/collaborators/{account} -X PUT -f permission=push

# 5. 設定 Pages
gh api repos/aipmtw/{name}/pages -X POST -f build_type=workflow
```

## 建立後自動處理

下次 `/do-issues` 巡檢時會自動：
- 在 aipm.com.tw 首頁新增該成員的卡片
- 將 repo 納入 do-issues / check-actions 巡檢範圍
- 同步卡片資訊（commit 時間、有效提交數）

## 簽章驗證（防竄改）

Issue 內文包含 SHA-256 簽章，防止用戶在提交後修改內容：
- Salt: `aipmtw-2026`
- 格式: `SHA-256(aipmtw-2026|{name}|{account})` 取前 16 碼
- `/approve` 執行前驗證：
  - 簽章存在且正確
  - Issue 標題與內文的名稱/帳號一致
- 驗證失敗 → 拒絕建立，留言說明原因

## 限制

- 每位 GitHub 用戶最多 2 個個人空間
- 空間名稱格式：`[a-z0-9][a-z0-9-]{0,30}[a-z0-9]`（2-32 字元，GitHub 上限 100）
- 不可與現有 repo 或待審批申請重名
- Issue 必須由申請人本人建立（追溯身份）
- Issue 內容不可修改（簽章驗證）
