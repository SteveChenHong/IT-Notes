# 20260507 .NET 程式崩潰：利用 C# 註冊 Windows Error Reporting（WER）自動產生 Crash Dump，並使用 Visual Studio 分析 `.dmp`

參考文章：
[Automatically create a crash dump file on error - Meziantou Blog](https://www.meziantou.net/tip-automatically-create-a-crash-dump-file-on-error.htm?utm_source=chatgpt.com)

---

# 1. 為什麼需要 Crash Dump

當 `.NET` 程式：

* 無預警閃退
* Service 當掉
* 發生 native crash
* 發生 AccessViolationException
* CLR 崩潰
* IIS Worker Process crash
* 在客戶端環境無法重現問題

單靠 log 很難找到真正原因。

這時候最有效的方法，就是：

> 讓 Windows 在程式崩潰當下，自動產生 `.dmp` 記憶體快照檔。

之後可直接用：

* Visual Studio
* WinDbg
* Rider

打開 dump 進行分析。

---

# 2. Windows Error Reporting（WER）

Windows 本身內建：

# Windows Error Reporting（WER）

當程式 crash 時：

Windows 其實會接管例外處理流程。

只要設定 Registry：

```txt
HKEY_LOCAL_MACHINE
 └─ SOFTWARE
    └─ Microsoft
       └─ Windows
          └─ Windows Error Reporting
             └─ LocalDumps
```

Windows 就會：

* 自動攔截 crash
* 自動產生 `.dmp`
* 不需要額外第三方工具
* 不需要 Visual Studio Attach
* 不需要 Procdump 常駐

這是：

> 「系統層級」的 crash dump 機制。

---

# 3. LocalDumps Registry 說明

## Registry 路徑

```txt
SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\<你的exe名稱>
```

例如：

```txt
SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\MyApp.exe
```

---

# 4. 常用設定值

## DumpFolder

Dump 檔儲存位置：

```txt
DumpFolder = D:\CrashDumps
```

---

## DumpType

控制 dump 詳細程度：

| 值 | 類型        | 說明        |
| - | --------- | --------- |
| 1 | Mini Dump | 較小，只含基本資訊 |
| 2 | Full Dump | 完整記憶體，最常用 |

通常建議：

```txt
DumpType = 2
```

因為：

Mini Dump 很多時候資訊不足。

---

## DumpCount

最多保留幾份 dump：

```txt
DumpCount = 10
```

超過後：

Windows 會自動覆蓋舊檔。

---

# 5. C# 自動註冊 WER

以下程式碼可在程式啟動時：

* 自動建立 Registry
* 自動建立 dump 資料夾
* 自動啟用 crash dump

---

# 6. 完整實作

```csharp
using Microsoft.Win32;
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace YourProject.Helpers;

/// <summary>
/// Windows Error Reporting (WER) 本地轉儲協助工具
/// 協助在程式發生 Crash 時自動產生 Dump 檔案供後續偵錯
/// </summary>
public static class WerDumpHelper
{
    private const string WerLocalDumpsPath = @"SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps";

    /// <summary>
    /// 註冊 WER LocalDumps 註冊表
    /// </summary>
    /// <param name="dumpFolder">Dump 檔案儲存路徑</param>
    /// <param name="dumpType">
    /// Dump 類型：
    /// 0: Custom dump, 1: Mini dump, 2: Full dump (預設)
    /// </param>
    /// <param name="dumpCount">保留的 Dump 檔案數量上限 (預設 10)</param>
    /// <exception cref="UnauthorizedAccessException">若無系統管理員權限則拋出</exception>
    public static void Register(string dumpFolder, int dumpType = 2, int dumpCount = 10)
    {
        if (!RuntimeInformation.IsOSPlatform(OSPlatform.Windows)) return;

        // 取得目前的執行檔名稱 (例如: MyApp.exe)
        string exeName = Path.GetFileName(Process.GetCurrentProcess().MainModule?.FileName) 
            ?? throw new InvalidOperationException("無法取得執行檔名稱");

        string keyPath = $@"{WerLocalDumpsPath}\{exeName}";

        // 建立或開啟註冊表路徑
        using var key = Registry.LocalMachine.CreateSubKey(keyPath);
        if (key == null)
        {
            throw new UnauthorizedAccessException("無法建立註冊表機碼，請確認是否以「系統管理員」身分執行。");
        }

        // 確保目標資料夾存在
        if (!Directory.Exists(dumpFolder))
        {
            Directory.CreateDirectory(dumpFolder);
        }

        // 設定 WER 參數
        // 使用 ExpandString 以支援如 %LOCALAPPDATA% 的環境變數
        key.SetValue("DumpFolder", dumpFolder, RegistryValueKind.ExpandString);
        key.SetValue("DumpType", dumpType, RegistryValueKind.DWord);
        key.SetValue("DumpCount", dumpCount, RegistryValueKind.DWord);
    }

    /// <summary>
    /// 移除 WER LocalDumps 設定，停止自動產生 Dump
    /// </summary>
    public static void Unregister()
    {
        if (!RuntimeInformation.IsOSPlatform(OSPlatform.Windows)) return;

        string exeName = Path.GetFileName(Process.GetCurrentProcess().MainModule?.FileName);
        if (string.IsNullOrEmpty(exeName)) return;

        string keyPath = $@"{WerLocalDumpsPath}\{exeName}";

        // 刪除子機碼，若不存在也不拋出例外
        Registry.LocalMachine.DeleteSubKeyTree(keyPath, throwOnMissingSubKey: false);
    }
}
```

---

# 7. 使用方式

## 啟動時註冊

建議：

在程式啟動最前面呼叫。

例如：

```csharp
WerDumpHelper.Register(
    dumpFolder: @"D:\CrashDumps",
    dumpType: 2,
    dumpCount: 20);
```

例如：

* Program.cs
* Windows Service OnStart
* ASP.NET Core 啟動初始化

都可以。

---

# 8. 測試 Crash Dump

故意讓程式 crash：

```csharp
throw new Exception("Test Crash");
```

或：

```csharp
Environment.FailFast("Force Crash");
```

之後到：

```txt
D:\CrashDumps
```

應該就會看到：

```txt
MyApp.exe.1234.dmp
```

---

# 9. 如何用 Visual Studio 開啟 `.dmp`

Visual Studio 可直接分析 dump：

## 開啟方式

```txt
File
→ Open
→ File
→ *.dmp
```

---

## 可以看到：

* Crash Call Stack
* Thread
* Exception
* Memory
* Loaded Modules
* Native Stack
* Managed Stack

---

# 10. 非常重要：PDB Symbol

如果沒有：

```txt
.pdb
```

很多 stack trace 會失真。

因此：

正式環境建議保留：

```txt
xxx.dll
xxx.pdb
```

至少：

* 保存 CI Build Artifact
* 或建立 Symbol Server

否則 dump 分析價值會大幅下降。

---

# 11. Full Dump 缺點

雖然 Full Dump 最好分析。

但有代價：

| 問題    | 說明             |
| ----- | -------------- |
| 檔案大   | 可能數 GB         |
| 含敏感資料 | 可能包含密碼/token   |
| IO 壓力 | Crash 當下會大量寫磁碟 |

因此正式環境：

要注意：

* dump 保存位置
* 磁碟容量
* 權限控管

---

# 12. 為什麼比 try-catch 更重要

很多 crash：

根本進不了 `try-catch`。

例如：

* StackOverflowException
* AccessViolationException
* Native crash
* CLR internal crash
* OOM
* FailFast

這些：

WER 都能接住。

因此：

> WER 是系統層級 crash 攔截。

不是應用程式層級。

---

# 13. 和 Procdump 的差異

## WER 優點

| 優點         | 說明          |
| ---------- | ----------- |
| Windows 內建 | 不需額外安裝      |
| 不需常駐程式     | 沒背景監控成本     |
| 系統層級       | 穩定          |
| 部署簡單       | Registry 即可 |

---

## Procdump 優點

| 優點          | 說明       |
| ----------- | -------- |
| 可監控 CPU/MEM | 更進階      |
| 可抓 Hang     | 不只 crash |
| 條件式 dump    | 更彈性      |

---

# 14. 建議最佳實踐

## 正式環境建議

### 1. 使用 Full Dump

```txt
DumpType = 2
```

---

### 2. 限制 DumpCount

避免硬碟爆掉：

```txt
DumpCount = 10~20
```

---

### 3. 使用獨立磁碟

避免：

```txt
C:\ 空間被吃滿
```

---

### 4. 保留 Symbol

至少保存：

```txt
dll + pdb
```

---

### 5. 啟動時自動註冊

不要手動設定 Registry。

應用程式自己管理最穩。

---

# 15. 補充：權限問題

因為寫入：

```txt
HKLM
```

所以需要：

* 系統管理員權限
* Windows Service 帳號有 Registry 權限

否則：

```csharp
UnauthorizedAccessException
```

---

# 16. 補充：32 位元 / 64 位元注意事項

Dump 分析時：

Visual Studio / WinDbg：

最好與 crash process 架構一致。

例如：

| 程式  | 建議分析工具       |
| --- | ------------ |
| x86 | x86 debugger |
| x64 | x64 debugger |

否則：

部分 native stack 可能異常。

---

# 17. 補充：建議增加 Exception Logging

WER 只能看到：

> crash 當下

因此仍然建議：

搭配：

* Serilog
* NLog
* ILogger

紀錄：

* request
* parameter
* business flow

這樣 dump 才真正有上下文。

---

# 18. 結論

WER LocalDumps 是：

> Windows 內建、最穩定、成本最低的 crash dump 解法。

尤其適合：

* Windows Service
* 後台程式
* 長時間執行程序
* 客戶端部署
* 無法 attach debugger 的正式環境

核心價值：

```txt
Crash 發生後
能真正還原現場
```

這是：

log 永遠做不到的事情。
