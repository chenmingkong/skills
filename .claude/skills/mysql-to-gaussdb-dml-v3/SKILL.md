---
name: mysql-to-gaussdb-dml-v3
description: 将 MySQL DML 语句转换为 GaussDB 兼容语法（优化版 v3）
argument-hint: "[Mapper文件或目录]"
---

# MySQL to GaussDB DML 转换（v3）

MySQL DML转GaussDB兼容语法。**版本505.2.1**

## 扫描文件范围

```
**/*Mapper.xml                     # MyBatis 映射文件
**/*Repository.java, **/*Dao.java  # @Query 原生 SQL
**/*.java                          # 其他 Java 文件中的 SQL
**/*.sql                           # DDL 文件（用于类型映射）
```

**重要：必须完整扫描所有匹配文件，逐行检查并转换。**

---

## 快速参考

### 引号转换规则

| 场景 | MySQL | GaussDB | 说明 |
|------|-------|---------|------|
| 表名/字段名（普通） | `` `user` ``, `` `name` `` | `user`, `name` | 去掉反引号，不加双引号 |
| 表名/字段名（关键字） | `` `order` ``, `` `desc` `` | `"order"`, `"desc"` | 关键字字段用双引号包裹 |
| 字段别名（Map） | `AS userName` | `AS "userName"` | resultType="map/HashMap" 时需加双引号 |
| 字段别名（实体类） | `AS userName` | `AS userName` | resultType 为实体类时不加引号 |
| 表别名 | `FROM user u` | `FROM user u` | 不需要引号 |
| 字符串值 | `"张三"`, `"%test%"` | `'张三'`, `'%test%'` | 必须用单引号 |

### 通用函数转换

| MySQL | GaussDB |
|-------|---------|
| `IFNULL(a, b)` | `COALESCE(a, b)` |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` |
| `GROUP_CONCAT(col)` | `STRING_AGG(col::text, ',')` |
| `GROUP_CONCAT(col SEPARATOR ';')` | `STRING_AGG(col::text, ';')` |
| `JSON_OBJECT(key, value)` | `json_build_object(key, value)` |
| `JSON_OBJECT(k1, v1, k2, v2, ...)` | `json_build_object(k1, v1, k2, v2, ...)` |
| `JSON_CONTAINS(col, val)` | `col::jsonb @> val::jsonb` |
| `ANY_VALUE(col)` | `MAX(col)` |
| `SELECT EXISTS(...)` | `SELECT (EXISTS(...))::int` |

### 日期时间函数

| MySQL | GaussDB |
|-------|---------|
| `NOW()`, `SYSDATE()` | `NOW()` |
| `CURDATE()` | `CURRENT_DATE` |
| `CURTIME()` | `CURRENT_TIME` |
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` |
| `UNIX_TIMESTAMP(date)` | `EXTRACT(EPOCH FROM date)::INTEGER` |
| `FROM_UNIXTIME(ts)` | `TO_TIMESTAMP(ts)` |
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:%s')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:SS')` |
| `DATE_FORMAT(d, '%Y-%m-%d %H:00:00')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:00:00')` |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:00')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:00')` |
| `STR_TO_DATE(str, '%Y-%m-%d')` | `TO_DATE(str, 'YYYY-MM-DD')` |
| `DATE_ADD(d, INTERVAL n DAY)` | `TO_CHAR(CAST(d AS TIMESTAMP) + INTERVAL 'n DAY', 'YYYY-MM-DD')` |
| `DATE_SUB(d, INTERVAL n DAY)` | `TO_CHAR(CAST(d AS TIMESTAMP) - INTERVAL 'n DAY', 'YYYY-MM-DD')` |
| `DATEDIFF(d1, d2)` | `(d1::date - d2::date)` |
| `TIMESTAMPDIFF(DAY, start, end)` | `EXTRACT(DAY FROM (end - start))` |
| `TIMESTAMPDIFF(HOUR, start, end)` | `EXTRACT(EPOCH FROM (end - start))::INTEGER / 3600` |
| `DATE(datetime)` | `TO_CHAR(datetime, 'YYYY-MM-DD')` |
| `TIME(datetime)` | `datetime::time` |
| `YEAR(date)` | `EXTRACT(YEAR FROM date)` |
| `MONTH(date)` | `EXTRACT(MONTH FROM date)` |
| `DAY(date)` | `EXTRACT(DAY FROM date)` |

