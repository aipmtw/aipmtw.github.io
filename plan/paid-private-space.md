# 付費用戶私有空間計畫

> 目標：讓付費用戶可將 repo 設為非公開，網站僅對指定用戶開放瀏覽，透過 Supabase Auth 實現存取控制。

## 背景

目前所有成員空間都是 public repo + 公開網頁，任何人都可瀏覽。付費用戶有需求將作品設為私有，僅分享給指定的人。

## 核心需求

1. **隱藏 Repo**：付費用戶可選擇將 repo 從 public 轉為 private，首頁卡片標示為「私有」
2. **授權分享**：私有空間的網站僅對被授權的用戶可見，透過 Supabase Auth 登入驗證
3. **分享管理**：空間擁有者可管理誰有權瀏覽自己的網站

## 架構設計

```
訪客瀏覽私有空間 URL
        ↓
Supabase Auth 登入頁面
        ↓
  ┌─────┴─────┐
  ↓           ↓
已授權用戶    未授權用戶
  ↓           ↓
顯示網站     顯示「無權限」
```

## 技術方案

### 部署方式變更

Private repo 無法使用 GitHub Pages。需改用其他方案：

| 方案 | 說明 | 優缺點 |
|------|------|--------|
| **A. Cloudflare Pages + Access** | 從 private repo 部署到 Cloudflare Pages，用 Access 控制 | 簡單但需 Cloudflare 帳號 |
| **B. Vercel + Middleware** | 部署到 Vercel，middleware 檢查 Supabase session | 免費方案足夠，整合性好 |
| **C. Supabase Edge Functions + Storage** | 靜態檔案存 Supabase Storage，Edge Function 做 auth 中間層 | 全 Supabase 方案，但較複雜 |

**建議方案：B（Vercel + Supabase Auth Middleware）**

理由：
- Vercel 支援從 private repo 部署
- Middleware 可在 edge 層面檢查 Supabase JWT
- 免費方案足以支撐小規模使用
- 與 Supabase Dashboard Enhancement 計畫相容

### 存取控制流程

```
1. 訪客請求 private-space.vercel.app
2. Vercel Middleware 檢查 Supabase session cookie
3. 無 session → redirect 至登入頁面
4. 有 session → 查詢 Supabase 授權表
5. 有權限 → 放行，顯示網站
6. 無權限 → 顯示「此空間為私有，您無瀏覽權限」
```

### 資料表設計

新增至 Supabase（延伸 `aipmtw-dashboard-enhancement.md` 的資料表）：

```sql
-- 擴展 repos 表
ALTER TABLE repos ADD COLUMN is_private BOOLEAN DEFAULT false;
ALTER TABLE repos ADD COLUMN owner_id UUID REFERENCES auth.users(id);
ALTER TABLE repos ADD COLUMN plan_type TEXT DEFAULT 'free'; -- 'free', 'paid'

-- 空間存取授權
CREATE TABLE space_access (
  id SERIAL PRIMARY KEY,
  repo_id TEXT REFERENCES repos(id) ON DELETE CASCADE,
  granted_to UUID REFERENCES auth.users(id),  -- 被授權的用戶
  granted_by UUID REFERENCES auth.users(id),  -- 授權者（空間擁有者）
  access_type TEXT DEFAULT 'view',             -- 'view', 'edit'
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ,                      -- 可設定到期時間
  UNIQUE(repo_id, granted_to)
);

-- 分享連結（可選，用於產生一次性或限時分享連結）
CREATE TABLE share_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id TEXT REFERENCES repos(id) ON DELETE CASCADE,
  created_by UUID REFERENCES auth.users(id),
  max_uses INT,                    -- 最大使用次數，NULL = 無限
  used_count INT DEFAULT 0,
  expires_at TIMESTAMPTZ,          -- 到期時間
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### RLS 政策

```sql
-- space_access: 擁有者可管理，被授權者可讀取自己的授權
ALTER TABLE space_access ENABLE ROW LEVEL SECURITY;

CREATE POLICY "owner_manage" ON space_access
  FOR ALL USING (
    granted_by = auth.uid()
  );

CREATE POLICY "grantee_read" ON space_access
  FOR SELECT USING (
    granted_to = auth.uid()
  );

-- share_links: 擁有者可管理
ALTER TABLE share_links ENABLE ROW LEVEL SECURITY;

CREATE POLICY "creator_manage" ON share_links
  FOR ALL USING (
    created_by = auth.uid()
  );
```

### Vercel Middleware（私有空間驗證）

```typescript
// middleware.ts
import { createClient } from '@supabase/supabase-js'
import { NextResponse } from 'next/server'

export async function middleware(request) {
  const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

  // 從 cookie 取得 session
  const session = await supabase.auth.getSession()

  if (!session.data.session) {
    // 未登入 → 重導至登入頁
    return NextResponse.redirect('/auth/login?redirect=' + request.url)
  }

  const userId = session.data.session.user.id
  const repoId = extractRepoFromUrl(request.url)

  // 檢查是否有存取權限
  const { data } = await supabase
    .from('space_access')
    .select('id')
    .eq('repo_id', repoId)
    .eq('granted_to', userId)
    .single()

  if (!data) {
    return new NextResponse('此空間為私有，您無瀏覽權限', { status: 403 })
  }

  return NextResponse.next()
}
```

## 首頁整合

### 卡片顯示變更

```html
<!-- 私有空間卡片 -->
<div class="card-hover ... border-amber-500/30">
  <div class="h-1.5 bg-gradient-to-r from-amber-500 to-orange-500"></div>
  <div class="p-6">
    <span class="text-xs bg-amber-500/20 text-amber-300 px-2 py-0.5 rounded">
      🔒 私有空間
    </span>
    ...
    <!-- 瀏覽按鈕改為需要登入提示 -->
    <a href="..." class="...">
      🔐 登入後瀏覽
    </a>
  </div>
</div>
```

### 管理介面

空間擁有者登入後可見的管理面板：

- 查看已授權的用戶清單
- 新增/移除授權用戶（輸入 email 或 GitHub 帳號）
- 產生分享連結（可設定到期時間和使用次數）
- 切換公開/私有狀態

## 付費方案設計（建議）

| 方案 | 價格 | 功能 |
|------|------|------|
| 免費 | $0 | 最多 2 個公開空間 |
| 基本 | $X/月 | 最多 5 個空間（含私有），最多分享給 10 人 |
| 進階 | $Y/月 | 無限空間，無限分享，自訂域名 |

付費狀態儲存在 Supabase `profiles` 表中，由 Stripe/其他支付系統 webhook 更新。

## 實作階段

| 階段 | 工作項目 | 前置條件 |
|------|---------|---------|
| 1 | Supabase Auth 設定（見 `supabase-auth.md`） | 無 |
| 2 | 資料表建立 + RLS 設定 | 階段 1 |
| 3 | 首頁登入/註冊整合 | 階段 1 |
| 4 | Vercel 部署 + Middleware | 階段 2 |
| 5 | 空間管理介面 | 階段 3 |
| 6 | 分享連結功能 | 階段 2 |
| 7 | 付費整合（Stripe） | 階段 5 |

## 與現有系統的相容性

- **公開空間不受影響**：免費用戶的公開空間繼續使用 GitHub Pages
- **巡檢系統調整**：`/check-issues` 需識別 private repo（目前只掃 public），付費用戶的 private repo 也要巡檢
- **首頁卡片**：private repo 仍顯示卡片，但標示 🔒 並導向 Vercel URL 而非 GitHub Pages
