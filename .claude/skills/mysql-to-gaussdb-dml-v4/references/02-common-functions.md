# 通用函数转换规则

## 1. IFNULL 转换

**规则：`IFNULL(a, b)` → `COALESCE(a, b)`**

```sql
-- MySQL
SELECT IFNULL(nickname, username) AS display_name FROM user;
SELECT IFNULL(price, 0) AS final_price FROM product;

-- GaussDB
SELECT COALESCE(nickname, username) AS display_name FROM user;
SELECT COALESCE(price, 0) AS final_price FROM product;
```

**检测模式：** `IFNULL\s*\(`

---

## 2. IF 条件转换

**规则：`IF(cond, a, b)` → `CASE WHEN cond THEN a ELSE b END`**

```sql
-- MySQL
SELECT IF(status = 1, '启用', '禁用') AS status_text FROM user;
SELECT IF(age >= 18, 'adult', 'minor') AS age_group FROM user;

-- GaussDB
SELECT CASE WHEN status = 1 THEN '启用' ELSE '禁用' END AS status_text FROM user;
SELECT CASE WHEN age >= 18 THEN 'adult' ELSE 'minor' END AS age_group FROM user;
```

**检测模式：** `\bIF\s*\(`

**注意：** 需区分 `IF()` 函数和 `IF EXISTS` 语句

---

## 3. GROUP_CONCAT 转换

**规则：`GROUP_CONCAT(col)` → `STRING_AGG(col::text, ',')`**

```sql
-- MySQL
SELECT GROUP_CONCAT(name) AS members FROM user GROUP BY team_id;
SELECT GROUP_CONCAT(name SEPARATOR ';') AS members FROM user GROUP BY team_id;
SELECT GROUP_CONCAT(DISTINCT name ORDER BY name) AS members FROM user;

-- GaussDB
SELECT STRING_AGG(name::text, ',') AS members FROM user GROUP BY team_id;
SELECT STRING_AGG(name::text, ';') AS members FROM user GROUP BY team_id;
SELECT STRING_AGG(DISTINCT name::text, ',' ORDER BY name) AS members FROM user;
```

**转换规则：**
- 默认分隔符：`,`
- 自定义分隔符：提取 `SEPARATOR` 后的值
- 需添加 `::text` 类型转换
- 保留 `DISTINCT` 和 `ORDER BY`

**检测模式：** `GROUP_CONCAT\s*\(`

---

## 4. JSON_OBJECT 转换（重要）

**规则：`JSON_OBJECT(...)` → `json_build_object(...)`**

参数格式保持一致，只改函数名。

```sql
-- 场景 1：单键值对
-- MySQL
SELECT JSON_OBJECT('name', user_name) AS user_json FROM user;
SELECT JSON_OBJECT('id', id) AS id_json FROM user;

-- GaussDB
SELECT json_build_object('name', user_name) AS user_json FROM user;
SELECT json_build_object('id', id) AS id_json FROM user;

-- 场景 2：多键值对
-- MySQL
SELECT JSON_OBJECT('id', id, 'name', user_name, 'age', age) AS user_json FROM user;
SELECT JSON_OBJECT('scene', 'test', 'count', '1', 'status', 'active') AS config;

-- GaussDB
SELECT json_build_object('id', id, 'name', user_name, 'age', age) AS user_json FROM user;
SELECT json_build_object('scene', 'test', 'count', '1', 'status', 'active') AS config;

-- 场景 3：字段名作为 key/value
-- MySQL
SELECT JSON_OBJECT(key_column, value_column) AS json_data FROM config;

-- GaussDB
SELECT json_build_object(key_column, value_column) AS json_data FROM config;
```

**转换规则：**
- 函数名替换：`JSON_OBJECT` → `json_build_object`
- 参数保持不变
- 字符串 key 使用单引号

**检测模式：** `JSON_OBJECT\s*\(`

---

## 5. JSON_CONTAINS 转换

**规则：`JSON_CONTAINS(col, val)` → `col::jsonb @> val::jsonb`**

```sql
-- MySQL
WHERE JSON_CONTAINS(tags, '"premium"')
WHERE JSON_CONTAINS(settings, '{"feature": "enabled"}')

-- GaussDB
WHERE tags::jsonb @> '"premium"'::jsonb
WHERE settings::jsonb @> '{"feature": "enabled"}'::jsonb
```

**检测模式：** `JSON_CONTAINS\s*\(`

---

## 6. ANY_VALUE 转换

**规则：`ANY_VALUE(col)` → `MAX(col)`**

```sql
-- MySQL
SELECT dept_id, ANY_VALUE(dept_name) AS dept_name, COUNT(*) FROM employee GROUP BY dept_id;

-- GaussDB
SELECT dept_id, MAX(dept_name) AS dept_name, COUNT(*) FROM employee GROUP BY dept_id;
```

**检测模式：** `ANY_VALUE\s*\(`

---

## 7. EXISTS 转换

**规则：`SELECT EXISTS(...)` → `SELECT (EXISTS(...))::int`**

当 EXISTS 需要返回整数时（如 0/1），需要类型转换。

```sql
-- MySQL
SELECT EXISTS(SELECT 1 FROM user WHERE id = 1) AS user_exists;

-- GaussDB
SELECT (EXISTS(SELECT 1 FROM user WHERE id = 1))::int AS user_exists;
```

**检测模式：** `SELECT\s+EXISTS\s*\(`

---

## 快速参考表

| MySQL | GaussDB | 检测模式 |
|-------|---------|---------|
| `IFNULL(a, b)` | `COALESCE(a, b)` | `IFNULL\s*\(` |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` | `\bIF\s*\(` |
| `GROUP_CONCAT(col)` | `STRING_AGG(col::text, ',')` | `GROUP_CONCAT\s*\(` |
| `GROUP_CONCAT(col SEPARATOR ';')` | `STRING_AGG(col::text, ';')` | `GROUP_CONCAT\s*\(` |
| `JSON_OBJECT(k, v)` | `json_build_object(k, v)` | `JSON_OBJECT\s*\(` |
| `JSON_CONTAINS(col, val)` | `col::jsonb @> val::jsonb` | `JSON_CONTAINS\s*\(` |
| `ANY_VALUE(col)` | `MAX(col)` | `ANY_VALUE\s*\(` |
| `SELECT EXISTS(...)` | `SELECT (EXISTS(...))::int` | `SELECT\s+EXISTS\s*\(` |

---

## 处理流程

1. **扫描 SQL 语句**
2. **依次匹配检测模式**：
   - `IFNULL\s*\(` → 转换为 `COALESCE`
   - `\bIF\s*\(` → 转换为 `CASE WHEN`
   - `GROUP_CONCAT\s*\(` → 转换为 `STRING_AGG`
   - `JSON_OBJECT\s*\(` → 转换为 `json_build_object`
   - `JSON_CONTAINS\s*\(` → 转换为 `::jsonb @>`
   - `ANY_VALUE\s*\(` → 转换为 `MAX`
   - `SELECT\s+EXISTS\s*\(` → 添加 `::int`
3. **如果没有匹配项，保持原样**