**DATE_FORMAT 格式符：** `%Y`→`YYYY`, `%m`→`MM`, `%d`→`DD`, `%H`→`HH24`, `%i`→`MI`, `%s`→`SS`

---

## 一、标识符与字符串转换

### 1.1 标识符（表名、字段名）

**规则：普通标识符去反引号；SQL 关键字用双引号**

#### 1.1.1 普通标识符

```sql
-- MySQL
SELECT `id`, `user_name` FROM `user` WHERE `status` = 1;
SELECT `User`.`Name` FROM `User`;

-- GaussDB（去反引号）
SELECT id, user_name FROM user WHERE status = 1;
SELECT User.Name FROM User;
```

#### 1.1.2 SQL 关键字字段（重要）

**表名或字段名是 SQL 关键字时，MySQL 用反引号，GaussDB 用双引号。**

常见关键字：`order`, `desc`, `group`, `key`, `value`, `type`, `comment`, `index`, `rank`, `level`, `user` 等

```sql
-- MySQL（关键字使用反引号）
SELECT `id`, `order`, `desc`, `key`, `value` FROM `order`;
SELECT t.`id`, t.`group`, t.`type` FROM `config` t WHERE t.`level` = 1;

-- GaussDB（关键字用双引号）
SELECT id, "order", "desc", "key", "value" FROM "order";
SELECT t.id, t."group", t."type" FROM config t WHERE t."level" = 1;
```

**转换规则：**
- `` `普通字段` `` → `普通字段` （去反引号）
- `` `关键字字段` `` → `"关键字字段"` （改为双引号）
- 需识别常见关键字并转换

### 1.2 字段别名（重要：仅 resultType="map" 需要加双引号）

**规则：检查 SQL 所在 `<select>` 的 resultType，只有 Map 类型需加双引号。**

#### 1.2.1 resultType 为 Map - 别名需要加双引号

当 `<select>` 的 resultType 为以下任一类型时，别名需要加双引号：
- `map`
- `java.util.Map`
- `java.util.HashMap`

```xml
<!-- GaussDB（✅ resultType="map"，别名加双引号）-->
<select id="getUserStats" resultType="map">
    SELECT user_name AS "userName", COUNT(*) AS "totalCount" FROM user;
</select>

<!-- GaussDB（✅ resultType="java.util.HashMap"，别名加双引号）-->
<select id="getUserMap" resultType="java.util.HashMap">
    SELECT user_id AS "userId", user_name AS "userName" FROM user;
</select>
```

**原因：** 别名作为 Map 的 key，需双引号保持驼峰。

#### 1.2.2 其他情况 - 别名不需要加引号

当 resultType 为实体类或使用 resultMap 时，别名不需要加引号：

```xml
<!-- GaussDB（❌ resultType 为实体类，别名不加引号）-->
<select id="getUser" resultType="com.example.entity.User">
    SELECT user_id AS userId, user_name AS userName FROM user;
</select>

<!-- GaussDB（❌ 使用 resultMap，别名不加引号）-->
<select id="getUserList" resultMap="userResultMap">
    SELECT user_id AS userId, user_name AS userName FROM user;
</select>
```

**原因：** MyBatis 通过反射或映射，不依赖别名。

### 1.3 字符串值必须用单引号

GaussDB 中字符串值必须用单引号：

```sql
-- MySQL
WHERE name = "张三" AND status = "active"
WHERE name LIKE "%test%"
WHERE code IN ("A", "B", "C")

-- GaussDB（用单引号）
WHERE name = '张三' AND status = 'active'
WHERE name LIKE '%test%'
WHERE code IN ('A', 'B', 'C')
```

