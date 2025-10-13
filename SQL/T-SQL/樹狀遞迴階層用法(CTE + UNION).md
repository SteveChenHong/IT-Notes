
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
