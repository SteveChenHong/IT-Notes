
# SQL Server 遞迴 CTE 教學

## 1️⃣ CTE（Common Table Expression）介紹

CTE 是 SQL Server 提供的一種臨時結果集，可用於：

- 提高查詢可讀性
- 分段撰寫複雜查詢
- 支援遞迴查詢（階層結構）

### 基本語法：

```sql
WITH CTE_Name (Column1, Column2, ...) AS
(
    -- 查詢內容
    SELECT ...
    FROM ...
    WHERE ...
)
SELECT *
FROM CTE_Name;
```

- `CTE_Name`：臨時表名稱  
- 括號內可定義欄位名稱  
- CTE 的生命週期僅限於緊接著的 `SELECT` / `INSERT` / `UPDATE` / `DELETE` 查詢  

---

## 2️⃣ UNION 與 UNION ALL 差異

| 關鍵字      | 功能                  | 去重重複 | 效能         |
|------------|---------------------|----------|------------|
| `UNION`     | 合併兩個查詢結果       | ✅ 去重   | 較慢，需要排序比對 |
| `UNION ALL` | 合併兩個查詢結果       | ❌ 不去重 | 較快，直接串接 |

### 範例資料

```sql
CREATE TABLE A (Name NVARCHAR(10));
CREATE TABLE B (Name NVARCHAR(10));

INSERT INTO A VALUES ('Tom'), ('Mary');
INSERT INTO B VALUES ('Tom'), ('John');
```

#### 使用 UNION

```sql
SELECT Name FROM A
UNION
SELECT Name FROM B;
```

結果：

| Name |
|------|
| John |
| Mary |
| Tom  |

#### 使用 UNION ALL

```sql
SELECT Name FROM A
UNION ALL
SELECT Name FROM B;
```

結果：

| Name |
|------|
| Tom  |
| Mary |
| Tom  |
| John |

💡 訣竅：  
- `UNION ALL` → 保留所有結果  
- `UNION` → 自動去除重複  

---

## 3️⃣ 遞迴 CTE 的目的

遞迴 CTE 最主要用途是 **展開階層結構（樹狀結構）**：

- 組織架構（主管 → 員工）  
- 分類/目錄樹  
- 家族血緣或任務依賴

遞迴 CTE 可以 **自動展開任意深度的階層**，SQL Server 每輪自動查找下一層資料。

---

## 4️⃣ 遞迴 CTE 語法

```sql
WITH CTE_Name AS
(
    -- 1️⃣ Anchor：起始層
    SELECT ...
    FROM Table
    WHERE ...

    UNION ALL -- 或 UNION

    -- 2️⃣ Recursive：下一層
    SELECT ...
    FROM Table t
    INNER JOIN CTE_Name cte ON t.ParentId = cte.Id
)
SELECT *
FROM CTE_Name;
```

- `Anchor`：起始資料，通常是最上層節點  
- `Recursive`：用 JOIN 尋找下一層  
- `UNION ALL`：每一輪新資料直接加入結果（效能高）  
- `UNION`：每輪去重（效能較慢，但確保結果唯一）  

---

## 5️⃣ 範例：組織架構

### 建立測試資料

```sql
CREATE TABLE Employees (
    Id INT,
    Name NVARCHAR(10),
    ManagerId INT
);

INSERT INTO Employees VALUES
(1, '老闆', NULL),
(2, '經理A', 1),
(3, '經理B', 1),
(4, '員工A1', 2),
(5, '員工A2', 2),
(6, '員工B1', 3);
```

### 遞迴 CTE 查詢整個階層

```sql
WITH EmployeeHierarchy AS (
    -- Anchor: 最上層
    SELECT Id, Name, ManagerId, 0 AS Level
    FROM Employees
    WHERE ManagerId IS NULL

    UNION ALL

    -- Recursive: 找子節點
    SELECT e.Id, e.Name, e.ManagerId, eh.Level + 1
    FROM Employees e
    INNER JOIN EmployeeHierarchy eh ON e.ManagerId = eh.Id
)
SELECT *
FROM EmployeeHierarchy
ORDER BY Level, Id;
```

### 執行結果

| Id | Name   | ManagerId | Level |
|----|--------|-----------|-------|
| 1  | 老闆   | NULL      | 0     |
| 2  | 經理A  | 1         | 1     |
| 3  | 經理B  | 1         | 1     |
| 4  | 員工A1 | 2         | 2     |
| 5  | 員工A2 | 2         | 2     |
| 6  | 員工B1 | 3         | 2     |