---

## 二、函数转换

### 2.1 通用函数

#### 2.1.1 JSON_OBJECT 转换（重要）

**MySQL 的 `JSON_OBJECT` 转换为 `json_build_object`，参数格式保持一致。**

```sql
-- 场景 1：单键值对
-- MySQL
SELECT JSON_OBJECT('name', user_name) AS user_json FROM user;
SELECT JSON_OBJECT('id', id) AS id_json FROM user;

-- GaussDB（函数名改为 json_build_object）
SELECT json_build_object('name', user_name) AS user_json FROM user;
SELECT json_build_object('id', id) AS id_json FROM user;

-- 场景 2：多键值对
-- MySQL
SELECT JSON_OBJECT('id', id, 'name', user_name, 'age', age) AS user_json FROM user;
SELECT JSON_OBJECT('scene', 'test', 'count', '1', 'status', 'active') AS config;

-- GaussDB（函数名改为 json_build_object，参数保持相同）
SELECT json_build_object('id', id, 'name', user_name, 'age', age) AS user_json FROM user;
SELECT json_build_object('scene', 'test', 'count', '1', 'status', 'active') AS config;

-- 场景 3：字段名作为 key/value
-- MySQL
SELECT JSON_OBJECT(key_column, value_column) AS json_data FROM config;

-- GaussDB
SELECT json_build_object(key_column, value_column) AS json_data FROM config;
```

**转换规则：**
- `JSON_OBJECT(...)` → `json_build_object(...)`
- 适用单/多键值对、字段名参数
- 字符串 key 用单引号

#### 2.1.2 其他通用函数示例

```sql
-- MySQL
SELECT IFNULL(nickname, username) AS display_name,
       IF(status = 1, '启用', '禁用') AS status_text,
       GROUP_CONCAT(name) AS members
FROM user;

-- GaussDB
SELECT COALESCE(nickname, username) AS display_name,
       CASE WHEN status = 1 THEN '启用' ELSE '禁用' END AS status_text,
       STRING_AGG(name::text, ',') AS members
FROM user;
```

### 2.2 日期时间函数示例

```sql
-- MySQL
SELECT DATE_FORMAT(create_time, '%Y-%m-%d') AS date_str,
       DATE_FORMAT(create_time, '%Y-%m-%d %H:00:00') AS hour_str,
       DATE_FORMAT(create_time, '%Y-%m-%d %H:%i:00') AS minute_str,
       DATE_ADD(create_time, INTERVAL 7 DAY) AS next_week,
       TIMESTAMPDIFF(DAY, create_time, NOW()) AS days_ago,
       YEAR(create_time) AS year
FROM orders;

-- GaussDB
SELECT TO_CHAR(create_time, 'YYYY-MM-DD') AS date_str,
       TO_CHAR(create_time, 'YYYY-MM-DD HH24:00:00') AS hour_str,
       TO_CHAR(create_time, 'YYYY-MM-DD HH24:MI:00') AS minute_str,
       TO_CHAR(CAST(create_time AS TIMESTAMP) + INTERVAL '7 DAY', 'YYYY-MM-DD') AS next_week,
       EXTRACT(DAY FROM (NOW() - create_time)) AS days_ago,
       EXTRACT(YEAR FROM create_time) AS year
FROM orders;
```

---

## 三、特殊场景处理

### 3.1 LAST_INSERT_ID 转换（需扫描 DDL）

**重要：必须先扫描 DDL 获取真实序列名，不要猜测！**

#### 步骤 1：扫描项目中的序列定义

```bash
grep -rn "CREATE SEQUENCE" --include="*.sql"
grep -rn "nextval" --include="*.sql"
```

#### 步骤 2：建立表与序列的映射关系

扫描 DDL 文件，找到序列定义：

```sql
-- DDL 文件示例
CREATE SEQUENCE seq_user_id START WITH 1 INCREMENT BY 1;

CREATE TABLE user (
    id BIGINT DEFAULT nextval('seq_user_id'),  -- 真实序列名：seq_user_id
    name NVARCHAR2(50)
);
```

