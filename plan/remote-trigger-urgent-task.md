# 遠端觸發緊急任務計畫

> 目標：讓管理員可從任何地方透過安全的網路方式，觸發家中 Claude Code 終端機執行緊急任務。

## 背景

Claude Code 終端機在家中電腦上運行，排程每 15 分鐘巡檢一次。但有時需要在外出時立即觸發任務（例如緊急修復、核准空間申請），不想等待下一次排程。

## 需求

1. 從手機或任何有網路的地方觸發家中 Claude Code
2. 安全性：只有授權管理員可觸發
3. 可指定要執行的命令（如 `/do-issues`、`/check-actions`、自訂 prompt）
4. 能看到執行結果

## 方案比較

| 方案 | 安全性 | 複雜度 | 成本 | 即時性 |
|------|--------|--------|------|--------|
| **A. GitHub Issues 觸發** | 高 | 低 | 免費 | ~15 分鐘 |
| **B. Supabase Realtime 監聽** | 高 | 中 | 免費 | 即時 |
| **C. Cloudflare Tunnel** | 高 | 中 | 免費 | 即時 |
| **D. Tailscale VPN** | 高 | 低 | 免費 | 即時 |
| **E. GitHub Actions + Self-hosted Runner** | 高 | 高 | 免費 | ~30 秒 |

---

### 方案 A：GitHub Issues 觸發（最簡單）

**原理**：在特定 repo 建立 issue，巡檢系統下次執行時偵測並處理。

```
管理員在手機上 → 開 GitHub Issue → 巡檢系統偵測 → 執行任務
```

**優點**：零額外設定，現有巡檢系統已支援
**缺點**：需等待下次巡檢（最多 15 分鐘）

**改進**：將巡檢頻率提高至每 5 分鐘，或加入「緊急標籤」偵測邏輯。

---

### 方案 B：Supabase Realtime 監聽（建議）

**原理**：家中 Claude Code 持續監聽 Supabase 資料表，管理員透過 API 或網頁寫入任務。

```
管理員（手機/網頁）
        ↓ 寫入任務到 Supabase
Supabase DB (tasks 表)
        ↓ Realtime 推送
家中監聽腳本
        ↓ 收到任務
執行 Claude Code
        ↓ 寫回結果
管理員看到結果
```

#### 資料表設計

```sql
CREATE TABLE remote_tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  command TEXT NOT NULL,           -- 要執行的命令，如 '/do-issues'
  status TEXT DEFAULT 'pending',   -- 'pending', 'running', 'completed', 'failed'
  created_by UUID REFERENCES auth.users(id),
  result TEXT,                     -- 執行結果摘要
  created_at TIMESTAMPTZ DEFAULT NOW(),
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ
);

-- RLS：僅管理員可寫入，認證用戶可讀取自己的任務
ALTER TABLE remote_tasks ENABLE ROW LEVEL SECURITY;
CREATE POLICY "admin_all" ON remote_tasks FOR ALL USING (
  auth.uid() IN (SELECT id FROM profiles WHERE github_username = 'aipmtw')
);
```

#### 監聽腳本（家中電腦）

```javascript
// listener.js — 在家中電腦背景執行
const { createClient } = require('@supabase/supabase-js')
const { execSync } = require('child_process')

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY)

// 監聽新任務
supabase
  .channel('remote-tasks')
  .on('postgres_changes',
    { event: 'INSERT', schema: 'public', table: 'remote_tasks' },
    async (payload) => {
      const task = payload.new
      console.log(`收到任務: ${task.command}`)

      // 更新狀態為 running
      await supabase.from('remote_tasks')
        .update({ status: 'running', started_at: new Date().toISOString() })
        .eq('id', task.id)

      try {
        // 執行 Claude Code
        const result = execSync(
          `cd ~/2026aipm-projects/2026biz && claude -p "${task.command}" --dangerously-skip-permissions`,
          { timeout: 300000, encoding: 'utf-8' }
        )

        await supabase.from('remote_tasks')
          .update({
            status: 'completed',
            result: result.slice(-500), // 保留最後 500 字元
            completed_at: new Date().toISOString()
          })
          .eq('id', task.id)
      } catch (err) {
        await supabase.from('remote_tasks')
          .update({
            status: 'failed',
            result: err.message.slice(-500),
            completed_at: new Date().toISOString()
          })
          .eq('id', task.id)
      }
    }
  )
  .subscribe()

console.log('監聽中... Ctrl+C 結束')
```

