# 特殊场景处理规则

## 1. LAST_INSERT_ID 转换（需扫描 DDL）

**重要：必须先扫描 DDL 获取真实序列名，不要猜测！**

### 步骤 1：扫描项目中的序列定义

```bash
grep -rn "CREATE SEQUENCE" --include="*.sql"
grep -rn "nextval" --include="*.sql"
```

### 步骤 2：建立表与序列的映射关系

扫描 DDL 文件，找到序列定义：

```sql
-- DDL 文件示例
CREATE SEQUENCE seq_user_id START WITH 1 INCREMENT BY 1;

CREATE TABLE user (
    id BIGINT DEFAULT nextval('seq_user_id'),  -- 真实序列名：seq_user_id
    name NVARCHAR2(50)
);
```

### 步骤 3：使用真实序列名转换

```sql
-- MySQL
INSERT INTO user(name) VALUES('张三');
SELECT LAST_INSERT_ID();

-- GaussDB（使用扫描到的真实序列名）
INSERT INTO user(name) VALUES('张三');
SELECT currval('seq_user_id');
```

**注意事项：**
- 必须先扫描 DDL 确定真实序列名
- 必须在同一会话中先执行 INSERT，再调用 currval()

**检测模式：** `LAST_INSERT_ID\s*\(`

---

## 2. ON DUPLICATE KEY UPDATE 转换（需扫描 DDL）

**GaussDB 505.2.1 支持 ON DUPLICATE KEY UPDATE，但不支持 ON CONFLICT**

### 步骤 1：扫描唯一索引定义

```bash
grep -rn "UNIQUE" --include="*.sql"
grep -rn "PRIMARY KEY" --include="*.sql"
```

### 步骤 2：建立表的唯一索引映射

```sql
-- DDL 文件示例
CREATE TABLE user (
    id BIGINT PRIMARY KEY,                    -- 主键（唯一）
    username NVARCHAR2(50) UNIQUE,            -- 唯一索引
    email NVARCHAR2(100),
    status INT
);
CREATE UNIQUE INDEX uk_user_email ON user(email);  -- 唯一索引
```

### 步骤 3：转换语句

```sql
-- MySQL
INSERT INTO user(id, username, email, status)
VALUES(1, 'admin', 'admin@test.com', 1)
ON DUPLICATE KEY UPDATE
    username = VALUES(username),  -- ❌ 唯一索引，不能更新
    email = VALUES(email),        -- ❌ 唯一索引，不能更新
    status = VALUES(status);      -- ✅ 非唯一索引，可以更新

-- GaussDB（移除唯一索引字段）
INSERT INTO user(id, username, email, status)
VALUES(1, 'admin', 'admin@test.com', 1)
ON DUPLICATE KEY UPDATE
    status = VALUES(status);      -- ✅ 只更新非唯一索引字段
```

**检测模式：** `ON\s+DUPLICATE\s+KEY\s+UPDATE`

---

## 3. INSERT/UPDATE 类型转换（需扫描 Java 类 + DDL）

**规则：只有当 Java 类型为 String，且数据库类型为 INT/JSON 时才需要转换**

### 步骤 1：扫描 DDL 获取字段类型

```bash
grep -rnE "\b(INT|INTEGER|BIGINT|SMALLINT|TINYINT)\b" --include="*.sql"
grep -rn "\bJSON\b" --include="*.sql"
```

### 步骤 2：扫描 Java 类获取参数类型

```bash
grep -rn "private String" --include="*.java"
grep -rnE "private (Integer|Long|int|long)" --include="*.java"
```

### 步骤 3：对比并转换

```java
// Java 实体类
public class User {
    private Long id;           // Long → DB BIGINT，类型匹配，不需要转换
    private Integer age;       // Integer → DB INTEGER，类型匹配，不需要转换
    private String score;      // ⚠️ String → DB INT，需要 ::int
    private String settings;   // ⚠️ String → DB JSON，需要 ::json
}
```

```sql
-- MySQL
INSERT INTO user(id, age, score, settings)
VALUES(#{id}, #{age}, #{score}, #{settings});

UPDATE user SET age = #{age}, score = #{score}, settings = #{settings}
WHERE id = #{id};

-- GaussDB（只转换 Java String → DB INT/JSON）
INSERT INTO user(id, age, score, settings)
VALUES(#{id}, #{age}, #{score}::int, #{settings}::json);
--    Long不转换 Integer不转换 String→INT  String→JSON

UPDATE user SET
    age = #{age},                  -- Integer，不需要转换
    score = #{score}::int,         -- String → INT，需要转换
    settings = #{settings}::json   -- String → JSON，需要转换
WHERE id = #{id};                  -- WHERE 子句不需要类型转换
```

