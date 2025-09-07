# .NET BackgroundService 高併發與背景任務完整指南（含詳盡註解）

## 什麼是 BackgroundService

`BackgroundService` 是 .NET Core 提供的 **長時間運行背景任務基類**，主要用途包括：

* 日誌記錄 (Logging)
* 消費佇列任務 (Queue Consumer)
* 定時工作 (Periodic Tasks)
* 高併發背景處理 (High Concurrency Background Jobs)

**特點**:

* 繼承自 `IHostedService`，整合到 Generic Host
* 支援非同步處理 (async/await)
* 支援優雅停止 (Graceful shutdown)
* 可注入其他服務 (Dependency Injection)

---

## 高併發任務處理方式比較

| 類別                 | 特性                          | 阻塞行為           | 適用場景              |
| ------------------ | --------------------------- | -------------- | ----------------- |
| ConcurrentQueue    | 非阻塞、多線程安全                   | 不阻塞            | 高併發生產者-消費者，簡單批次處理 |
| BlockingCollection | 支援容量限制，內部可用 ConcurrentQueue | 可阻塞 (Add/Take) | 需要控制記憶體，支援阻塞讀寫    |
| Channel            | 官方推薦，支援 async/await         | 可非阻塞或等待        | 高併發非同步背景任務，適合批次處理 |

---

## 1️⃣ ConcurrentQueue 範例

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;

public class ConcurrentQueueLogProcessor
{
    // 使用 ConcurrentQueue 保存 log，支援多線程安全
    private static readonly ConcurrentQueue<string> _queue = new();

    // 生產者：將 log 放入佇列
    public void EnqueueLog(string log) => _queue.Enqueue(log);

    // 消費者：批次處理 log
    public void ProcessLogs()
    {
        var batch = new List<string>();
        int batchSize = 50; // 每次處理 50 筆

        // 嘗試取出資料，非阻塞
        while (_queue.TryDequeue(out var log))
        {
            batch.Add(log);
            if (batch.Count >= batchSize) break;
        }

        if (batch.Count > 0) WriteBatch(batch);
    }

    // 模擬寫入磁碟或資料庫
    private void WriteBatch(List<string> batch)
    {
        Console.WriteLine($"ConcurrentQueue: Writing {batch.Count} logs to disk...");
    }
}
```

---

## 2️⃣ BlockingCollection 範例

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Threading.Tasks;

public class BlockingCollectionLogProcessor
{
    // 最大容量 1000，可阻塞生產者直到有空間
    private readonly BlockingCollection<string> _collection = new(1000);

    // 生產者：放入 log，可能阻塞
    public void EnqueueLog(string log) => _collection.Add(log);

    // 消費者：批次處理 log
    public void ProcessLogs()
    {
        var batch = new List<string>();
        int batchSize = 50;

        // 嘗試取出資料，可能阻塞直到有資料
        while (_collection.TryTake(out var log))
        {
            batch.Add(log);
            if (batch.Count >= batchSize) break;
        }

        if (batch.Count > 0) WriteBatch(batch);
    }

    // 模擬寫入磁碟或資料庫
    private void WriteBatch(List<string> batch)
    {
        Console.WriteLine($"BlockingCollection: Writing {batch.Count} logs to disk...");
    }
}
```

---

## 3️⃣ Channel 範例 (官方推薦)

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;

public class ChannelLogWorker : BackgroundService
{
    // 注入 Channel 物件
    private readonly Channel<string> _channel;

    public ChannelLogWorker(Channel<string> channel) => _channel = channel;

    // 背景服務主執行邏輯
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var batch = new List<string>();
        int batchSize = 50;

        // 非同步讀取 channel 內容，支援優雅停止
        await foreach (var log in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            batch.Add(log);
            if (batch.Count >= batchSize)
            {
                await WriteBatch(batch);
                batch.Clear();
            }
        }

        // 停止前處理剩餘 log
        if (batch.Count > 0) await WriteBatch(batch);
    }

    private Task WriteBatch(List<string> batch)
    {
        Console.WriteLine($"Channel: Writing {batch.Count} logs to disk...");
        return Task.CompletedTask;
    }
}
```

---

## 優雅停止範例

```csharp
// ExecuteAsync 使用 CancellationToken 偵測停止訊號
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        // 背景工作邏輯
        await Task.Delay(1000, stoppingToken); // 模擬工作
    }

    // 優雅釋放資源或處理最後批次
    await CleanupBeforeShutdown();
}

private Task CleanupBeforeShutdown()
{
    Console.WriteLine("Cleaning up before shutdown...");
    return Task.CompletedTask;
}
```

---

## 啟動 BackgroundService

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System.Threading.Channels;

var builder = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        // 註冊 ConcurrentQueueLogProcessor
        services.AddSingleton<ConcurrentQueueLogProcessor>();

        // 註冊 BlockingCollectionLogProcessor
        services.AddSingleton<BlockingCollectionLogProcessor>();

        // 註冊 Channel，無界限
        services.AddSingleton(Channel.CreateUnbounded<string>());

        // 註冊背景服務 ChannelLogWorker
        services.AddHostedService<ChannelLogWorker>();
    });

// 啟動 Console 應用程式 Generic Host
await builder.RunConsoleAsync();
```

---

## 小結

* **ConcurrentQueue**: 非阻塞，高併發簡單批次處理
* **BlockingCollection**: 支援容量限制，可阻塞操作，控制記憶體使用
* **Channel**: 官方推薦，支援 async/await，非阻塞高併發批次處理
* **BackgroundService + CancellationToken**: 可優雅停止任務
* **Generic Host**: 管理生命週期，簡化啟動流程

---

## 參考資源

* [.NET BackgroundService 官方文檔](https://learn.microsoft.com/zh-tw/dotnet/core/extensions/workers)
* [微軟 Windows Service 教學](https://learn.microsoft.com/zh-tw/dotnet/core/extensions/windows-service)
* [高併發 Channel 範例](https://devblogs.microsoft.com/dotnet/dotnet-channels/)