💡 說明：

- SQL 已經展開整個階層成平面清單  
- `Level` 用來標示每個節點層級，可做排序或縮排顯示  
- `UNION ALL` 每輪直接加入結果，效能最好  
- `UNION` 每輪會去重，效能稍差，但確保唯一

---

## 6️⃣ C# 遞迴組樹示意

如果需要組成真正樹狀物件，可以在 C# 再遞迴：

```csharp
class EmployeeNode {
    public int Id { get; set; }
    public string Name { get; set; }
    public List<EmployeeNode> Children { get; set; } = new List<EmployeeNode>();
}

EmployeeNode BuildTree(int id, List<Employee> all) {
    var node = new EmployeeNode {
        Id = id,
        Name = all.First(e => e.Id == id).Name
    };
    node.Children.AddRange(
        all.Where(e => e.ManagerId == id)
           .Select(e => BuildTree(e.Id, all))
    );
    return node;
}

// 使用範例
var allEmployees = db.Query<Employee>("...遞迴CTE SQL...").ToList();
var rootId = allEmployees.First(e => e.ManagerId == null).Id;
var treeRoot = BuildTree(rootId, allEmployees);
```

- C# 遞迴組樹**不需要 SQL 的 Level**，只需要父子關係  
- SQL 遞迴 CTE 的 Level 主要用於**排序或縮排顯示**  

---

## 7️⃣ 小結

1. `WITH CTE` → 定義臨時表，可用於遞迴  
2. `UNION ALL` → 遞迴時高效展開全部資料  
3. `UNION` → 遞迴時去重，保證唯一值  
4. 遞迴 CTE → SQL 層面展開階層，產生平面清單  
5. C# 遞迴 → 可選，把平面清單組成真正樹狀物件  
6. `Level` → 標示層級，方便排序或縮排展示


## 8 額外補充，釐清邏輯觀念

# SQL Server 遞迴 CTE (Common Table Expression) 教學整理

> 整理日期：2025/10/13  
> 作者：ChatGPT GPT-5 教學整合  
> 主題：SQL Server 的遞迴查詢 (WITH ... UNION ALL ...)

---

## 🧩 一、什麼是 CTE

CTE 全名是 **Common Table Expression**，中文稱作「共用資料表運算式」。  
語法上使用 `WITH` 開頭，讓你在一個查詢中建立一個暫時的結果集。

### ✅ 基本語法

```sql
WITH CTE名稱 AS (
    SELECT 查詢定義
)
SELECT * FROM CTE名稱;
```

CTE 在語法上可視為一個暫時的「可再利用查詢結果」，不會永久存在資料庫中。

---

## 🧮 二、遞迴 CTE 的結構

遞迴 CTE 有兩個主要部分：

| 部分名稱 | 作用 | 執行時機 |
|-----------|------|-----------|
| Anchor Member | 起始層（例如：最上層主管） | 執行一次 |
| Recursive Member | 從上層結果延伸出下一層 | 反覆執行直到沒有新資料 |

### 🔹 官方定義（Microsoft Docs）

> A recursive common table expression (CTE) is a CTE that references itself.  
> It consists of an **anchor member** and a **recursive member** combined by `UNION ALL` or `UNION`.  
> The anchor member produces the base result set.  
> The recursive member repeatedly references the CTE to produce subsequent result sets until no more rows are returned.

---

## ⚙️ 三、SQL Server 遞迴 CTE 範例

### 範例資料

```sql
CREATE TABLE Employees (
    Id INT PRIMARY KEY,
    Name NVARCHAR(50),
    ManagerId INT NULL
);

INSERT INTO Employees VALUES
(1, 'CEO', NULL),
(2, 'Manager A', 1),
(3, 'Manager B', 1),
(4, 'Staff A1', 2),
(5, 'Staff A2', 2),
(6, 'Staff B1', 3);
```

### 範例查詢

```sql
WITH EmployeeHierarchy AS (
    -- Anchor (第一層：沒有主管的人)
    SELECT Id, Name, ManagerId, 0 AS Level
    FROM Employees
    WHERE ManagerId IS NULL

    UNION ALL

    -- Recursive (從上一層往下找下一層)
    SELECT e.Id, e.Name, e.ManagerId, eh.Level + 1
    FROM Employees e
    INNER JOIN EmployeeHierarchy eh ON e.ManagerId = eh.Id
)
SELECT * FROM EmployeeHierarchy;
```

### 執行結果