### 类型转换规则表

| 数据库字段类型 | Java 类型 | 是否需要转换 | GaussDB 转换 |
|---------------|----------|-------------|-------------|
| INT/INTEGER/BIGINT | Integer/Long | ❌ 不需要 | - |
| INT/INTEGER/BIGINT | String | ✅ 需要 | `#{val}::int` |
| SMALLINT | Short/Integer | ❌ 不需要 | - |
| SMALLINT | String | ✅ 需要 | `#{val}::int2` |
| JSON | String | ✅ 需要 | `#{val}::json` |
| VARCHAR/TEXT | String | ❌ 不需要 | - |

**检测模式：** `INSERT\s+INTO`, `UPDATE\s+\w+\s+SET`

---

## 4. GROUP BY 非聚合列处理

**规则：GaussDB 严格遵循 SQL 标准，SELECT 中的非聚合列必须出现在 GROUP BY 中或使用聚合函数**

```sql
-- MySQL
SELECT user_id, username, department, COUNT(*) AS order_count
FROM orders GROUP BY user_id;

-- GaussDB（非聚合列需用聚合函数包装）
SELECT user_id, MAX(username) AS username, MAX(department) AS department, COUNT(*) AS order_count
FROM orders GROUP BY user_id;
```

**处理步骤：**
1. 找到 SELECT 子句中的所有列
2. 找到 GROUP BY 子句中的列
3. 找到使用聚合函数的列（COUNT, SUM, MAX, MIN, AVG 等）
4. 对于不在 GROUP BY 中且没有聚合函数的列，使用 MAX() 或 MIN() 包装

**检测模式：** `GROUP\s+BY`

---

## 5. ORDER BY 排序 NULL 值处理

**规则：GaussDB 中 NULL 值的默认排序行为与 MySQL 不同，需要显式指定**

```sql
-- MySQL
ORDER BY age ASC, create_time DESC

-- GaussDB（需指定 NULL 排序位置）
ORDER BY age ASC NULLS FIRST, create_time DESC NULLS LAST
```

**规则：**
- 升序（ASC）：添加 `NULLS FIRST`
- 降序（DESC）：添加 `NULLS LAST`

**处理步骤：**
1. 找到 ORDER BY 子句
2. 解析每个排序字段
3. 如果是 ASC 或没有指定，添加 `NULLS FIRST`
4. 如果是 DESC，添加 `NULLS LAST`

**检测模式：** `ORDER\s+BY`

---

## 6. LIMIT 语法（兼容，无需转换）

**GaussDB 直接支持 MySQL 的 LIMIT 语法**

```sql
-- MySQL & GaussDB（都兼容）
SELECT * FROM user LIMIT 10;
SELECT * FROM user LIMIT 10, 20;  -- offset 10, count 20
SELECT * FROM user LIMIT 20 OFFSET 10;
```

**无需转换**

---

## 快速参考表

| 场景 | 检测模式 | 处理方式 |
|------|---------|---------|
| LAST_INSERT_ID | `LAST_INSERT_ID\s*\(` | 扫描 DDL 获取序列名 → `currval('序列名')` |
| ON DUPLICATE KEY | `ON\s+DUPLICATE\s+KEY\s+UPDATE` | 扫描 DDL 获取唯一索引 → 移除唯一索引字段 |
| INSERT/UPDATE 类型转换 | `INSERT\s+INTO`, `UPDATE\s+\w+\s+SET` | 扫描 Java 类和 DDL → String→INT/JSON 添加 ::int/::json |
| GROUP BY 非聚合列 | `GROUP\s+BY` | SELECT 非聚合列用 MAX/MIN 包装 |
| ORDER BY NULL 处理 | `ORDER\s+BY` | ASC 添加 NULLS FIRST，DESC 添加 NULLS LAST |
| LIMIT | `LIMIT\s+` | 兼容，无需转换 |

---

## 处理流程

1. **扫描 SQL 语句**
2. **检查是否需要 DDL 信息**：
   - LAST_INSERT_ID → 扫描序列定义
   - ON DUPLICATE KEY UPDATE → 扫描唯一索引
   - INSERT/UPDATE → 扫描 DDL 和 Java 类
3. **应用转换规则**
4. **如果没有匹配项，保持原样**
