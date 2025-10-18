下面給你一份「可直接貼進專案與筆記」的**完整講義**：前端（JavaScript Streams API）＋後端（ASP.NET Core `IAsyncEnumerable<T>`）一次到位，含高效、安全、可維運的實作與部署要點。你可以直接把整份複製成 `.md` 存到你的 IT-Notes / GitHub。

---

# JavaScript Streams API × ASP.NET Core IAsyncEnumerable 串流講義（安全高效完整範例）

> 目標：用 **前端 JS Streams API**（`ReadableStream`／`TextDecoder`／`TransformStream`）搭配 **後端 ASP.NET Core + IAsyncEnumerable<T>**，實作「即時分段（chunked）」的安全高效串流 API 與前端 UI。
> 內含：完整 Server/Client 程式、錯誤處理、取消、Backpressure、NDJSON/SSE 格式、Nginx/IIS/Cloudflare 部署重點。

---

## 1) 串流核心與資料格式

* **HTTP 規格**：只有一筆 HTTP Response，但 **Body 以 chunked 方式分段送**。
* **Server**：每次 `yield return`（或寫入 `Response.BodyWriter`）就立即送出一個 chunk。
* **Client**：用 `response.body.getReader()` 逐塊讀取、邊收邊顯示，不用等整包。

### 推薦兩種資料格式

1. **NDJSON**（每行一筆 JSON）

```
{"type":"token","data":"第一段"}
{"type":"token","data":"第二段"}
{"type":"done"}
```

2. **SSE（Server-Sent Events）**

```
event: token
data: 第一段

event: token
data: 第二段

event: done
data: {}
```

> 二選一即可。NDJSON 易於 POST 請求、雙向協議外掛擴充；SSE 有既有事件語法（GET 最方便）。

---

## 2) 後端：ASP.NET Core 串流（.NET 8/9 Minimal API）

### 2.1 Program.cs（NDJSON 與 SSE 兩路並存）

```csharp
using System.Runtime.CompilerServices;

var builder = WebApplication.CreateBuilder(args);

// CORS（如前後端不同網域）
builder.Services.AddCors(opt =>
{
    opt.AddDefaultPolicy(policy => policy
        .AllowAnyHeader()
        .AllowAnyMethod()
        .WithOrigins("https://your-frontend.example.com") // TODO: 改成你的網域
        .AllowCredentials());
});

var app = builder.Build();
app.UseCors();

app.MapGet("/healthz", () => Results.Ok(new { ok = true }));

// ===== NDJSON：每次 yield = 一行 JSON + \n =====
app.MapPost("/api/stream/ndjson", (HttpContext http, CancellationToken ct) =>
{
    http.Response.Headers.CacheControl = "no-cache";
    http.Response.Headers.Pragma = "no-cache";
    http.Response.Headers["X-Accel-Buffering"] = "no"; // Nginx 關閉緩衝
    http.Response.ContentType = "application/x-ndjson; charset=utf-8";
    return StreamNdjson(ct);
});

// ===== SSE：每次 yield = 一個 event（以空行分隔）=====
app.MapPost("/api/stream/sse", (HttpContext http, CancellationToken ct) =>
{
    http.Response.Headers.CacheControl = "no-cache";
    http.Response.Headers.Pragma = "no-cache";
    http.Response.Headers["X-Accel-Buffering"] = "no";
    http.Response.Headers.Connection = "keep-alive";
    http.Response.ContentType = "text/event-stream; charset=utf-8";
    return StreamSse(ct);
});

app.Run();

static async IAsyncEnumerable<string> StreamNdjson([EnumeratorCancellation] CancellationToken ct)
{
    for (int i = 0; i < 5; i++)
    {
        ct.ThrowIfCancellationRequested();
        var jsonLine = $"{{\"type\":\"token\",\"data\":\"第{i + 1}段\"}}\n";
        yield return jsonLine;
        await Task.Delay(400, ct);
    }
    yield return "{\"type\":\"done\"}\n";
}

static async IAsyncEnumerable<string> StreamSse([EnumeratorCancellation] CancellationToken ct)
{
    for (int i = 0; i < 5; i++)
    {
        ct.ThrowIfCancellationRequested();
        var sseEvent =
            $"event: token\n" +
            $"data: 第{i + 1}段\n\n";
        yield return sseEvent;
        await Task.Delay(400, ct);
    }
    yield return "event: done\ndata: {}\n\n";
}
```

#### 重點

* `IAsyncEnumerable<string>` 讓 ASP.NET Core 自動以 **chunked** 寫出。
* `[EnumeratorCancellation]` 可讓前端 Abort 直接中斷迭代。
* `X-Accel-Buffering: no` 關閉 Nginx 緩衝；IIS/Cloudflare 同理需關閉 buffer（見 §5）。
* 需要驗證時，請加上 `UseAuthentication/UseAuthorization` 並檢查 Token。

### 2.2 進階：更細控 Flush（`Response.BodyWriter`）

