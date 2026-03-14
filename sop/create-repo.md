# SOP：建立個人空間（Create Repo）

## 觸發方式

### 方式一：透過首頁申請表單（自助）

1. 申請人在 https://aipm.com.tw/ 底部填寫：
   - **空間名稱**（小寫英文+數字+連字號，2-30 字元）
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

## 建立流程

### 自動化（GitHub Actions `create-space.yml`）

管理員留言 `/approve` → Actions 自動執行：

1. 解析 Issue 標題取得空間名稱和 GitHub Account
2. `gh repo create aipmtw/{name} --public`
3. 初始化 Vite + React + Tailwind CSS 專案
   - `vite.config.js` 設定 `base: '/{name}/'`
   - 首頁顯示空間名稱 + 返回 aipm.com.tw 連結
   - SVG favicon（首字母）
   - `.github/workflows/deploy.yml`
4. `npm install` 生成 `package-lock.json`
5. `git push` 到新 repo
6. 加入協作者：`gh api repos/aipmtw/{name}/collaborators/{account} -X PUT`
7. 設定 GitHub Pages：`gh api repos/aipmtw/{name}/pages -X POST -f build_type=workflow`
8. 關閉 Issue，留言通知申請人

### 手動建立

```bash
# 1. 建立 repo
gh repo create aipmtw/{name} --public

# 2. 複製模板、修改名稱
cp -r template/ {name}/
cd {name}
# 修改 package.json, vite.config.js, index.html, App.jsx 中的名稱

# 3. npm install 並 push
npm install
git init && git add -A && git commit -m "初始建立" && git push

# 4. 加入協作者
gh api repos/aipmtw/{name}/collaborators/{account} -X PUT -f permission=push

# 5. 設定 Pages
gh api repos/aipmtw/{name}/pages -X POST -f build_type=workflow
```

## 建立後自動處理

下次 `/check-issues` 巡檢時會自動：
- 在 aipm.com.tw 首頁新增該成員的卡片
- 將 repo 納入 check-issues / check-actions 巡檢範圍
- 同步卡片資訊（commit 時間、有效提交數）

## 限制

- 每位 GitHub 用戶最多 2 個個人空間
- 空間名稱格式：`[a-z0-9][a-z0-9-]{0,28}[a-z0-9]`
- 不可與現有 repo 或待審批申請重名