#### 步骤 3：使用真实序列名转换

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

### 3.2 ON DUPLICATE KEY UPDATE 转换（需扫描 DDL）

**支持,不支持ON CONFLICT,UPDATE无唯一索**

#### 步骤 1：扫描唯一索引定义

```bash
grep -rn "UNIQUE" --include="*.sql"
grep -rn "PRIMARY KEY" --include="*.sql"
```

#### 步骤 2：建立表的唯一索引映射

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

#### 步骤 3：转换语句

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

### 3.3 INSERT/UPDATE 类型转换（需扫描 Java 类 + DDL）

**规则：只有当 Java 类型为 String，且数据库类型为 INT/JSON 时才需要转换**

#### 步骤 1：扫描 DDL 获取字段类型

```bash
grep -rnE "\b(INT|INTEGER|BIGINT|SMALLINT|TINYINT)\b" --include="*.sql"
grep -rn "\bJSON\b" --include="*.sql"
```

#### 步骤 2：扫描 Java 类获取参数类型

```bash
grep -rn "private String" --include="*.java"
grep -rnE "private (Integer|Long|int|long)" --include="*.java"
```

#### 步骤 3：对比并转换

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

**类型转换规则：**

| 数据库字段类型 | Java 类型 | 是否需要转换 | GaussDB 转换 |
|---------------|----------|-------------|-------------|
| INT/INTEGER/BIGINT | Integer/Long | ❌ 不需要 | - |
| INT/INTEGER/BIGINT | String | ✅ 需要 | `#{val}::int` |
| SMALLINT | Short/Integer | ❌ 不需要 | - |
| SMALLINT | String | ✅ 需要 | `#{val}::int2` |
| JSON | String | ✅ 需要 | `#{val}::json` |
| VARCHAR/TEXT | String | ❌ 不需要 | - |

### 3.4 GROUP BY 非聚合列处理

GaussDB 严格遵循 SQL 标准，SELECT 中的非聚合列必须出现在 GROUP BY 中或使用聚合函数：

```sql
-- MySQL
SELECT user_id, username, department, COUNT(*) AS order_count
FROM orders GROUP BY user_id;

-- GaussDB（非聚合列需用聚合函数包装）
SELECT user_id, MAX(username) AS username, MAX(department) AS department, COUNT(*) AS order_count
FROM orders GROUP BY user_id;
```

### 3.5 ORDER BY 排序 NULL 值处理

GaussDB 中 NULL 值的默认排序行为与 MySQL 不同，需要显式指定：

```sql
-- MySQL
ORDER BY age ASC, create_time DESC

-- GaussDB（需指定 NULL 排序位置）
ORDER BY age ASC NULLS FIRST, create_time DESC NULLS LAST
```

**规则：**
- 升序（ASC）：添加 `NULLS FIRST`
- 降序（DESC）：添加 `NULLS LAST`

---

## 四、兼容的 MySQL 语法（无需转换）

以下语法 GaussDB 直接兼容：
- `LIMIT offset, count` / `LIMIT count` - 分页语法
- `NOW()` - 当前时间
- `COALESCE()` - 空值处理
- `CONCAT()` - 字符串拼接
- `TRIM()`, `UPPER()`, `LOWER()` - 字符串函数
- `ABS()`, `ROUND()`, `CEIL()`, `FLOOR()` - 数学函数

---

## 五、注意事项

### XML 转义字符保留

- `&lt;` 不需要改成 `<`
- `&gt;` 不需要改成 `>`
- `&amp;` 不需要改成 `&`

这些是 XML 的标准转义，在 MyBatis Mapper XML 中用于比较运算符，必须保留。

### 最小化修改原则

只修改需要转换的语法问题，保持文件其他内容不变，不要做多余的修改。

---

## 六、迁移完整性校验

转换完成后，执行以下检查确保没有遗漏。

### 6.1 必须转换项（期望：无匹配）

**标识符和字符串：**

