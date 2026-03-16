# Supabase Auth 認證方案

> 目標：為 aipm.com.tw 平台建立統一的用戶認證系統，支撐私有空間存取控制、管理介面、付費功能等需求。

## 為什麼需要 Auth

| 功能 | 需要 Auth 的原因 |
|------|-----------------|
| 私有空間瀏覽 | 驗證訪客身份，檢查是否有存取權限 |
| 空間管理介面 | 擁有者需登入才能管理授權和設定 |
| 付費功能 | 識別用戶身份，綁定付費狀態 |
| 操作追蹤 | 記錄誰在何時做了什麼操作 |

## Supabase Auth 配置

### 支援的登入方式

| 方式 | 優先級 | 說明 |
|------|--------|------|
| **GitHub OAuth** | 高 | 所有用戶都有 GitHub 帳號，最自然的選擇 |
| **Email + Password** | 中 | 給不想用 GitHub 登入的訪客 |
| **Magic Link** | 低 | 無密碼登入，透過 email 發送一次性連結 |

**建議先只實作 GitHub OAuth**，覆蓋 99% 使用場景。

### GitHub OAuth 設定步驟

1. **GitHub 端**：
   - 前往 GitHub Settings → Developer Settings → OAuth Apps → New
   - Application name: `AIPM TW`
   - Homepage URL: `https://aipm.com.tw`
   - Authorization callback URL: `https://<supabase-project>.supabase.co/auth/v1/callback`
   - 取得 Client ID 和 Client Secret

2. **Supabase 端**：
   - Dashboard → Authentication → Providers → GitHub
   - 填入 Client ID 和 Client Secret
   - 啟用 GitHub provider

3. **前端整合**：

```javascript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

// GitHub 登入
async function signInWithGitHub() {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'github',
    options: {
      redirectTo: 'https://aipm.com.tw/auth/callback'
    }
  })
}

// 登出
async function signOut() {
  await supabase.auth.signOut()
}

// 取得當前用戶
async function getUser() {
  const { data: { user } } = await supabase.auth.getUser()
  return user
}

// 監聽 auth 狀態變化
supabase.auth.onAuthStateChange((event, session) => {
  if (event === 'SIGNED_IN') {
    updateUI(session.user)
  } else if (event === 'SIGNED_OUT') {
    updateUI(null)
  }
})
```

## 用戶資料表

```sql
-- Supabase Auth 自動建立 auth.users 表
-- 我們額外建立 profiles 表存放應用層資料

CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  github_username TEXT,          -- 從 GitHub OAuth 取得
  display_name TEXT,
  avatar_url TEXT,
  plan_type TEXT DEFAULT 'free', -- 'free', 'basic', 'premium'
  plan_expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 自動建立 profile（新用戶註冊時）
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id, github_username, display_name, avatar_url)
  VALUES (
    NEW.id,
    NEW.raw_user_meta_data->>'user_name',
    COALESCE(NEW.raw_user_meta_data->>'full_name', NEW.raw_user_meta_data->>'user_name'),
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

## RLS 政策

```sql
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- 任何人可讀取公開資料
CREATE POLICY "public_read" ON profiles
  FOR SELECT USING (true);

-- 只有自己可修改
CREATE POLICY "self_update" ON profiles
  FOR UPDATE USING (id = auth.uid());
```

## 首頁 UI 整合

### 未登入狀態

```html
<nav>
  <button onclick="signInWithGitHub()" class="...">
    <svg><!-- GitHub icon --></svg>
    用 GitHub 登入
  </button>
</nav>
```

### 已登入狀態

```html
<nav>
  <div class="flex items-center gap-3">
    <img src="{avatar_url}" class="w-8 h-8 rounded-full" />
    <span>{display_name}</span>
    <button onclick="signOut()" class="...">登出</button>
  </div>
</nav>
```

### 管理員額外功能

登入用戶若為 `aipmtw`（管理員），顯示管理面板入口。

## Session 管理

| 設定 | 值 | 說明 |
|------|------|------|
| JWT expiry | 3600 (1h) | Access token 有效期 |
| Refresh token | 啟用 | 自動續期，用戶不需重複登入 |
| Session 儲存 | localStorage | Supabase JS client 預設 |

## 安全考量

1. **ANON KEY 安全**：Supabase anon key 可公開暴露在前端，安全性由 RLS 政策保障
2. **Service Role Key**：僅存於後端/AI 巡檢環境，不可暴露在前端
3. **CORS 設定**：Supabase Dashboard 中設定允許的 origin（`https://aipm.com.tw`）
4. **Rate Limiting**：Supabase Auth 內建防暴力破解機制

## 與其他計畫的關係

```
supabase-auth.md（本文件）
    ↓ 提供認證基礎
aipmtw-dashboard-enhancement.md
    ↓ 提供資料庫基礎
paid-private-space.md
    ↓ 實現私有空間存取控制
```

## 實作步驟

| 步驟 | 工作項目 |
|------|---------|
| 1 | 建立 Supabase 專案（若尚未建立） |
| 2 | 設定 GitHub OAuth provider |
| 3 | 建立 profiles 表 + trigger |
| 4 | 設定 RLS 政策 |
| 5 | 前端加入登入/登出按鈕 |
| 6 | 測試 OAuth 流程（登入 → 建立 profile → 登出） |
| 7 | 整合至首頁 nav |
