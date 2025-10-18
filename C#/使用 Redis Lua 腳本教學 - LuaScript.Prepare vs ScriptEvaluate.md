# 📘 Part 1 – 完整修正版講義 (FULL GUIDE)

## C# 中使用 Redis Lua 腳本：LuaScript.Prepare vs ScriptEvaluate

### 目錄 (TOC)

- [前言](#前言)
- [1. 環境設定](#1-環境設定)
- [2. Lua 腳本基礎概念](#2-lua-腳本基礎概念)
- [3. LuaScriptPrepare 教學（高效能模式）](#3-luascriptprepare-教學高效能模式)
- [4. LoadedLuaScript 與快取機制](#4-loadedluascript-與快取機制)
- [5. ScriptEvaluate 教學（一次性腳本模式）](#5-scriptevaluate-教學一次性腳本模式)
- [6. 效能比較與適用情境](#6-效能比較與適用情境)
- [7. 最佳實務與常見誤區](#7-最佳實務與常見誤區)
- [8. 結語](#8-結語)

---

## 前言

Redis 支援 Lua 腳本，可在伺服器端以「原子性」執行邏輯，避免 race condition 與多次往返操作。在 C# 環境中，最常見的 Redis 客戶端為 **StackExchange.Redis**，提供兩種 Lua 執行方式：

| 方式 | 語意 | 使用情境 |
|------|------|----------|
| `LuaScript.Prepare` | 預先編譯並快取，改用 `EVALSHA` | 多次呼叫相同腳本 |
| `IDatabase.ScriptEvaluate` | 每次傳完整腳本，使用 `EVAL` | 臨時 / 測試 / 一次性邏輯 |

---

## 1. 環境設定

```bash
dotnet add package StackExchange.Redis
```

```csharp
using StackExchange.Redis;

class Program
{
    static void Main()
    {
        ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost:6379");
        IDatabase db = redis.GetDatabase();
        redis.Dispose();
    }
}
```

---

## 2. Lua 腳本基礎概念

StackExchange.Redis 提供 @arg 語法糖：

| 腳本中的 @變數 | 對應 Redis | 用途 |
|---------------|-------------|------|
| @key          | KEYS        | Key 類型 |
| @value        | ARGV        | Value 類型 |

---

## 3. LuaScriptPrepare 教學（高效能模式）

```csharp
using StackExchange.Redis;
using StackExchange.Redis.Scripting;

class LuaPrepareExample
{
    public static void Run(IDatabase db)
    {
        string script = @"
            if redis.call('exists', @key) == 1 then
                return redis.call('get', @key)
            else
                redis.call('set', @key, @value)
                return @value
            end";

        LuaScript prepared = LuaScript.Prepare(script);

        RedisValue r1 = (RedisValue)prepared.Evaluate(db,
            new { key = (RedisKey)"sample-key", value = "hello" });

        RedisValue r2 = (RedisValue)prepared.Evaluate(db,
            new { key = (RedisKey)"sample-key", value = "ignored" });
    }
}
```

---

## 4. LoadedLuaScript 與快取機制

```csharp
using System;
using System.Collections.Concurrent;
using System.Security.Cryptography;
using System.Text;
using StackExchange.Redis;
using StackExchange.Redis.Scripting;

class LoadedLuaCacheExample
{
    private static readonly ConcurrentDictionary<string, LoadedLuaScript> Cache =
        new ConcurrentDictionary<string, LoadedLuaScript>();

    private static string ComputeSha1(string script)
    {
        script = script.TrimEnd();
        using var sha1 = SHA1.Create();
        byte[] hash = sha1.ComputeHash(Encoding.UTF8.GetBytes(script));
        return BitConverter.ToString(hash).Replace("-", "").ToLowerInvariant();
    }

    public static void Run(IDatabase db, IServer server)
    {
        string script = @"
            if redis.call('exists', @key) == 1 then
                return redis.call('get', @key)
            else
                redis.call('set', @key, @value)
                return @value
            end";

        string sha1 = ComputeSha1(script);

        LoadedLuaScript loaded = Cache.GetOrAdd(sha1, _ =>
        {
            LuaScript prepared = LuaScript.Prepare(script);
            return prepared.Load(server);
        });

        RedisValue result = (RedisValue)loaded.Evaluate(db,
            new { key = (RedisKey)"sample-key", value = "warmup" });
    }
}
```

---

## 5. ScriptEvaluate 教學（一次性腳本模式）

```csharp
using StackExchange.Redis;

class LuaEvaluateExample
{
    public static void Run(IDatabase db)
    {
        string script = @"
            if redis.call('exists', @key) == 1 then
                return redis.call('get', @key)
            else
                redis.call('set', @key, @value)
                return @value
            end";

        RedisValue result = (RedisValue)db.ScriptEvaluate(
            script,
            new { key = (RedisKey)"eval-key", value = "hello-eval" });
    }
}
```

---

## 6. 效能比較與適用情境

| 指標 | Prepare | ScriptEvaluate |
|------|---------|----------------|
| 傳輸 | SHA1 | 整段腳本 |
| 重複執行 | ✅ | ❌ |
| cluster | lazy load | 每次傳 |
| 適用情境 | 生產 | 臨時 |

---

## 7. 最佳實務與常見誤區

| 誤解 | 正確 |
|------|--------|
| Prepare 自動快取到全部節點 | ❌ lazy load |
| ScriptEvaluate 會快取 | ❌ 不會 |

---

## 8. 結語

Prepare = 生產標配  
ScriptEvaluate = 一次性腳本

---

# 📗 Part 2 – 精簡簡報版 (SLIDE EDITION)

| 方式 | 適用 |
|------|------|
| Prepare | 重複執行 |
| Evaluate | 一次性 |

---

# 📙 Part 3 – 技術部門教學版 (TRAINING EDITION)

✅ 檢查清單  
- 是否 cluster？  
- 是否需要 warm-up？  
- 是否重複執行？  

---

( 完整三合一教學文件 )
