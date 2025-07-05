# C# `record` 與 `struct` 詳解整理
作者：ChatGPT + 使用者理解整理

---

## ✅ `record` 是什麼？（C# 9 起支援）

- 預設為 `class`（參考型別）
- 提供：
  - 值相等比較 (`Equals`)
  - 支援 `with` 表達式（非破壞性複製）
  - 自動產生 `ToString()`、`Equals()`、`GetHashCode()`
- **用途**：設定物件、DTO、查詢參數、gRPC metadata 設定等

### 範例：
```csharp
public record User(string Name, int Age);

var u1 = new User("Steve", 30);
var u2 = u1 with { Name = "Tom" }; // u2 是新物件
Console.WriteLine(u1 == u2); // False（值不同）
```

---

## ✅ `record struct` 是什麼？（C# 10 起支援）

- 是值型別（struct），但支援 record 所有語法糖：
  - 值相等、`with`、自動產生方法
- 適合封裝小型不可變資料，效能高、免 GC

### 範例：
```csharp
public readonly record struct Point(int X, int Y);

var p1 = new Point(1, 2);
var p2 = p1 with { Y = 9 }; // 新值型別複製
Console.WriteLine(p1.X); // 1
```

---

## ✅ 型別差異總表

| 類型             | 型別屬性  | 值相等 | 支援 `with` | 支援版本 | 備註                       |
|------------------|------------|--------|--------------|-----------|----------------------------|
| `class`          | Reference  | ❌     | ❌           | -         | 一般邏輯物件               |
| `record`         | Reference  | ✅     | ✅           | C# 9 起   | 適合 DTO/設定/複製需求     |
| `struct`         | Value      | ✅     | ❌           | -         | 效能導向資料容器           |
| `record struct`  | Value      | ✅     | ✅           | C# 10 起  | 高效資料封裝 + 語法糖      |

---

## ✅ 概念說明

- `class` / `record`：傳遞的是「參考」，多個變數指向同一物件
- `struct` / `record struct`：傳遞的是「值」，每次都是資料本身複製
- ⚠ 如果屬性是參考型別（如 `List<string>`），仍然是共用記憶體
  - ✅ 解法：手動深複製 → `new List<string>(原值)`

### 範例（陷阱說明）：
```csharp
public record struct Config(List<string> Items);

var c1 = new Config(new List<string> { "A" });
var c2 = c1 with { }; // 新物件，但共用 List
c2.Items.Add("B");
Console.WriteLine(c1.Items.Count); // 2 ⚠️
```

---

## ✅ 小結語

- ➤ 宣告是 `struct` 才會值複製；**跟屬性型別無關**
- ➤ `record struct` 適合封裝小型不可變資料
- ➤ 若你希望：
  - 支援值比較
  - 可使用 `with` 複製
  - 低記憶體壓力
  ➤ 就使用 `record struct`

---

## ✅ 建議用法

| 用途               | 建議型別       |
|--------------------|----------------|
| 組態設定           | `record`       |
| gRPC 註冊資訊      | `record`       |
| HTTP route 描述器  | `record`       |
| 位置、範圍、快取鍵 | `record struct`|
```