#### 管理員觸發介面

可在首頁管理員區域加入簡單的觸發面板：

```html
<div class="admin-panel">
  <h3>遠端任務</h3>
  <div class="flex gap-2">
    <button onclick="triggerTask('/do-issues')">巡檢 Issues</button>
    <button onclick="triggerTask('/check-actions')">巡檢 Actions</button>
  </div>
  <input type="text" id="custom-cmd" placeholder="自訂命令..." />
  <button onclick="triggerTask(document.getElementById('custom-cmd').value)">執行</button>

  <div id="task-status">
    <!-- 即時顯示任務狀態 -->
  </div>
</div>

<script>
async function triggerTask(command) {
  const { data } = await supabase.from('remote_tasks').insert({ command })
  // 監聽結果更新...
}
</script>
```

---

### 方案 C：Cloudflare Tunnel（進階）

**原理**：Cloudflare Tunnel 將家中電腦的服務安全暴露到網路。

```bash
# 安裝 cloudflared
# 建立 tunnel
cloudflared tunnel create aipmtw-home

# 設定 config.yml
tunnel: <TUNNEL_ID>
credentials-file: ~/.cloudflared/<TUNNEL_ID>.json
ingress:
  - hostname: trigger.aipm.com.tw
    service: http://localhost:8080
  - service: http_status:404

# 啟動
cloudflared tunnel run aipmtw-home
```

家中電腦啟動一個簡單的 HTTP 服務，接收 webhook：

```javascript
const express = require('express')
const { execSync } = require('child_process')

const app = express()
app.use(express.json())

// 驗證 bearer token
app.use((req, res, next) => {
  if (req.headers.authorization !== `Bearer ${process.env.TRIGGER_SECRET}`) {
    return res.status(401).json({ error: 'Unauthorized' })
  }
  next()
})

app.post('/trigger', (req, res) => {
  const { command } = req.body
  // 非同步執行，立即回應
  exec(`cd ~/2026aipm-projects/2026biz && claude -p "${command}" --dangerously-skip-permissions`)
  res.json({ status: 'triggered', command })
})

app.listen(8080)
```

---

### 方案 D：Tailscale VPN（最安全）

**原理**：用 Tailscale 建立私有網路，從手機直接 SSH 到家中電腦。

```bash
# 家中電腦 + 手機都安裝 Tailscale
# 從手機 SSH
ssh user@home-pc.tailnet-name.ts.net

# 執行命令
cd ~/2026aipm-projects/2026biz && claude -p "/do-issues" --dangerously-skip-permissions
```

**優點**：最安全（端對端加密、無暴露任何端口）
**缺點**：需安裝 Tailscale、手機操作不如網頁方便

---

## 建議實施順序

| 優先 | 方案 | 理由 |
|------|------|------|
| 1 | **方案 A** | 零成本零設定，立即可用，解決 90% 場景 |
| 2 | **方案 B** | Supabase 已在規劃中，順帶實作即時觸發 |
| 3 | **方案 D** | 安裝 Tailscale 即可，適合需要完整 SSH 控制的場景 |
| 4 | **方案 C** | 需要對外暴露服務時才考慮 |

## 方案 A 快速實作

最快的方式：在 aipmtw.github.io repo 建立帶有 `urgent` 標籤的 issue，巡檢系統偵測到後優先處理。

```bash
# 從手機用 GitHub App 建立 issue：
# 標題：[urgent] 執行 /do-issues
# 標籤：urgent

# 或用 gh CLI（如果手機有安裝）：
gh issue create -R aipmtw/aipmtw.github.io -t "[urgent] /do-issues" -l urgent
```

修改 `/do-issues` SOP，加入 `urgent` 標籤偵測：在排程的空檔也輪詢 urgent issues。

## 安全考量

| 面向 | 措施 |
|------|------|
| 認證 | Supabase Auth（方案 B）/ Bearer Token（方案 C）/ Tailscale（方案 D） |
| 授權 | 僅 aipmtw 管理員可觸發 |
| 命令注入 | 白名單限制可執行的命令（僅 `/do-issues`、`/check-actions` 等已知指令） |
| 日誌 | 所有觸發記錄寫入 Supabase，可追溯 |
| 速率限制 | 每分鐘最多 1 次觸發，防止誤操作 |
