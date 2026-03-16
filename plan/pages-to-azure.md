# 從 GitHub Pages 遷移至 Azure Web App 計畫

> 目標：將所有成員網站從 GitHub Pages 遷移至 Azure Web App，統一管理在子目錄下，透過 Supabase Auth 實現付費用戶的私有空間存取控制。

## 背景與動機

### 現況

```
aipmtw.github.io          → https://aipm.com.tw/
aipmtw.github.io/sherry   → https://aipm.com.tw/sherry/
aipmtw.github.io/mark     → https://aipm.com.tw/mark/
...（每個成員空間是獨立 repo，透過 GitHub Pages 部署）
```

### 問題

1. **Private repo 無法用 GitHub Pages**：付費用戶想隱藏 repo，但 GitHub Pages 僅支援 public repo（免費方案）
2. **存取控制不可能**：GitHub Pages 是純靜態，無法做 Auth middleware
3. **分散管理**：15+ 個 repo 各自獨立部署，無統一入口控制
4. **成本考量**：GitHub Pages 免費但功能受限；Azure Web App 免費方案足以支撐初期

### 目標架構

```
Azure Web App (aipm.com.tw)
├── /                    ← 首頁（靜態，公開）
├── /sherry/             ← 成員空間（公開）
├── /mark/               ← 成員空間（公開）
├── /robert/             ← 成員空間（付費私有，需登入）
├── /swim-meet/          ← 成員空間（公開）
├── /auth/login          ← Supabase Auth 登入頁
└── /api/check-access    ← 存取控制 API
```

## 架構設計

### 整體架構

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│   Browser   │────▶│  Azure Web App   │────▶│   Supabase   │
│             │     │  (Node.js/Express)│     │  Auth + DB   │
└─────────────┘     │                  │     └──────────────┘
                    │  Middleware:      │
                    │  ① 靜態檔案服務  │     ┌──────────────┐
                    │  ② Auth 檢查     │────▶│  GitHub API  │
                    │  ③ 路由分發      │     │  (取原始碼)   │
                    └──────────────────┘     └──────────────┘
```

### 請求處理流程

```
訪客請求 https://aipm.com.tw/robert/
        ↓
Azure Web App Express Middleware
        ↓
查詢 Supabase: repos 表 → robert 是否為 private?
        ↓
  ┌─────┴─────┐
  ↓           ↓
 公開空間    私有空間
  ↓           ↓
直接回傳     檢查 Supabase Auth session
靜態檔案           ↓
           ┌──────┴──────┐
           ↓             ↓
        有授權         無授權
           ↓             ↓
        回傳檔案     redirect /auth/login
```

### 為什麼選 Azure 而非 Vercel/Cloudflare

| 面向 | Azure Web App | Vercel | Cloudflare Pages |
|------|--------------|--------|-----------------|
| **免費方案** | F1: 1GB RAM, 60 min/day CPU | Hobby: 100GB bandwidth | 無限靜態 |
| **自訂 middleware** | 完整 Node.js 環境 | Edge middleware 限制 | Workers 限制 |
| **子目錄路由** | 完全控制 | 需多專案或 rewrites | 需 Workers |
| **台灣機房** | 東亞（日本/香港） | 全球 CDN | 全球 CDN |
| **擴展性** | 可升級至 B1/S1 | 需 Pro 方案 | 需 Workers Paid |
| **整合性** | GitHub Actions 直接部署 | Git 整合 | Git 整合 |

**選 Azure 的理由**：
1. 完整 Node.js 環境，可做任何 middleware 邏輯
2. 所有成員空間在同一個 Web App 下，統一管理
3. 免費方案足以支撐初期（< 100 位用戶）
4. 未來可平滑升級至付費方案

## 技術實作

### 1. Azure Web App 設定

```bash
# 建立 Resource Group
az group create --name aipmtw-rg --location eastasia

# 建立 App Service Plan (免費)
az appservice plan create --name aipmtw-plan --resource-group aipmtw-rg --sku F1 --is-linux

# 建立 Web App (Node.js 20)
az webapp create --name aipmtw --resource-group aipmtw-rg --plan aipmtw-plan --runtime "NODE:20-lts"

