# .NET BackgroundService 高併發與背景任務完整指南（含 IHostedService 比較）

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
    private static readonly ConcurrentQueue<string> _queue = new();

    public void EnqueueLog(string log) => _queue.Enqueue(log);

    public void ProcessLogs()
    {
        var batch = new List<string>();
        int batchSize = 50;

        while (_queue.TryDequeue(out var log))
        {
            batch.Add(log);
            if (batch.Count >= batchSize) break;
        }

        if (batch.Count > 0) WriteBatch(batch);
    }

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
    private readonly BlockingCollection<string> _collection = new(1000);

    public void EnqueueLog(string log) => _collection.Add(log);

    public void ProcessLogs()
    {
        var batch = new List<string>();
        int batchSize = 50;

        while (_collection.TryTake(out var log))
        {
            batch.Add(log);
            if (batch.Count >= batchSize) break;
        }

        if (batch.Count > 0) WriteBatch(batch);
    }

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
    private readonly Channel<string> _channel;

    public ChannelLogWorker(Channel<string> channel) => _channel = channel;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var batch = new List<string>();
        int batchSize = 50;

        await foreach (var log in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            batch.Add(log);
            if (batch.Count >= batchSize)
            {
                await WriteBatch(batch);
                batch.Clear();
            }
        }

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
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await Task.Delay(1000, stoppingToken);
    }

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
        services.AddSingleton<ConcurrentQueueLogProcessor>();
        services.AddSingleton<BlockingCollectionLogProcessor>();
        services.AddSingleton(Channel.CreateUnbounded<string>());

        services.AddHostedService<ChannelLogWorker>();
    });

await builder.RunConsoleAsync();
```

---

## IHostedService vs BackgroundService 比較

### 基本概念

| 類別 | 描述 |
|------|------|
| `IHostedService` | 定義 StartAsync/StopAsync 介面，需自行管理邏輯和資源釋放，靈活性高 |
| `BackgroundService` | 已實作 IHostedService 並提供 ExecuteAsync，簡化開發流程，內建 CancellationToken |

### 使用差異

| 特性 | IHostedService | BackgroundService |
|------|----------------|-----------------|
| 開發複雜度 | 高 | 低 |
| 非同步支援 | 手動實作 | 內建 async/await |
| CancellationToken | 需自行管理 | 內建支援優雅停止 |
| 多個服務共存 | 支援 | 支援 |
| 彈性 | 完全控制 | 部分控制 |

### 範例對比

#### IHostedService 範例

```csharp
public class MyHostedService : IHostedService
{
    private Task _backgroundTask;
    private CancellationTokenSource _cts;

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        _backgroundTask = Task.Run(async () =>
        {
            while (!_cts.Token.IsCancellationRequested)
            {
                Console.WriteLine("IHostedService running...");
                await Task.Delay(1000);
            }
        });
        return Task.CompletedTask;
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        _cts.Cancel();
        await _backgroundTask;
        Console.WriteLine("IHostedService stopped.");
    }
}
```

#### BackgroundService 範例

```csharp
public class MyBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine("BackgroundService running...");
            await Task.Delay(1000, stoppingToken);
        }
        Console.WriteLine("BackgroundService stopped.");
    }
}
```

### 建議使用場景

| 類別 | 適合場景 | 建議理由 |
|------|------------|---------|
| IHostedService | 需要完全自訂 Start/Stop 流程或管理多個背景任務 | 高度彈性，但開發成本高 |
| BackgroundService | 一般背景工作、長時間運行任務、非同步任務 | 開發簡單，支援優雅停止與 async/await，官方推薦 |

---

## 小結

* ConcurrentQueue、BlockingCollection、Channel 都可做高併發背景任務處理，Channel 為官方推薦。
* BackgroundService 封裝 IHostedService，簡化背景任務開發，支援優雅停止。
* IHostedService 適合需要高度自訂流程的情境。
* 結合 Generic Host 可統一管理生命週期與依賴注入。

---

## 參考資源

* [.NET BackgroundService 官方文檔](https://learn.microsoft.com/zh-tw/dotnet/core/extensions/workers)
* [微軟 Windows Service 教學](https://learn.microsoft.com/zh-tw/dotnet/core/extensions/windows-service)
* [高併發 Channel 範例](https://devblogs.microsoft.com/dotnet/dotnet-channels/)