```csharp
app.MapPost("/api/stream/lowlevel", async (HttpContext http, CancellationToken ct) =>
{
    http.Response.Headers.CacheControl = "no-cache";
    http.Response.Headers.Pragma = "no-cache";
    http.Response.Headers["X-Accel-Buffering"] = "no";
    http.Response.ContentType = "application/x-ndjson; charset=utf-8";

    var writer = http.Response.BodyWriter;
    for (int i = 0; i < 5; i++)
    {
        ct.ThrowIfCancellationRequested();
        var payload = $"{{\"type\":\"token\",\"data\":\"低階{i + 1}\"}}\n";
        await writer.WriteAsync(System.Text.Encoding.UTF8.GetBytes(payload), ct);
        await http.Response.BodyWriter.FlushAsync(ct);
        await Task.Delay(400, ct);
    }
    await writer.WriteAsync(System.Text.Encoding.UTF8.GetBytes("{\"type\":\"done\"}\n"), ct);
    await http.Response.BodyWriter.FlushAsync(ct);
});
```

> 適合插入心跳、控制 chunk 邊界、或混合二進位資料的場景。

---

## 3) 前端：JavaScript Streams API（Fetch + ReadableStream）

### 3.1 最小可用 NDJSON（逐行解析＋防亂碼）

```html
<div id="log"></div>
<script>
const log = document.getElementById('log');

async function streamNdjson() {
  const res = await fetch('/api/stream/ndjson', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ q: 'hello' })
  });

  if (!res.ok || !res.body) throw new Error('無法建立串流');

  const reader = res.body.getReader();
  const decoder = new TextDecoder(); // 預設 utf-8
  let buf = '';

  try {
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;

      buf += decoder.decode(value, { stream: true });

      // 以換行拆分 NDJSON
      const lines = buf.split('\n');
      buf = lines.pop() ?? '';

      for (const line of lines) {
        if (!line.trim()) continue;
        const obj = JSON.parse(line);

        const div = document.createElement('div');
        div.textContent = JSON.stringify(obj);
        log.appendChild(div);
      }
    }
  } finally {
    // 收尾避免殘段
    const tail = buf.trim();
    if (tail) {
      try {
        const obj = JSON.parse(tail);
        const div = document.createElement('div');
        div.textContent = JSON.stringify(obj);
        log.appendChild(div);
      } catch {}
    }
  }
}

streamNdjson().catch(console.error);
</script>
```

> 關鍵：`TextDecoder(..., { stream: true })` 可避免 UTF-8 多位元組被拆開導致亂碼；**伺服器每筆請務必加 `\n`**。

### 3.2 用 TransformStream 做「行切割器」（可重用）

```js
function makeLineSplitter() {
  let buff = '';
  const decoder = new TextDecoder();
  return new TransformStream({
    transform(chunk, controller) {
      buff += decoder.decode(chunk, { stream: true });
      const lines = buff.split('\n');
      buff = lines.pop() ?? '';
      for (const line of lines) {
        if (line.trim()) controller.enqueue(line);
      }
    },
    flush(controller) {
      if (buff.trim()) controller.enqueue(buff);
    }
  });
}

async function streamNdjsonWithPipe() {
  const res = await fetch('/api/stream/ndjson', { method: 'POST' });
  const readable = res.body
    .pipeThrough(makeLineSplitter())
    .pipeThrough(new TransformStream({
      transform(line, controller) {
        controller.enqueue(JSON.parse(line));
      }
    }));

  const reader = readable.getReader();
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    // value 已是解析後物件
    console.log(value);
  }
}
```

### 3.3 SSE：Fetch 版（可 POST）與 EventSource 版（GET 專用）

**(A) Fetch 讀取 SSE（自寫 parser，適合 POST 與自訂 header）**

```js
async function streamSseByFetch() {
  const res = await fetch('/api/stream/sse', { method: 'POST' });
  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buf = '';

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    buf += decoder.decode(value, { stream: true });

    // SSE 事件由「空行」分隔
    const events = buf.split('\n\n');
    buf = events.pop() ?? '';

    for (const evt of events) {
      const lines = evt.split('\n');
      let type = 'message', data = '';
      for (const ln of lines) {
        if (ln.startsWith('event:')) type = ln.slice(6).trim();
        if (ln.startsWith('data:'))  data += (ln.slice(5).trim() + '\n');
      }
      console.log('SSE', type, data.trim());
    }
  }
}
```

**(B) EventSource（最簡，僅支援 GET）**

```js
const es = new EventSource('/api/stream/sse');
es.addEventListener('token', e => console.log('token', e.data));
es.addEventListener('done',  e => { console.log('done'); es.close(); });
es.onerror = (e) => { console.error('SSE error', e); es.close(); };
```

### 3.4 取消與逾時（AbortController）

```js
const ac = new AbortController();
const timer = setTimeout(() => ac.abort('timeout'), 30_000);

try {
  const res = await fetch('/api/stream/ndjson', { method: 'POST', signal: ac.signal });
  // ...讀取串流
} catch (e) {
  if (ac.signal.aborted) console.warn('已取消：', ac.signal.reason);
} finally {
  clearTimeout(timer);
}
```