# 設定自訂域名
az webapp config hostname add --webapp-name aipmtw --resource-group aipmtw-rg --hostname aipm.com.tw
```

### 2. Express 應用程式架構

```
azure-app/
├── server.js                 ← Express 入口
├── middleware/
│   ├── auth.js              ← Supabase Auth 中間層
│   └── access-control.js   ← 存取控制邏輯
├── public/
│   ├── index.html           ← 首頁
│   ├── auth/
│   │   └── login.html       ← 登入頁面
│   └── spaces/
│       ├── sherry/          ← 成員空間靜態檔案
│       ├── mark/
│       └── robert/
├── scripts/
│   └── sync-spaces.js      ← 從 GitHub 同步成員空間
├── package.json
└── .env
```

### 3. Express Server

```javascript
// server.js
const express = require('express')
const { createClient } = require('@supabase/supabase-js')
const cookieParser = require('cookie-parser')
const path = require('path')

const app = express()
app.use(cookieParser())

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SERVICE_KEY
)

// 首頁（公開）
app.use('/', express.static(path.join(__dirname, 'public')))

// 成員空間路由
app.use('/:space/*', async (req, res, next) => {
  const spaceName = req.params.space

  // 查詢 repo 是否為私有
  const { data: repo } = await supabase
    .from('repos')
    .select('is_private, id')
    .eq('id', spaceName)
    .single()

  if (!repo) return res.status(404).send('空間不存在')

  // 公開空間：直接服務靜態檔案
  if (!repo.is_private) {
    return express.static(
      path.join(__dirname, 'public', 'spaces', spaceName)
    )(req, res, next)
  }

  // 私有空間：檢查 Auth
  const token = req.cookies['sb-access-token']
  if (!token) {
    return res.redirect('/auth/login?redirect=' + req.originalUrl)
  }

  // 驗證 token
  const { data: { user } } = await supabase.auth.getUser(token)
  if (!user) {
    return res.redirect('/auth/login?redirect=' + req.originalUrl)
  }

  // 檢查存取權限
  const { data: access } = await supabase
    .from('space_access')
    .select('id')
    .eq('repo_id', spaceName)
    .eq('granted_to', user.id)
    .single()

  if (!access) {
    return res.status(403).send(`
      <h1>此空間為私有</h1>
      <p>您沒有瀏覽權限。請聯繫空間擁有者取得授權。</p>
      <a href="/">返回首頁</a>
    `)
  }

  // 有權限：服務靜態檔案
  express.static(
    path.join(__dirname, 'public', 'spaces', spaceName)
  )(req, res, next)
})

app.listen(process.env.PORT || 3000)
```

### 4. 空間同步腳本

從各 GitHub repo 同步 build 產出至 Azure：

```javascript
// scripts/sync-spaces.js
const { execSync } = require('child_process')
const fs = require('fs')
const path = require('path')

const SPACES_DIR = path.join(__dirname, '..', 'public', 'spaces')

async function syncSpace(repoName) {
  const spaceDir = path.join(SPACES_DIR, repoName)

  // Clone 或 pull
  if (!fs.existsSync(spaceDir)) {
    execSync(`git clone https://github.com/aipmtw/${repoName}.git /tmp/${repoName}`)
  } else {
    execSync(`cd /tmp/${repoName} && git pull`)
  }

  // Build
  execSync(`cd /tmp/${repoName} && npm install && npm run build`)

  // 複製 build 產出到 spaces 目錄
  const buildDir = path.join('/tmp', repoName, 'dist')
  execSync(`cp -r ${buildDir}/* ${spaceDir}/`)
}

// 同步所有空間
const repos = ['sherry', 'mark', 'felix', 'amy', 'robert', /* ... */]
repos.forEach(syncSpace)
```

### 5. GitHub Actions 部署流程

```yaml
# .github/workflows/deploy-azure.yml
name: Deploy to Azure
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install & Build
        run: npm install

      - name: Sync all spaces
        run: node scripts/sync-spaces.js
        env:
          GITHUB_TOKEN: ${{ secrets.ORG_ADMIN_TOKEN }}

      - name: Deploy to Azure
        uses: azure/webapps-deploy@v3
        with:
          app-name: aipmtw
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
```

### 6. 成員空間更新觸發

當成員 repo 有新的 push 時，觸發 Azure 重新部署該空間：

```yaml
# 各成員 repo 的 .github/workflows/notify-azure.yml
name: Notify Azure
on:
  push:
    branches: [main]
jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Azure redeploy
        run: |
          curl -X POST "${{ secrets.AZURE_WEBHOOK_URL }}" \
            -H "Content-Type: application/json" \
            -d '{"repo": "${{ github.repository }}"}'
```

## Supabase 整合

### 資料表（延伸現有設計）

```sql
-- repos 表新增欄位
ALTER TABLE repos ADD COLUMN is_private BOOLEAN DEFAULT false;
ALTER TABLE repos ADD COLUMN owner_user_id UUID REFERENCES auth.users(id);

-- 存取控制表（同 paid-private-space.md）
-- space_access, share_links 表結構不變
```

### 登入頁面

```html
<!-- public/auth/login.html -->
<script src="https://unpkg.com/@supabase/supabase-js@2"></script>
<script>
  const supabase = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

  async function loginWithGitHub() {
    await supabase.auth.signInWithOAuth({
      provider: 'github',
      options: { redirectTo: new URLSearchParams(location.search).get('redirect') || '/' }
    })
  }
</script>
```

## 遷移計畫

### 階段一：準備（不影響現有服務）

1. 建立 Azure Web App 和 Express 應用
2. 設定 Supabase 專案（Auth + 資料表）
3. 建立空間同步腳本
4. 在測試域名（如 aipmtw.azurewebsites.net）驗證

### 階段二：平行運行

1. 部署 Azure Web App，使用測試域名
2. 確認所有成員空間在 Azure 上正常運作
3. 測試 Auth 登入和存取控制流程
4. GitHub Pages 繼續正常服務

### 階段三：DNS 切換

1. 將 aipm.com.tw 的 DNS 從 GitHub Pages 切換至 Azure
2. 設定 Azure 的 SSL 憑證（免費 App Service Managed Certificate）
3. 監控 24 小時確認穩定

### 階段四：清理

1. 各成員 repo 的 GitHub Pages 設定可保留（作為備援）
2. 更新巡檢 SOP：`/do-issues` 和 `/check-actions` 適配新架構
3. 更新首頁 index.html 的部署方式

## 遷移前後對比

| 面向 | GitHub Pages (現況) | Azure Web App (目標) |
|------|-------------------|---------------------|
| 部署方式 | 每個 repo 獨立 Pages | 統一 Web App，子目錄 |
| 私有空間 | 不支援 | Supabase Auth 控制 |
| 存取控制 | 無 | Middleware 層級 |
| 自訂域名 | 需個別設定 | 統一管理 |
| 成本 | 免費 | 免費（F1 方案） |
| 效能 | GitHub CDN | Azure CDN（可選） |
| 擴展性 | 受限 | 可升級方案 |
| 巡檢整合 | 需改寫 HTML | API 寫入 Supabase |

## 風險與緩解

| 風險 | 緩解策略 |
|------|---------|
| Azure 免費方案效能不足 | F1 有 60 min CPU/day 限制，初期足夠；成長後升級 B1（~NT$400/月） |
| DNS 切換中斷 | 設定低 TTL，切換時選擇低流量時段 |
| 空間同步失敗 | 保留 GitHub Pages 作為備援，同步腳本加入錯誤通知 |
| Supabase 免費方案限制 | 50K MAU 足夠初期；可升級 Pro（$25/月） |

## 與現有計畫的關係

```
supabase-auth.md          ← Auth 認證基礎（直接採用）
aipmtw-dashboard-enhancement.md ← 資料庫設計（直接採用）
paid-private-space.md     ← 私有空間需求（本計畫取代 Vercel 方案，改用 Azure）
aipmtw-business-model.md  ← 商業模式（付費功能依賴本計畫）
```

**注意**：本計畫取代 `paid-private-space.md` 中的 Vercel 方案。Azure Web App 提供更完整的控制能力，且所有成員空間集中管理，更適合長期運營。