```bash
# 检查反引号（普通字段去掉，关键字字段改为双引号）
grep -rn "\`" --include="*.xml" --include="*.java"

# 检查双引号字符串值（需区分字符串值和关键字字段）
grep -rnE "=\s*\"[^\"]+\"" --include="*.xml" --include="*.java"
```

**通用函数：**

```bash
grep -rn "IFNULL" --include="*.xml" --include="*.java"
grep -rnE "\bIF\s*\(" --include="*.xml" --include="*.java"
grep -rn "GROUP_CONCAT" --include="*.xml" --include="*.java"
grep -rn "JSON_OBJECT" --include="*.xml" --include="*.java"
grep -rn "JSON_CONTAINS" --include="*.xml" --include="*.java"
grep -rn "ANY_VALUE" --include="*.xml" --include="*.java"
```

**日期时间函数：**

```bash
grep -rn "DATE_FORMAT" --include="*.xml" --include="*.java"
grep -rn "STR_TO_DATE" --include="*.xml" --include="*.java"
grep -rn "UNIX_TIMESTAMP" --include="*.xml" --include="*.java"
grep -rn "FROM_UNIXTIME" --include="*.xml" --include="*.java"
grep -rn "CURDATE" --include="*.xml" --include="*.java"
grep -rn "CURTIME" --include="*.xml" --include="*.java"
grep -rn "SYSDATE" --include="*.xml" --include="*.java"
grep -rnE "DATE_ADD|DATE_SUB" --include="*.xml" --include="*.java"
grep -rn "DATEDIFF" --include="*.xml" --include="*.java"
grep -rn "TIMESTAMPDIFF" --include="*.xml" --include="*.java"
grep -rnE "\bDATE\s*\(" --include="*.xml" --include="*.java"
grep -rnE "\bTIME\s*\(" --include="*.xml" --include="*.java"
grep -rnE "\b(YEAR|MONTH|DAY|HOUR|MINUTE|SECOND)\s*\(" --include="*.xml" --include="*.java"
```

### 6.2 需人工检查项

```bash
# 字段别名（需检查 <select> 的 resultType，只有 Map 类型需加双引号）
grep -rnE "\bAS\s+[a-zA-Z_][a-zA-Z0-9_]*\s*[,\s\n\r\)]" --include="*.xml"
grep -rn 'resultType="map"' --include="*.xml"  # 辅助：查找 map
grep -rn 'resultType="java.util.HashMap"' --include="*.xml"  # 辅助：查找 HashMap

grep -rn "LAST_INSERT_ID" --include="*.xml"  # 需扫描 DDL 获取序列名
grep -rn "ON DUPLICATE KEY" --include="*.xml"  # UPDATE 不能包含唯一索引字段
grep -rnE "ORDER\s+BY" --include="*.xml"  # 确认 NULLS FIRST/LAST
grep -rnE "GROUP\s+BY" --include="*.xml"  # 确认非聚合列用聚合函数
grep -rnE "INSERT\s+INTO|UPDATE\s+\w+\s+SET" --include="*.xml"  # 确认类型转换
```

### 6.3 校验清单

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| 反引号 | `grep -rn "\`"` | 人工确认（关键字用双引号，普通字段去掉） |
| 双引号字符串 | `grep -rnE "=\s*\"[^\"]+\""` | 人工确认（区分字符串值和关键字字段） |
| IFNULL/IF/GROUP_CONCAT | `grep -rnE "IFNULL\|IF\(\|GROUP_CONCAT"` | 无匹配 |
| JSON_OBJECT/JSON_CONTAINS | `grep -rn "JSON_OBJECT\|JSON_CONTAINS"` | 无匹配 |
| DATE_FORMAT/STR_TO_DATE | `grep -rn "DATE_FORMAT\|STR_TO_DATE"` | 无匹配 |
| UNIX_TIMESTAMP/FROM_UNIXTIME | `grep -rn "UNIX_TIMESTAMP\|FROM_UNIXTIME"` | 无匹配 |
| CURDATE/CURTIME/SYSDATE | `grep -rn "CURDATE\|CURTIME\|SYSDATE"` | 无匹配 |
| DATE_ADD/DATE_SUB/DATEDIFF | `grep -rnE "DATE_ADD\|DATE_SUB\|DATEDIFF"` | 无匹配 |
| 字段别名 | `grep -rnE "\bAS\s+[a-zA-Z]"` | Map 类型加双引号 |
| ORDER BY | `grep -rnE "ORDER\s+BY"` | 确认 NULLS FIRST/LAST |
| GROUP BY | `grep -rnE "GROUP\s+BY"` | 确认聚合函数 |
| LAST_INSERT_ID | `grep -rn "LAST_INSERT_ID"` | 扫描 DDL 获取序列名 |
| INSERT/UPDATE 类型 | `grep -rn "INSERT\|UPDATE"` | 确认类型转换 |

