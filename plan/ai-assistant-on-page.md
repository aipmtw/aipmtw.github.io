# 首頁 AI 助手：讓用戶透過對話建立 Issues

> 目標：在 aipm.com.tw 首頁嵌入 AI 聊天助手，讓不熟悉 GitHub 的用戶可以用自然語言描述需求，AI 自動轉換為結構化的 GitHub Issue。

## 背景

目前用戶提交需求的方式是點擊「我要下達需求」按鈕，跳轉至 GitHub Issues 頁面手動建立。這對不熟悉 GitHub 的用戶（尤其是非技術背景的會員）是一道障礙。

## 核心需求

1. 首頁嵌入 AI 聊天視窗，用戶用自然語言描述需求
2. AI 理解需求後，自動產生結構化的 Issue（標題 + 描述）
3. 用戶確認後，一鍵提交至對應 repo 的 GitHub Issues

## 架構設計

```
用戶在首頁輸入需求
        ↓
AI 助手理解並整理
        ↓
產生 Issue 預覽（標題 + 描述）
        ↓
用戶確認
        ↓
  ┌─────┴─────┐
  ↓           ↓
已登入       未登入
  ↓           ↓
API 建立     跳轉 GitHub
Issue        預填 Issue 頁面
```

## 技術方案

### 方案 A：Claude API 直接整合（建議）

```
首頁 JS → Supabase Edge Function → Claude API → 回傳結構化 Issue
```

**優點**：品質最高、可自訂 system prompt
**成本**：Claude API 按 token 計費（Haiku 極便宜）

### 方案 B：預設模板 + 表單引導

不用 AI，改用結構化表單引導用戶填寫：
- 選擇需求類型（修改內容/新增功能/修復問題/其他）
- 填寫描述
- 自動產生 Issue

**優點**：零成本
**缺點**：彈性低，用戶仍需分類思考

### 建議：先 B 再 A

先用方案 B 快速上線，收集用戶行為數據。驗證需求後再升級為方案 A。

## 方案 A 詳細設計

### Supabase Edge Function

```typescript
// supabase/functions/ai-create-issue/index.ts
import { serve } from 'https://deno.land/std/http/server.ts'
import Anthropic from '@anthropic-ai/sdk'

const anthropic = new Anthropic({ apiKey: Deno.env.get('CLAUDE_API_KEY') })

serve(async (req) => {
  const { message, repoId, userGithubToken } = await req.json()

  // 用 Claude 解析用戶需求
  const response = await anthropic.messages.create({
    model: 'claude-haiku-4-5-20251001',
    max_tokens: 500,
    system: `你是 AIPMTW 的 AI 助手。用戶會用自然語言描述他們對網站的修改需求。
你的任務是將需求轉換為結構化的 GitHub Issue。
回覆 JSON 格式：{"title": "簡短標題", "body": "詳細描述（繁體中文）"}
標題要簡潔明確，描述要包含具體的修改內容。`,
    messages: [{ role: 'user', content: message }]
  })

  const issue = JSON.parse(response.content[0].text)

  // 如果有 GitHub token，直接建立 Issue
  if (userGithubToken) {
    const ghResponse = await fetch(
      `https://api.github.com/repos/aipmtw/${repoId}/issues`,
      {
        method: 'POST',
        headers: {
          Authorization: `token ${userGithubToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(issue)
      }
    )
    const created = await ghResponse.json()
    return new Response(JSON.stringify({ ...issue, url: created.html_url }))
  }

  // 無 token：回傳預填 URL
  const prefilledUrl = `https://github.com/aipmtw/${repoId}/issues/new?title=${encodeURIComponent(issue.title)}&body=${encodeURIComponent(issue.body)}`
  return new Response(JSON.stringify({ ...issue, prefilledUrl }))
})
```

### 前端聊天 UI

```html
<!-- 浮動聊天按鈕 -->
<button id="ai-chat-btn" class="fixed bottom-6 right-6 z-50 w-14 h-14 rounded-full
  bg-indigo-600 hover:bg-indigo-500 text-white shadow-lg flex items-center justify-center">
  💬
</button>

<!-- 聊天視窗 -->
<div id="ai-chat" class="fixed bottom-24 right-6 z-50 hidden w-80 bg-slate-800
  border border-white/20 rounded-2xl shadow-2xl">
  <div class="p-4 border-b border-white/10">
    <h3 class="font-bold text-white">AI 需求助手</h3>
    <p class="text-xs text-slate-400">描述你想要的修改，AI 幫你建立需求</p>
  </div>
  <div id="chat-messages" class="p-4 h-64 overflow-y-auto space-y-3">
    <div class="text-sm text-slate-300 bg-indigo-500/10 rounded-lg p-3">
      你好！請告訴我你想對網站做什麼修改？
    </div>
  </div>
  <div class="p-4 border-t border-white/10 flex gap-2">
    <input id="chat-input" type="text" placeholder="描述你的需求..."
      class="flex-1 bg-white/5 border border-white/20 rounded-lg px-3 py-2 text-sm text-white" />
    <button id="chat-send" class="bg-indigo-600 px-3 py-2 rounded-lg text-sm">送出</button>
  </div>
</div>
```

### 對話流程

```
用戶：「我想把首頁的背景換成藍色」
AI：我幫你整理了這個需求：

📝 標題：更換首頁背景顏色為藍色
📋 描述：將首頁的背景顏色從目前的漸層色調改為藍色系。

要提交到哪個空間？
[sherry] [mark] [robert] ...

用戶：點擊 [sherry]
AI：已建立 Issue！#5 更換首頁背景顏色為藍色
巡檢系統將在 15 分鐘內自動處理。
```

## 方案 B 詳細設計（快速上線版）

### 結構化需求表單

```html
<div class="space-y-4">
  <!-- 選擇空間 -->
  <select id="req-space">
    <option>選擇你的空間</option>
    <option value="sherry">Sherry</option>
    <option value="mark">Mark</option>
    ...
  </select>

  <!-- 需求類型 -->
  <select id="req-type">
    <option value="modify">修改內容</option>
    <option value="feature">新增功能</option>
    <option value="fix">修復問題</option>
    <option value="other">其他</option>
  </select>

  <!-- 描述 -->
  <textarea id="req-desc" placeholder="詳細描述你的需求..."></textarea>

  <!-- 提交（跳轉至 GitHub Issue 預填頁面） -->
  <button onclick="submitReq()">提交需求</button>
</div>
```

## 實作階段

| 階段 | 工作 | 前置條件 |
|------|------|---------|
| 1 | 方案 B：結構化表單上線 | 無 |
| 2 | Supabase Auth 整合（知道用戶是誰） | supabase-auth.md |
| 3 | 方案 A：Claude API Edge Function | Supabase 專案 |
| 4 | 聊天 UI + 對話流程 | 階段 3 |
| 5 | 已登入用戶免跳轉直接建 Issue | 階段 2 + GitHub OAuth |

## 成本估算

| 項目 | 月費 |
|------|------|
| Claude Haiku API（1000 次/月） | ~$0.5 USD |
| Supabase Edge Function | 免費（500K 次/月） |
| 合計 | ~$0.5 USD/月 |