---

## 4) 安全與效能最佳實務

### 安全

* **身分驗證**：建議 Bearer Token（Authorization: Bearer …），或 Cookie + CSRF 防護（POST）。
* **輸入驗證**：白名單、長度限制、速率限制（Rate Limiting）。
* **輸出處理**：NDJSON/SSE 的 `data` 請**不要**直接 InnerHTML；先 parse → escape → 再渲染，避免 XSS。

### 效能

* **Backpressure**：Web Streams 天生支援；前端避免每筆都操作 DOM，**批次 append**。
* **Chunk 粒度**：每次 100～300 字元／合理大小，兼顧即時與代理緩衝。
* **Flush**：關鍵節點（例如每段 token 完成）再 Flush，降低上下文切換。
* **取消/逾時**：前端 `AbortController`，後端尊重 `RequestAborted`/`CancellationToken`，避免孤兒工作持續跑。

### 錯誤處理

* **前端**：解析錯誤要有 fallback／原始 chunk 記錄；網路錯誤顯示提示、提供重試。
* **後端**：例外時盡可能發送結尾訊號（若來得及）、敏感訊息不要外洩；記錄必要 context（traceId / userId / route）。

---

## 5) 反向代理與雲端部署重點

### Nginx

```nginx
location /api/stream/ {
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_buffering off;    # 關閉代理緩衝，確保即時
    proxy_cache off;
    proxy_read_timeout 3600s;
}
```

並在回應標頭加 `X-Accel-Buffering: no`（範例已加）。

### IIS / Azure App Service

* 關閉動態壓縮與輸出快取，不然會被緩衝到「不即時」。
* 視需求調整 `web.config` 的 `responseBufferLimit`。
* 反向代理（Application Gateway / Front Door）請確保支援長連線與非緩衝。

### CDN（Cloudflare 等）

* 確認支援串流且關閉 buffering；否則 chunk 會被延遲合併。

---

## 6) 本機測試小抄

```bash
# NDJSON
curl -N -X POST http://localhost:5000/api/stream/ndjson

# SSE
curl -N -X POST http://localhost:5000/api/stream/sse
```

Node.js 測 NDJSON（fetch + reader）：

```js
import fetch from 'node-fetch';

const res = await fetch('http://localhost:5000/api/stream/ndjson', { method: 'POST' });
const reader = res.body.getReader();
const decoder = new TextDecoder();
let buf = '';

for (;;) {
  const { value, done } = await reader.read();
  if (done) break;
  buf += decoder.decode(value, { stream: true });
  const lines = buf.split('\n'); buf = lines.pop() ?? '';
  for (const line of lines) if (line.trim()) console.log(JSON.parse(line));
}
```

---

## 7) 常見地雷與排錯

* **「看起來不即時」**：反向代理 / CDN 有開 buffering。
* **中文亂碼**：全程 UTF-8；`TextDecoder('utf-8', { stream: true })`；回應 `charset=utf-8`。
* **EventSource 想 POST**：SSE 規範 GET-only；要 POST 請改用 `fetch` 讀流。
* **讀到一半中斷**：逾時、網路或前端 abort；後端要尊重 `CancellationToken`，避免工作殭屍化。
* **DOM 卡卡**：每筆 append DOM 會卡；請**累積 N 筆再一次更新**。

---

## 8) 最小整合（可直接貼）

### Server（NDJSON）

```csharp
app.MapPost("/api/stream", (HttpContext http, CancellationToken ct) =>
{
    http.Response.Headers["X-Accel-Buffering"] = "no";
    http.Response.Headers.CacheControl = "no-cache";
    http.Response.ContentType = "application/x-ndjson; charset=utf-8";
    return StreamNdjson(ct); // 見前述 StreamNdjson
});
```

### Client（HTML + JS）

```html
<button id="start">Start</button>
<pre id="out"></pre>
<script>
document.getElementById('start').onclick = async () => {
  const out = document.getElementById('out');
  const res = await fetch('/api/stream', { method: 'POST' });
  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buf = '';

  for (;;) {
    const { value, done } = await reader.read();
    if (done) break;
    buf += decoder.decode(value, { stream: true });
    const lines = buf.split('\n'); buf = lines.pop() ?? '';
    for (const line of lines) if (line.trim()) out.textContent += line + '\n';
  }
};
</script>
```

---

### 9) 一句話總結

> **就是一筆 HTTP 回應，多個 Body chunk**。
> 後端用 `IAsyncEnumerable<T>`（或 `BodyWriter`）分段送；前端用 **Streams API** 逐塊讀，配合 NDJSON 或 SSE 設計，並關閉反向代理緩衝，就能得到低延遲、可維運、可安全擴充的串流體驗。

---

需要我把這份講義轉成 **.md 檔**供你直接下載嗎？我可以立刻生成檔案並給你下載連結。