### 6.4 生成校验报告

扫描完成后，输出到 `dml_完整性校验报告.txt`：

```
============ DML 迁移校验报告 ============

【标识符和字符串】
✅ 普通字段反引号已去掉
✅ SQL 关键字字段已用双引号（order, desc, group, key, value, type 等）
✅ 字符串值已使用单引号
✅ 字段别名（Map 类型加双引号：map/HashMap）

【通用函数】
✅ IFNULL → COALESCE
✅ IF() → CASE WHEN
✅ GROUP_CONCAT → STRING_AGG
✅ JSON_OBJECT → json_build_object
✅ JSON_CONTAINS → ::jsonb @> ::jsonb
✅ ANY_VALUE → MAX

【日期时间函数】
✅ DATE_FORMAT → TO_CHAR
✅ STR_TO_DATE → TO_DATE/TO_TIMESTAMP
✅ UNIX_TIMESTAMP → EXTRACT(EPOCH FROM ...)
✅ FROM_UNIXTIME → TO_TIMESTAMP
✅ CURDATE/CURTIME/SYSDATE → CURRENT_DATE/CURRENT_TIME/NOW()
✅ DATE_ADD/DATE_SUB → INTERVAL
✅ DATEDIFF/TIMESTAMPDIFF → EXTRACT
✅ DATE/TIME/YEAR/MONTH/DAY → TO_CHAR/::time/EXTRACT

【类型转换】
✅ Java String → DB INT/JSON 已添加 ::int/::json
✅ Java Integer/Long → DB INT无需转换

【特殊语法】
✅ LAST_INSERT_ID → currval('真实序列名')
✅ ON DUPLICATE KEY UPDATE（支持,不支持CONFLICT）
✅ ORDER BY 已添加 NULLS FIRST/LAST
✅ GROUP BY 非聚合列已用聚合函数包装

【待人工检查】
⚠️ SQL 关键字字段：X 处（需用双引号）
⚠️ 字段别名：X 处（Map 类型加双引号：map/HashMap）
⚠️ ORDER BY：X 处（需确认 NULLS）
⚠️ GROUP BY：X 处（需确认聚合函数）
⚠️ LAST_INSERT_ID：X 处（需获取序列名）
⚠️ ON DUPLICATE KEY：X 处（需确认无唯一索引）
⚠️ INSERT/UPDATE 类型转换：X 处（需对照 Java 类）

【统计】
扫描文件数：X 个 | 修改文件数：X 个 | 已转换项：X 处 | 待检查：X 处

============================================
```

---

## 七、完整转换示例

### 示例 1：基础查询转换