| Id | Name | ManagerId | Level |
|----|------|------------|-------|
| 1 | CEO | NULL | 0 |
| 2 | Manager A | 1 | 1 |
| 3 | Manager B | 1 | 1 |
| 4 | Staff A1 | 2 | 2 |
| 5 | Staff A2 | 2 | 2 |
| 6 | Staff B1 | 3 | 2 |

---

## 🧠 四、為什麼要使用 UNION ALL

### 🔸 遞迴 CTE 結構依賴 UNION ALL

- `UNION ALL` 用來分隔「起始查詢」與「遞迴查詢」。
- 每次執行都會將新的結果「累加」回去，直到沒有新資料。

### 🔸 為什麼不是 UNION？

| 比較項目 | UNION ALL | UNION |
|-----------|------------|--------|
| 是否去除重複資料 | 否 | 是 |
| 效能 | 快 | 慢 |
| 遞迴結果是否完整 | 是 | 可能被截斷 |
| 官方建議 | ✅ 使用 | ⚠️ 不建議 |

### ⚠️ 使用 UNION 的風險
- SQL Server 會嘗試 DISTINCT 去重複，導致資料中斷。
- 若同一筆資料出現在不同層級，可能被去除，造成遞迴錯誤。

---

## 🔄 五、內部執行原理（為什麼會遞迴）

1. **Anchor 部分** 先執行一次 → 產生第一層結果。
2. SQL Server 會自動把該結果餵進遞迴部分。
3. 遞迴部分再產生新結果，繼續餵回自己。
4. 直到遞迴查詢產生 0 筆資料 → 結束。

> 這就像是 SQL Server 幫你在背後做一個 while 迴圈：  
> `WHILE 新結果有資料 → 執行下一層遞迴`。

---

## 🧰 六、不用 UNION ALL 的替代方法

### 1️⃣ 模擬遞迴（使用暫存表 + 迴圈）

```sql
DECLARE @Level INT = 0;
DECLARE @tmp TABLE (Id INT, Name NVARCHAR(50), ManagerId INT, Level INT);

-- 第一層
INSERT INTO @tmp
SELECT Id, Name, ManagerId, 0 FROM Employees WHERE ManagerId IS NULL;

-- 模擬遞迴
WHILE EXISTS (
    SELECT 1 FROM Employees e
    INNER JOIN @tmp t ON e.ManagerId = t.Id WHERE t.Level = @Level
)
BEGIN
    SET @Level = @Level + 1;
    INSERT INTO @tmp
    SELECT e.Id, e.Name, e.ManagerId, @Level
    FROM Employees e
    INNER JOIN @tmp t ON e.ManagerId = t.Id
    WHERE t.Level = @Level - 1;
END

SELECT * FROM @tmp;
```

### 2️⃣ 使用 HierarchyID

```sql
CREATE TABLE Employees (
    Id INT PRIMARY KEY,
    Name NVARCHAR(50),
    ManagerId INT,
    Node hierarchyid
);
```

使用內建函數：  
- `GetAncestor()`  
- `GetDescendant()`  
- `IsDescendantOf()`

即可查詢上下層關係。

### 3️⃣ 在程式端遞迴（C# 範例）

```csharp
List<Employee> BuildTree(List<Employee> all, int? managerId)
{
    return all
        .Where(e => e.ManagerId == managerId)
        .Select(e => {
            e.Children = BuildTree(all, e.Id);
            return e;
        }).ToList();
}
```

---

## 📚 七、重點整理

| 主題 | 說明 |
|------|------|
| 遞迴 CTE 關鍵語法 | `WITH CTE AS (Anchor UNION ALL Recursive)` |
| 為什麼用 UNION ALL | 不去重複、效能佳、層級不會被截斷 |
| 遞迴行為關鍵點 | 第二段查詢再次 JOIN 自己 (CTE) |
| 是否可以用 UNION | 語法可行，但不建議 |
| C# 是否還要再遞迴 | 若要組成樹狀物件則需要，否則可直接展示 |

---

## ✅ 八、結論

- `WITH ... UNION ALL ...` 是 SQL Server 官方的 **唯一正式遞迴機制**。  
- 它透過「Anchor + Recursive Member」的結構自我呼叫，直到沒有新資料為止。  
- `UNION ALL` 是關鍵，因為它讓每層資料完整保留。  
- 若不想用 SQL 遞迴，也可在 C# 端遞迴組樹。

---

> 📘 參考：  
> [Microsoft Learn - Recursive Queries Using Common Table Expressions](https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql)