```xml
<!-- ========== MySQL ========== -->
<select id="getUserStats" resultType="map">
    SELECT
        `user_id`,
        `username`,
        DATE_FORMAT(`create_time`, '%Y-%m-%d') AS create_date,
        IFNULL(`nickname`, `username`) AS display_name,
        IF(`status` = 1, '启用', '禁用') AS status_text,
        TIMESTAMPDIFF(DAY, `create_time`, NOW()) AS days_since_created
    FROM `user` u
    WHERE `status` = "active"
      AND `create_time` &gt;= DATE_SUB(NOW(), INTERVAL 30 DAY)
    ORDER BY `create_time` DESC
</select>

<!-- ========== GaussDB ========== -->
<select id="getUserStats" resultType="map">
    SELECT
        user_id,
        username,
        TO_CHAR(create_time, 'YYYY-MM-DD') AS "createDate",
        COALESCE(nickname, username) AS "displayName",
        CASE WHEN status = 1 THEN '启用' ELSE '禁用' END AS "statusText",
        EXTRACT(DAY FROM (NOW() - create_time)) AS "daysSinceCreated"
    FROM user u
    WHERE status = 'active'
      AND create_time &gt;= TO_CHAR(CAST(NOW() AS TIMESTAMP) - INTERVAL '30 DAY', 'YYYY-MM-DD')
    ORDER BY create_time DESC NULLS LAST
</select>
```

### 示例 2：INSERT/UPDATE 类型转换

```java
// Java 实体类
public class ConfigDTO {
    private Long userId;         // Long → DB INTEGER，不需要转换
    private Integer age;         // Integer → DB INTEGER，不需要转换
    private String score;        // String → DB INT，需要 ::int
    private String settings;     // String → DB JSON，需要 ::json
}
```

```xml
<!-- ========== MySQL ========== -->
<insert id="insertConfig">
    INSERT INTO config(user_id, age, score, settings)
    VALUES(#{userId}, #{age}, #{score}, #{settings})
</insert>

<update id="updateConfig">
    UPDATE config
    SET age = #{age}, score = #{score}, settings = #{settings}
    WHERE user_id = #{userId}
</update>

<!-- ========== GaussDB ========== -->
<insert id="insertConfig">
    INSERT INTO config(user_id, age, score, settings)
    VALUES(#{userId}, #{age}, #{score}::int, #{settings}::json)
</insert>

<update id="updateConfig">
    UPDATE config
    SET age = #{age},                  -- Integer，不转换
        score = #{score}::int,         -- String → INT，转换
        settings = #{settings}::json   -- String → JSON，转换
    WHERE user_id = #{userId}          -- WHERE 不转换
</update>
```

### 示例 3：SQL 关键字字段转换

当表名或字段名是 SQL 关键字时，需要特殊处理：

```xml
<!-- ========== MySQL（关键字使用反引号） ========== -->
<select id="getOrderList" resultType="map">
    SELECT
        `id`,
        `order`,           -- order 是 SQL 关键字
        `desc`,            -- desc 是 SQL 关键字
        `key`,             -- key 是 SQL 关键字
        `value`,           -- value 是 SQL 关键字
        `type`,            -- type 是 SQL 关键字
        `level`,           -- level 是 SQL 关键字
        `group`            -- group 是 SQL 关键字
    FROM `order`          -- order 是 SQL 关键字
    WHERE `type` = "normal" AND `level` = 1
    ORDER BY `order` ASC, `desc` DESC
</select>

<!-- ========== GaussDB（关键字使用双引号） ========== -->
<select id="getOrderList" resultType="map">
    SELECT
        id,                -- 普通字段，不加引号
        "order",           -- 关键字字段，用双引号
        "desc",            -- 关键字字段，用双引号
        "key",             -- 关键字字段，用双引号
        "value",           -- 关键字字段，用双引号
        "type",            -- 关键字字段，用双引号
        "level",           -- 关键字字段，用双引号
        "group"            -- 关键字字段，用双引号
    FROM "order"          -- 关键字表名，用双引号
    WHERE "type" = 'normal' AND "level" = 1
    ORDER BY "order" ASC NULLS FIRST, "desc" DESC NULLS LAST
</select>
```

**常见 SQL 关键字需加双引号：**

`order`, `desc`, `asc`, `group`, `key`, `value`, `type`, `comment`, `index`, `rank`, `level`, `user`, `position`, `option`, `select`, `insert`, `update`, `delete`, `where`, `from`, `table`, `column`, `check`, `default` 等
```
