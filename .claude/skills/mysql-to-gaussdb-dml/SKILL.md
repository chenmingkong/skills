---
name: mysql-to-gaussdb-dml
description: 将 MySQL DML 语句转换为 GaussDB 兼容语法（查询、函数、日期处理等）
argument-hint: "[Mapper文件或目录]"
---

# MySQL to GaussDB DML 转换

将 MySQL 的 DML（数据操作语言）转换为 GaussDB 兼容语法，包括 SELECT、INSERT、UPDATE、DELETE 语句中的函数和语法。

**GaussDB 内核版本：505.2.1**

## 扫描文件

```
**/*Mapper.xml                        # MyBatis 映射文件
**/*Repository.java, **/*Dao.java    # @Query 原生 SQL
**/*.java                            # 其他 Java 文件中的 SQL
```

**重要：必须对所有匹配的文件进行完整扫描，逐行检查并转换，确保文件从头到尾每一行都经过处理，不要遗漏文件后半部分的内容。**

## 1. 标识符引号转换（表名、字段名）

### 1.1 普通标识符

MySQL 使用反引号 `` ` `` 包裹表名和字段名，GaussDB 中需要根据情况处理：

```sql
-- MySQL 写法（有反引号）
SELECT `id`, `user_name` FROM `user` WHERE `status` = 1;
SELECT `User`.`Name` FROM `User`;

-- GaussDB 写法（反引号改为双引号）
SELECT "id", "user_name" FROM "user" WHERE "status" = 1;
SELECT "User"."Name" FROM "User";

-- MySQL 写法（无反引号）
SELECT id, user_name FROM user WHERE status = 1;

-- GaussDB 写法（保持原样）
SELECT id, user_name FROM user WHERE status = 1;
```

**转换规则：**
- `` `identifier` `` → `"identifier"` （反引号改为双引号）
- `` `table`.`column` `` → `"table"."column"` （反引号改为双引号）
- `identifier` → `identifier` （无反引号则保持原样）

### 1.2 SQL 关键字字段（重要）

当表名或字段名是 SQL 关键字时，MySQL 使用反引号，GaussDB 必须使用双引号：

常见 SQL 关键字：`order`, `desc`, `group`, `key`, `value`, `type`, `comment`, `index`, `rank`, `level`, `user` 等

```sql
-- MySQL（关键字使用反引号）
SELECT `id`, `order`, `desc`, `key`, `value` FROM `order`;
SELECT t.`id`, t.`group`, t.`type` FROM `config` t WHERE t.`level` = 1;

-- GaussDB（关键字必须使用双引号）
SELECT id, "order", "desc", "key", "value" FROM "order";
SELECT t.id, t."group", t."type" FROM config t WHERE t."level" = 1;
```

**转换规则：**
- `` `普通字段` `` → `"普通字段"` （反引号改为双引号）
- `` `关键字字段` `` → `"关键字字段"` （关键字必须用双引号）
- `关键字字段` → `"关键字字段"` （无反引号的关键字也需加双引号）
- `普通字段` → `普通字段` （无反引号的普通字段保持原样）

### 1.3 字段别名（重要：仅 resultType="map/HashMap" 需要加双引号）

**规则：需检查 SQL 所在 `<select>` 标签的 resultType，只有 Map 类型需加双引号，其他情况不需要。**

#### 1.3.1 resultType 为 Map - 别名需要加双引号

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

**原因：** 别名作为 Map 的 key，需要双引号保持驼峰命名。

#### 1.3.2 其他情况 - 别名不需要加引号

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

**原因：** MyBatis 通过反射或自定义映射，不依赖别名大小写。

**注意事项：**
- 表别名（如 `FROM user u`）不需要加引号，保持原样即可
- 必须检查 SQL 所在的 `<select>` 标签的 resultType 属性

## 2. 字符串值引号转换

GaussDB 中双引号仅用于别名，**字符串值必须用单引号**，特别是 WHERE 子句中的值：

```sql
-- MySQL（双引号和单引号都可用于字符串）
SELECT * FROM user WHERE name = "张三";
SELECT * FROM user WHERE status = "active" AND type = "admin";
INSERT INTO user(name) VALUES("李四");

-- GaussDB（字符串必须用单引号，双引号会被识别为标识符）
SELECT * FROM user WHERE name = '张三';
SELECT * FROM user WHERE status = 'active' AND type = 'admin';
INSERT INTO user(name) VALUES('李四');
```

**WHERE 子句中常见的双引号转换：**

```sql
-- MySQL
WHERE status = "1"
WHERE type = "normal"
WHERE name LIKE "%张%"
WHERE code IN ("A", "B", "C")

-- GaussDB
WHERE status = '1'
WHERE type = 'normal'
WHERE name LIKE '%张%'
WHERE code IN ('A', 'B', 'C')
```

**转换规则：**
- `"字符串值"` → `'字符串值'` （双引号改为单引号）
- `LIKE "%xxx%"` → `LIKE '%xxx%'` （LIKE 中的双引号也要改）
- `IN ("a", "b")` → `IN ('a', 'b')` （IN 列表中的双引号也要改）

## 3. 通用函数转换

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `IFNULL(a, b)` | `COALESCE(a, b)` | 空值处理 |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` | 条件判断 |
| `GROUP_CONCAT(col)` | `STRING_AGG(col::text, ',')` | 字符串聚合 |
| `GROUP_CONCAT(col SEPARATOR ';')` | `STRING_AGG(col::text, ';')` | 指定分隔符 |
| `JSON_OBJECT(k1, v1, k2, v2)` | `json_build_object(k1, v1, k2, v2)` | 构建JSON对象（多键值对） |
| `JSON_OBJECT(key, value)` | `json_build_object(key, value)` | 构建JSON对象（单键值对） |
| `JSON_CONTAINS(col, json_val)` | `col::jsonb @> json_val::jsonb` | JSON包含检查 |
| `ANY_VALUE(col)` | `MAX(col)` | 任意值 |
| `SELECT EXISTS(...)` | `SELECT (EXISTS(...))::int` | EXISTS返回0/1 |
| `LAST_INSERT_ID()` | `currval('表名_列名_seq')` | 获取最后插入的自增ID |

**示例：**

```sql
-- MySQL
SELECT IFNULL(nickname, username) AS display_name FROM user;
SELECT IF(status = 1, '启用', '禁用') AS status_text FROM user;
SELECT department, GROUP_CONCAT(name) AS members FROM user GROUP BY department;
SELECT * FROM article WHERE JSON_CONTAINS(dataset, '{"隐私":[]}');
SELECT JSON_OBJECT('scene', 'test', 'count', '1') AS config;  -- 多键值对
SELECT JSON_OBJECT('test', '1') AS data;  -- 单键值对

-- GaussDB
SELECT COALESCE(nickname, username) AS display_name FROM user;
SELECT CASE WHEN status = 1 THEN '启用' ELSE '禁用' END AS status_text FROM user;
SELECT department, STRING_AGG(name::text, ',') AS members FROM user GROUP BY department;
SELECT * FROM article WHERE dataset::jsonb @> '{"隐私":[]}'::jsonb;
SELECT json_build_object('scene', 'test', 'count', '1') AS config;  -- 多键值对
SELECT json_build_object('test', '1') AS data;  -- 单键值对
```

### 3.1 LAST_INSERT_ID 转换

MySQL 的 `LAST_INSERT_ID()` 用于获取最后插入的自增 ID，GaussDB 需要使用 `currval('序列名')` 替代。

**重要：必须先扫描 DDL/SQL 文件获取真实序列名，不要猜测！**

#### 步骤 1：扫描项目中的 SQL 文件，查找序列定义

```bash
# 查找所有序列定义
grep -rn "CREATE SEQUENCE" --include="*.sql"

# 查找序列与表的关联（DEFAULT nextval）
grep -rn "nextval" --include="*.sql"
```

#### 步骤 2：建立表与序列的映射关系

扫描 DDL 文件，找到类似以下的定义：

```sql
-- DDL 文件中的序列定义示例
CREATE SEQUENCE seq_user_id START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE sys_user_id_seq START WITH 1 INCREMENT BY 1;

-- 表定义中引用序列
CREATE TABLE user (
    id BIGINT DEFAULT nextval('seq_user_id'),  -- 真实序列名：seq_user_id
    name NVARCHAR2(50)
);

CREATE TABLE sys_user (
    id BIGINT DEFAULT nextval('sys_user_id_seq'),  -- 真实序列名：sys_user_id_seq
    username NVARCHAR2(50)
);
```

#### 步骤 3：使用真实序列名进行转换

```sql
-- MySQL
INSERT INTO user(name) VALUES('张三');
SELECT LAST_INSERT_ID();

INSERT INTO sys_user(username) VALUES('admin');
SELECT LAST_INSERT_ID();

-- GaussDB（使用从 DDL 扫描到的真实序列名）
INSERT INTO user(name) VALUES('张三');
SELECT currval('seq_user_id');  -- 真实序列名，非猜测

INSERT INTO sys_user(username) VALUES('admin');
SELECT currval('sys_user_id_seq');  -- 真实序列名，非猜测
```

#### 序列名映射表（转换前先填写）

| 表名 | 自增列 | 真实序列名 | 来源文件 |
|------|--------|-----------|----------|
| user | id | seq_user_id | schema.sql:15 |
| sys_user | id | sys_user_id_seq | schema.sql:28 |
| orders | order_id | orders_order_id_seq | orders.sql:5 |

**注意事项：**
- **必须**先扫描 DDL 文件确定真实序列名，不要假设命名规则
- 序列名可能是自定义的，如 `seq_xxx`、`xxx_sequence` 等
- 必须在同一会话中先执行 INSERT，再调用 `currval()`
- 可通过数据库命令 `\ds` 或 `SELECT * FROM pg_sequences` 查看所有序列

### 3.2 ON DUPLICATE KEY UPDATE 转换

**重要说明：**
- GaussDB **支持** `ON DUPLICATE KEY UPDATE` 语法，保持使用这个语法即可
- **不要使用** `ON CONFLICT DO UPDATE` 语法，我们的 GaussDB 版本不支持
- UPDATE 子句中**不能包含唯一索引字段**

**转换规则：保持 ON DUPLICATE KEY UPDATE 语法，只需移除 UPDATE 子句中的唯一索引字段**

**重要：必须先扫描 DDL/SQL 文件获取表的唯一索引字段！**

#### 步骤 1：扫描项目中的 SQL 文件，查找唯一索引定义

```bash
# 查找唯一索引定义
grep -rn "UNIQUE" --include="*.sql"

# 查找主键定义
grep -rn "PRIMARY KEY" --include="*.sql"
```

#### 步骤 2：建立表的唯一索引映射关系

扫描 DDL 文件，找到类似以下的定义：

```sql
-- DDL 文件中的唯一索引定义示例
CREATE TABLE user (
    id BIGINT PRIMARY KEY,                    -- 主键（唯一）
    username NVARCHAR2(50) UNIQUE,            -- 唯一索引字段
    email NVARCHAR2(100),
    phone NVARCHAR2(20),
    status INT
);

CREATE UNIQUE INDEX uk_user_email ON user(email);  -- 唯一索引字段
CREATE UNIQUE INDEX uk_user_phone ON user(phone);  -- 唯一索引字段
```

#### 步骤 3：转换 ON DUPLICATE KEY UPDATE 语句

```sql
-- MySQL 原始语句
INSERT INTO user(id, username, email, phone, status)
VALUES(1, 'admin', 'admin@test.com', '13800138000', 1)
ON DUPLICATE KEY UPDATE
    username = VALUES(username),  -- ❌ username 是唯一索引，不能更新
    email = VALUES(email),        -- ❌ email 是唯一索引，不能更新
    phone = VALUES(phone),        -- ❌ phone 是唯一索引，不能更新
    status = VALUES(status);      -- ✅ status 不是唯一索引，可以更新

-- GaussDB 转换后（移除唯一索引字段的更新）
INSERT INTO user(id, username, email, phone, status)
VALUES(1, 'admin', 'admin@test.com', '13800138000', 1)
ON DUPLICATE KEY UPDATE
    status = VALUES(status);      -- ✅ 只更新非唯一索引字段
```

#### 唯一索引映射表（转换前先填写）

| 表名 | 唯一索引字段 | 索引类型 | 来源文件 |
|------|-------------|----------|----------|
| user | id | PRIMARY KEY | schema.sql:10 |
| user | username | UNIQUE | schema.sql:11 |
| user | email | UNIQUE INDEX | schema.sql:20 |
| user | phone | UNIQUE INDEX | schema.sql:21 |
| orders | order_no | UNIQUE | orders.sql:8 |

#### 转换规则

| 字段类型 | ON DUPLICATE KEY UPDATE 中 |
|---------|---------------------------|
| PRIMARY KEY | ❌ 不能包含 |
| UNIQUE 约束字段 | ❌ 不能包含 |
| UNIQUE INDEX 字段 | ❌ 不能包含 |
| 普通字段 | ✅ 可以更新 |

**注意事项：**
- **必须**先扫描 DDL 文件确定表的所有唯一索引字段
- UPDATE 子句中只能包含**非唯一索引**的普通字段
- 如果所有需要更新的字段都是唯一索引，需要改用其他方案（如先 DELETE 再 INSERT）

## 4. 日期时间函数转换（重要）

### 4.1 获取当前时间

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `NOW()` | `NOW()` | 兼容，无需修改 |
| `CURDATE()` | `CURRENT_DATE` | 获取当前日期 |
| `CURTIME()` | `CURRENT_TIME` | 获取当前时间 |
| `SYSDATE()` | `NOW()` | 系统时间 |

### 4.2 时间戳转换

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` | 当前时间戳 |
| `UNIX_TIMESTAMP(date)` | `EXTRACT(EPOCH FROM date)::INTEGER` | 日期转时间戳 |
| `FROM_UNIXTIME(ts)` | `TO_TIMESTAMP(ts)` | 时间戳转日期 |

### 4.3 日期格式化

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` | 格式化日期 |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:%s')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:SS')` | 格式化日期时间 |
| `DATE_FORMAT(d, '%Y-%m-%d %H:00:00')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:00:00')` | 格式化到小时 |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:00')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:00')` | 格式化到分钟 |
| `DATE_FORMAT(d, '%Y%m%d')` | `TO_CHAR(d, 'YYYYMMDD')` | 无分隔符 |
| `DATE_FORMAT(d, '%H:%i:%s')` | `TO_CHAR(d, 'HH24:MI:SS')` | 格式化时间 |
| `STR_TO_DATE(str, '%Y-%m-%d')` | `TO_DATE(str, 'YYYY-MM-DD')` | 字符串转日期 |
| `STR_TO_DATE(str, '%Y-%m-%d %H:%i:%s')` | `TO_TIMESTAMP(str, 'YYYY-MM-DD HH24:MI:SS')` | 字符串转时间戳 |

**DATE_FORMAT 格式符对照：**

| MySQL | GaussDB | 含义 |
|-------|---------|------|
| `%Y` | `YYYY` | 4位年份 |
| `%y` | `YY` | 2位年份 |
| `%m` | `MM` | 月份(01-12) |
| `%d` | `DD` | 日(01-31) |
| `%H` | `HH24` | 小时(00-23) |
| `%h` | `HH12` | 小时(01-12) |
| `%i` | `MI` | 分钟(00-59) |
| `%s` | `SS` | 秒(00-59) |
| `%W` | `DAY` | 星期名称 |
| `%w` | `D` | 星期(0-6) |

### 4.4 日期加减

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `DATE_ADD(date, INTERVAL n DAY)` | `TO_CHAR(CAST(date AS TIMESTAMP) + INTERVAL 'n DAY', 'YYYY-MM-DD')` | 加天数 |
| `DATE_ADD(date, INTERVAL n MONTH)` | `TO_CHAR(CAST(date AS TIMESTAMP) + INTERVAL 'n MONTH', 'YYYY-MM-DD')` | 加月数 |
| `DATE_ADD(date, INTERVAL n YEAR)` | `TO_CHAR(CAST(date AS TIMESTAMP) + INTERVAL 'n YEAR', 'YYYY-MM-DD')` | 加年数 |
| `DATE_SUB(date, INTERVAL n DAY)` | `TO_CHAR(CAST(date AS TIMESTAMP) - INTERVAL 'n DAY', 'YYYY-MM-DD')` | 减天数 |
| `DATE_SUB(date, INTERVAL n MONTH)` | `TO_CHAR(CAST(date AS TIMESTAMP) - INTERVAL 'n MONTH', 'YYYY-MM-DD')` | 减月数 |
| `DATE_SUB(date, INTERVAL n YEAR)` | `TO_CHAR(CAST(date AS TIMESTAMP) - INTERVAL 'n YEAR', 'YYYY-MM-DD')` | 减年数 |
| `date + INTERVAL n DAY` | `date + INTERVAL 'n DAY'` | 简写加法 |
| `date - INTERVAL n DAY` | `date - INTERVAL 'n DAY'` | 简写减法 |

### 4.5 日期差值计算

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `DATEDIFF(date1, date2)` | `(date1::date - date2::date)` | 日期差（天） |
| `TIMESTAMPDIFF(DAY, start, end)` | `EXTRACT(DAY FROM (end - start))` | 时间差（天） |
| `TIMESTAMPDIFF(HOUR, start, end)` | `EXTRACT(EPOCH FROM (end - start))::INTEGER / 3600` | 时间差（小时） |
| `TIMESTAMPDIFF(MINUTE, start, end)` | `EXTRACT(EPOCH FROM (end - start))::INTEGER / 60` | 时间差（分钟） |
| `TIMESTAMPDIFF(SECOND, start, end)` | `EXTRACT(EPOCH FROM (end - start))::INTEGER` | 时间差（秒） |

### 4.6 提取日期部分

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `DATE(datetime)` | `TO_CHAR(datetime, 'YYYY-MM-DD')` | 提取日期 |
| `TIME(datetime)` | `datetime::time` 或 `CAST(datetime AS TIME)` | 提取时间 |
| `YEAR(date)` | `EXTRACT(YEAR FROM date)` | 提取年份 |
| `MONTH(date)` | `EXTRACT(MONTH FROM date)` | 提取月份 |
| `DAY(date)` | `EXTRACT(DAY FROM date)` | 提取日 |
| `HOUR(time)` | `EXTRACT(HOUR FROM time)` | 提取小时 |
| `MINUTE(time)` | `EXTRACT(MINUTE FROM time)` | 提取分钟 |
| `SECOND(time)` | `EXTRACT(SECOND FROM time)` | 提取秒 |
| `DAYOFWEEK(date)` | `EXTRACT(DOW FROM date) + 1` | 星期几(1-7) |
| `DAYOFMONTH(date)` | `EXTRACT(DAY FROM date)` | 月中第几天 |
| `DAYOFYEAR(date)` | `EXTRACT(DOY FROM date)` | 年中第几天 |

### 4.7 日期函数示例

```sql
-- MySQL
SELECT
    DATE_FORMAT(create_time, '%Y-%m-%d') AS date_str,
    DATE_ADD(create_time, INTERVAL 7 DAY) AS next_week,
    TIMESTAMPDIFF(DAY, create_time, NOW()) AS days_ago,
    YEAR(create_time) AS year
FROM orders;

-- GaussDB
SELECT
    TO_CHAR(create_time, 'YYYY-MM-DD') AS date_str,
    TO_CHAR(CAST(create_time AS TIMESTAMP) + INTERVAL '7 DAY', 'YYYY-MM-DD') AS next_week,
    EXTRACT(DAY FROM (NOW() - create_time)) AS days_ago,
    EXTRACT(YEAR FROM create_time) AS year
FROM orders;
```

## 5. GROUP BY 非聚合列处理

GaussDB 严格遵循 SQL 标准，SELECT 中的非聚合列必须出现在 GROUP BY 中或使用聚合函数：

```sql
-- MySQL（允许，ONLY_FULL_GROUP_BY 关闭时）
SELECT user_id, username, department, COUNT(*) AS order_count
FROM orders GROUP BY user_id;

-- GaussDB（必须处理非聚合列）
SELECT user_id, MAX(username) AS username, MAX(department) AS department, COUNT(*) AS order_count
FROM orders GROUP BY user_id;
```

## 6. ORDER BY 排序 NULL 值处理

GaussDB 中 NULL 值的默认排序行为与 MySQL 不同，需要显式指定：

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `ORDER BY col` | `ORDER BY col NULLS FIRST` | 升序（默认），NULL 排前面 |
| `ORDER BY col ASC` | `ORDER BY col ASC NULLS FIRST` | 升序，NULL 排前面 |
| `ORDER BY col DESC` | `ORDER BY col DESC NULLS LAST` | 降序，NULL 排后面 |

**示例：**

```sql
-- MySQL
SELECT * FROM user ORDER BY age;
SELECT * FROM user ORDER BY age ASC;
SELECT * FROM user ORDER BY create_time DESC;
SELECT * FROM user ORDER BY status ASC, create_time DESC;

-- GaussDB
SELECT * FROM user ORDER BY age NULLS FIRST;
SELECT * FROM user ORDER BY age ASC NULLS FIRST;
SELECT * FROM user ORDER BY create_time DESC NULLS LAST;
SELECT * FROM user ORDER BY status ASC NULLS FIRST, create_time DESC NULLS LAST;
```

## 7. INSERT/UPDATE 类型转换

GaussDB 对类型匹配更严格，**只有当 Java 类中使用 String 类型传递给数据库的 INT/JSON 字段时**，才需要显式类型转换。

**重要：需要同时扫描 DDL 文件和 Java 类，确定哪些字段需要转换！**

### 步骤 1：扫描 SQL 文件，查找 INT 和 JSON 类型字段

```bash
# 查找 INT 类型字段（包括 INT, INTEGER, BIGINT, SMALLINT, TINYINT）
grep -rnE "\b(INT|INTEGER|BIGINT|SMALLINT|TINYINT|int2|int4|int8)\b" --include="*.sql"

# 查找 JSON 类型字段
grep -rn "\bJSON\b" --include="*.sql"
```

### 步骤 2：扫描 Java 类，查找参数类型

```bash
# 查找实体类/DTO 中的 String 类型字段
grep -rn "private String" --include="*.java"

# 查找实体类/DTO 中的数值类型字段（这些不需要转换）
grep -rnE "private (Integer|Long|int|long)" --include="*.java"
```

### 步骤 3：对比数据库字段类型和 Java 参数类型

```java
// Java 实体类示例
public class User {
    private Long id;           // Long 类型 → 数据库 BIGINT，类型匹配，不需要转换
    private Integer age;       // Integer 类型 → 数据库 INTEGER，类型匹配，不需要转换
    private String score;      // ⚠️ String 类型 → 数据库 INT，需要 ::int 转换
    private String level;      // ⚠️ String 类型 → 数据库 SMALLINT，需要 ::int2 转换
    private String name;       // String 类型 → 数据库 VARCHAR，类型匹配，不需要转换
    private String settings;   // ⚠️ String 类型 → 数据库 JSON，需要 ::json 转换
    private String metadata;   // ⚠️ String 类型 → 数据库 JSON，需要 ::json 转换
}
```

### 步骤 4：建立类型转换映射表（转换前先填写）

| 表名 | 字段名 | 数据库类型 | Java 类型 | 是否需要转换 |
|------|--------|-----------|-----------|-------------|
| user | id | BIGINT | Long | ❌ 不需要 |
| user | age | INTEGER | Integer | ❌ 不需要 |
| user | score | INT | String | ✅ `::int` |
| user | level | SMALLINT | String | ✅ `::int2` |
| user | name | VARCHAR | String | ❌ 不需要 |
| user | settings | JSON | String | ✅ `::json` |
| user | metadata | JSON | String | ✅ `::json` |

### 步骤 5：转换 INSERT/UPDATE 语句

**只对 Java 类型为 String 且数据库类型为 INT/JSON 的字段添加转换：**

```sql
-- MySQL 原始语句
INSERT INTO user(id, age, score, level, name, settings, metadata)
VALUES(#{id}, #{age}, #{score}, #{level}, #{name}, #{settings}, #{metadata});

UPDATE user SET
    age = #{age},
    score = #{score},
    level = #{level},
    name = #{name},
    settings = #{settings},
    metadata = #{metadata}
WHERE id = #{id};

-- GaussDB 转换后（只转换 Java 为 String 且数据库为 INT/JSON 的字段）
INSERT INTO user(id, age, score, level, name, settings, metadata)
VALUES(#{id}, #{age}, #{score}::int, #{level}::int2, #{name}, #{settings}::json, #{metadata}::json);
--     ↑ Long不转换  ↑ Integer不转换  ↑ String→INT  ↑ String→SMALLINT  ↑ String不转换  ↑ String→JSON

UPDATE user SET
    age = #{age},                  -- Integer 类型，不需要转换
    score = #{score}::int,         -- String → INT，需要转换
    level = #{level}::int2,        -- String → SMALLINT，需要转换
    name = #{name},                -- String → VARCHAR，不需要转换
    settings = #{settings}::json,  -- String → JSON，需要转换
    metadata = #{metadata}::json   -- String → JSON，需要转换
WHERE id = #{id};                  -- WHERE 子句不需要类型转换
```

### 7.1 转换规则总结

**只有当 Java 类型为 String，且数据库类型为 INT/JSON 时才需要转换：**

| 数据库字段类型 | Java 类型 | 是否需要转换 | GaussDB 转换 |
|---------------|----------|-------------|-------------|
| INT / INTEGER | Integer | ❌ 不需要 | - |
| INT / INTEGER | String | ✅ 需要 | `#{value}::int` |
| BIGINT | Long | ❌ 不需要 | - |
| BIGINT | String | ✅ 需要 | `#{value}::int` |
| SMALLINT | Short / Integer | ❌ 不需要 | - |
| SMALLINT | String | ✅ 需要 | `#{value}::int2` |
| JSON | String | ✅ 需要 | `#{value}::json` |
| VARCHAR / TEXT | String | ❌ 不需要 | - |
| TIMESTAMP / DATE | Date / LocalDateTime | ❌ 不需要 | - |

### 7.2 注意事项

- **必须**同时扫描 DDL 文件和 Java 类，确定字段类型和参数类型
- **只有 Java String 类型传给 INT/JSON 字段时才需要转换**
- **Java Integer/Long 等数值类型传给 INT 字段，不需要转换**
- **只转换 INSERT VALUES 和 UPDATE SET 中的输入值**
- **WHERE 子句中的字段不需要类型转换**
- 可通过数据库命令 `\d 表名` 查看表结构确认字段类型

## 8. 兼容的 MySQL 语法（无需转换）

以下语法 GaussDB 直接兼容：
- `LIMIT offset, count` - 分页语法
- `LIMIT count` - 限制条数
- `NOW()` - 当前时间
- `COALESCE()` - 空值处理
- `CONCAT()` - 字符串拼接
- `TRIM()`, `UPPER()`, `LOWER()` - 字符串函数
- `ABS()`, `ROUND()`, `CEIL()`, `FLOOR()` - 数学函数

## 9. 注意事项

**XML 转义字符保留：**
- `&lt;` 不需要改成 `<`
- `&gt;` 不需要改成 `>`
- `&amp;` 不需要改成 `&`

这些是 XML 的标准转义，在 MyBatis Mapper XML 中用于比较运算符，必须保留。

## 10. 迁移完整性校验

转换完成后，执行以下检查确保没有遗漏。

### 10.1 扫描残留 MySQL 语法

```bash
# ========== 1. 标识符和字符串检查 ==========
# 检查反引号（MySQL 特有，应直接去掉，不加双引号）
grep -rn "\`" --include="*.xml" --include="*.java"

# 检查双引号字符串值（GaussDB 双引号仅用于别名，字符串应用单引号）
grep -rnE "=\s*\"[^\"]+\"" --include="*.xml" --include="*.java"

# 检查字段别名是否加了双引号（需人工确认）
# AS 后面的别名必须加双引号以保持大小写
grep -rnE "\bAS\s+[a-zA-Z_][a-zA-Z0-9_]*\s*[,\s\n\r\)]" --include="*.xml" --include="*.java"

# ========== 2. 通用函数检查 ==========
# 检查 IFNULL（应转为 COALESCE）
grep -rn "IFNULL" --include="*.xml" --include="*.java"

# 检查 IF(（应转为 CASE WHEN）
grep -rnE "\bIF\s*\(" --include="*.xml" --include="*.java"

# 检查 GROUP_CONCAT（应转为 STRING_AGG）
grep -rn "GROUP_CONCAT" --include="*.xml" --include="*.java"

# 检查 JSON_OBJECT（应转为 json_build_object）
grep -rn "JSON_OBJECT" --include="*.xml" --include="*.java"

# 检查 JSON_CONTAINS（应转为 ::jsonb @> ::jsonb）
grep -rn "JSON_CONTAINS" --include="*.xml" --include="*.java"

# 检查 ANY_VALUE（应转为 MAX）
grep -rn "ANY_VALUE" --include="*.xml" --include="*.java"

# 检查 LAST_INSERT_ID（应转为 currval('序列名')，需扫描 DDL 获取真实序列名）
grep -rn "LAST_INSERT_ID" --include="*.xml" --include="*.java"

# ========== 3. 日期函数检查（重点） ==========
# 检查 DATE_FORMAT（应转为 TO_CHAR）
grep -rn "DATE_FORMAT" --include="*.xml" --include="*.java"

# 检查 STR_TO_DATE（应转为 TO_DATE/TO_TIMESTAMP）
grep -rn "STR_TO_DATE" --include="*.xml" --include="*.java"

# 检查 UNIX_TIMESTAMP（应转为 EXTRACT(EPOCH FROM ...)）
grep -rn "UNIX_TIMESTAMP" --include="*.xml" --include="*.java"

# 检查 FROM_UNIXTIME（应转为 TO_TIMESTAMP）
grep -rn "FROM_UNIXTIME" --include="*.xml" --include="*.java"

# 检查 CURDATE（应转为 CURRENT_DATE）
grep -rn "CURDATE" --include="*.xml" --include="*.java"

# 检查 CURTIME（应转为 CURRENT_TIME）
grep -rn "CURTIME" --include="*.xml" --include="*.java"

# 检查 SYSDATE（应转为 NOW()）
grep -rn "SYSDATE" --include="*.xml" --include="*.java"

# 检查 DATE_ADD/DATE_SUB（应转为 INTERVAL 运算）
grep -rnE "DATE_ADD|DATE_SUB" --include="*.xml" --include="*.java"

# 检查 DATEDIFF（应转为 date1::date - date2::date）
grep -rn "DATEDIFF" --include="*.xml" --include="*.java"

# 检查 TIMESTAMPDIFF（应转为 EXTRACT）
grep -rn "TIMESTAMPDIFF" --include="*.xml" --include="*.java"

# 检查 DATE(（应转为 TO_CHAR(..., 'YYYY-MM-DD')）
grep -rnE "\bDATE\s*\(" --include="*.xml" --include="*.java"

# 检查 TIME(（应转为 ::time）
grep -rnE "\bTIME\s*\(" --include="*.xml" --include="*.java"

# 检查日期提取函数 YEAR/MONTH/DAY/HOUR/MINUTE/SECOND（应转为 EXTRACT）
grep -rnE "\b(YEAR|MONTH|DAY|HOUR|MINUTE|SECOND)\s*\(" --include="*.xml" --include="*.java"

# 检查 DAYOFWEEK/DAYOFMONTH/DAYOFYEAR（应转为 EXTRACT(DOW/DAY/DOY FROM ...)）
grep -rnE "\bDAYOF(WEEK|MONTH|YEAR)\s*\(" --include="*.xml" --include="*.java"

# ========== 4. ORDER BY 检查 ==========
# 检查 ORDER BY 是否添加了 NULLS FIRST/LAST（需人工确认）
grep -rnE "ORDER\s+BY" --include="*.xml" --include="*.java"

# ========== 5. GROUP BY 检查 ==========
# 检查 GROUP BY 语句，确认非聚合列已用聚合函数包装（需人工确认）
grep -rnE "GROUP\s+BY" --include="*.xml" --include="*.java"

# ========== 6. INSERT/UPDATE 类型转换检查 ==========
# 需同时扫描 Java 类确认参数类型，只有 Java String 传给 INT/JSON 字段才需要转换

# 扫描 Java 类中的 String 类型字段
grep -rn "private String" --include="*.java"

# 扫描 DDL 中的 INT 类型字段
grep -rnE "\b(INT|INTEGER|BIGINT|SMALLINT)\b" --include="*.sql"

# 扫描 DDL 中的 JSON 类型字段
grep -rn "\bJSON\b" --include="*.sql"

# 检查 INSERT 语句（需人工对照 Java 类型和 DDL 类型）
grep -rnE "INSERT\s+INTO" --include="*.xml" --include="*.java"

# 检查 UPDATE 语句（需人工对照 Java 类型和 DDL 类型）
grep -rnE "UPDATE\s+\w+\s+SET" --include="*.xml" --include="*.java"

# ========== 7. 需人工处理的语法 ==========
# 检查 ON DUPLICATE KEY UPDATE（UPDATE 子句中不能包含唯一索引字段，需扫描 DDL 确认）
grep -rn "ON DUPLICATE KEY" --include="*.xml" --include="*.java"

# 扫描唯一索引定义
grep -rn "UNIQUE" --include="*.sql"
grep -rn "PRIMARY KEY" --include="*.sql"

# 检查 SELECT EXISTS（如需返回 0/1 需转为 (EXISTS(...))::int）
grep -rnE "SELECT\s+EXISTS" --include="*.xml" --include="*.java"
```

### 10.2 校验清单

#### 必须转换项（期望：无匹配）

**标识符和字符串：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| 反引号 | `grep -rn "\`"` | 人工确认（反引号需改为双引号） |
| 双引号字符串 | `grep -rnE "=\s*\"[^\"]+\""` | 人工确认（字符串值需改为单引号） |

**通用函数：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| IFNULL | `grep -rn "IFNULL"` | 无匹配 |
| IF( | `grep -rnE "\bIF\s*\("` | 无匹配 |
| GROUP_CONCAT | `grep -rn "GROUP_CONCAT"` | 无匹配 |
| JSON_OBJECT | `grep -rn "JSON_OBJECT"` | 无匹配 |
| JSON_CONTAINS | `grep -rn "JSON_CONTAINS"` | 无匹配 |
| ANY_VALUE | `grep -rn "ANY_VALUE"` | 无匹配 |
| LAST_INSERT_ID | `grep -rn "LAST_INSERT_ID"` | 无匹配 |

**日期时间函数：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| DATE_FORMAT | `grep -rn "DATE_FORMAT"` | 无匹配 |
| STR_TO_DATE | `grep -rn "STR_TO_DATE"` | 无匹配 |
| UNIX_TIMESTAMP | `grep -rn "UNIX_TIMESTAMP"` | 无匹配 |
| FROM_UNIXTIME | `grep -rn "FROM_UNIXTIME"` | 无匹配 |
| CURDATE | `grep -rn "CURDATE"` | 无匹配 |
| CURTIME | `grep -rn "CURTIME"` | 无匹配 |
| SYSDATE | `grep -rn "SYSDATE"` | 无匹配 |
| DATE_ADD | `grep -rn "DATE_ADD"` | 无匹配 |
| DATE_SUB | `grep -rn "DATE_SUB"` | 无匹配 |
| DATEDIFF | `grep -rn "DATEDIFF"` | 无匹配 |
| TIMESTAMPDIFF | `grep -rn "TIMESTAMPDIFF"` | 无匹配 |
| DATE( | `grep -rnE "\bDATE\s*\("` | 无匹配 |
| TIME( | `grep -rnE "\bTIME\s*\("` | 无匹配 |
| YEAR/MONTH/DAY | `grep -rnE "\b(YEAR\|MONTH\|DAY)\s*\("` | 无匹配 |
| HOUR/MINUTE/SECOND | `grep -rnE "\b(HOUR\|MINUTE\|SECOND)\s*\("` | 无匹配 |
| DAYOFWEEK/MONTH/YEAR | `grep -rnE "\bDAYOF(WEEK\|MONTH\|YEAR)\s*\("` | 无匹配 |

#### 需人工检查项

| 检查项 | 命令 | 检查要点 |
|--------|------|----------|
| 字段别名 | `grep -rnE "\bAS\s+[a-zA-Z_][a-zA-Z0-9_]*\s*[,\s)]"` | 需检查 `<select>` 的 resultType，只有 Map 类型需加双引号 |
| resultType="map" | `grep -rn 'resultType="map"' --include="*.xml"` | 辅助检查：查找 map 类型的查询 |
| resultType="HashMap" | `grep -rn 'resultType="java.util.HashMap"' --include="*.xml"` | 辅助检查：查找 HashMap 类型的查询 |
| LAST_INSERT_ID | `grep -rn "LAST_INSERT_ID"` | 需扫描 DDL 获取真实序列名，转为 `currval('序列名')` |
| ON DUPLICATE KEY | `grep -rn "ON DUPLICATE KEY"` | UPDATE 子句中不能包含唯一索引字段，需扫描 DDL 确认 |
| ORDER BY | `grep -rnE "ORDER\s+BY"` | 确认已添加 NULLS FIRST/LAST |
| GROUP BY | `grep -rnE "GROUP\s+BY"` | 确认非聚合列已用 MAX() 等函数包装 |
| SELECT EXISTS | `grep -rnE "SELECT\s+EXISTS"` | 如需返回 0/1 需转为 `(EXISTS(...))::int` |
| INSERT 语句 | `grep -rnE "INSERT\s+INTO"` | 需对照 Java 类，只有 Java String 传给 INT/JSON 才加 `::int`/`::json` |
| UPDATE 语句 | `grep -rnE "UPDATE\s+\w+\s+SET"` | 需对照 Java 类，只有 Java String 传给 INT/JSON 才加 `::int`/`::json`（WHERE 不转换） |

### 10.3 生成校验报告

扫描完成后，将报告输出到项目根目录下的 `dml_完整性校验报告.txt` 文件：

```
============ DML 迁移校验报告 ============

✅ 标识符和字符串
   - 反引号已改为双引号（有反引号的改为双引号）
   - 无反引号的标识符保持原样
   - SQL 关键字字段已用双引号包裹
   - 字符串值已使用单引号
   - 字段别名（需检查 resultType，只有 Map 类型加双引号）

✅ 通用函数
   - IFNULL 已转为 COALESCE
   - IF() 已转为 CASE WHEN
   - GROUP_CONCAT 已转为 STRING_AGG
   - JSON_OBJECT 已转为 json_build_object
   - JSON_CONTAINS 已转为 ::jsonb @> ::jsonb
   - ANY_VALUE 已转为 MAX

✅ 日期时间函数
   - DATE_FORMAT 已转为 TO_CHAR
   - STR_TO_DATE 已转为 TO_DATE/TO_TIMESTAMP
   - UNIX_TIMESTAMP 已转为 EXTRACT(EPOCH FROM ...)
   - FROM_UNIXTIME 已转为 TO_TIMESTAMP
   - CURDATE 已转为 CURRENT_DATE
   - CURTIME 已转为 CURRENT_TIME
   - SYSDATE 已转为 NOW()
   - DATE_ADD/DATE_SUB 已转为 INTERVAL 运算
   - DATEDIFF 已转为 date1::date - date2::date
   - TIMESTAMPDIFF 已转为 EXTRACT
   - DATE() 已转为 TO_CHAR(..., 'YYYY-MM-DD')
   - TIME() 已转为 ::time
   - YEAR/MONTH/DAY/HOUR/MINUTE/SECOND 已转为 EXTRACT

✅ 类型转换（需对照 Java 类）
   - Java String → DB INT 字段已添加 ::int
   - Java String → DB JSON 字段已添加 ::json
   - Java Integer/Long → DB INT 字段无需转换
   - WHERE 子句无需类型转换

✅ 特殊语法
   - LAST_INSERT_ID 已转为 currval('真实序列名')
   - ON DUPLICATE KEY UPDATE 子句中不包含唯一索引字段

⚠️ 待人工检查项（如有）
   - 字段别名：需检查 <select> 的 resultType，只有 Map 类型加双引号
   - ORDER BY 语句：确认已添加 NULLS FIRST/LAST
   - GROUP BY 语句：确认非聚合列已用 MAX() 包装
   - SELECT EXISTS：如需返回 0/1 需转为 (EXISTS(...))::int
   - LAST_INSERT_ID：需扫描 DDL 确认真实序列名
   - ON DUPLICATE KEY：需扫描 DDL 确认唯一索引字段
   - INSERT/UPDATE：需对照 Java 类确认哪些字段需要 ::int/::json
   - UserMapper.xml:45 - 发现 xxx（示例）

❌ 未通过项（如有）
   - OrderMapper.xml:78 - 仍存在 DATE_FORMAT
   - ConfigMapper.xml:23 - 仍存在 IFNULL

============ 统计 ============
扫描文件数: XX 个
已转换项: XX 处
待人工检查: XX 处
未通过项: XX 处
```

## 11. 完整转换示例

```xml
<!-- ========== MySQL 原始 Mapper ========== -->
<select id="getUserStats" resultType="map">
    SELECT
        `user_id`,
        `username`,
        DATE_FORMAT(`create_time`, '%Y-%m-%d') AS create_date,
        IFNULL(`nickname`, `username`) AS display_name,
        IF(`status` = 1, '启用', '禁用') AS status_text,
        TIMESTAMPDIFF(DAY, `create_time`, NOW()) AS days_since_created,
        (SELECT EXISTS(SELECT 1 FROM orders WHERE user_id = u.id)) AS has_orders
    FROM `user` u
    WHERE `status` = 1
      AND `create_time` &gt;= DATE_SUB(NOW(), INTERVAL 30 DAY)
</select>

<!-- ========== GaussDB 转换后 Mapper ========== -->
<!-- 反引号改为双引号，resultType="map" 时别名加双引号 -->
<select id="getUserStats" resultType="map">
    SELECT
        "user_id",
        "username",
        TO_CHAR("create_time", 'YYYY-MM-DD') AS "createDate",
        COALESCE("nickname", "username") AS "displayName",
        CASE WHEN "status" = 1 THEN '启用' ELSE '禁用' END AS "statusText",
        EXTRACT(DAY FROM (NOW() - "create_time")) AS "daysSinceCreated",
        (SELECT (EXISTS(SELECT 1 FROM orders WHERE user_id = u.id))::int) AS "hasOrders"
    FROM "user" u
    WHERE "status" = 1
      AND "create_time" &gt;= TO_CHAR(CAST(NOW() AS TIMESTAMP) - INTERVAL '30 DAY', 'YYYY-MM-DD')
</select>
```

### INSERT/UPDATE 类型转换示例

**步骤 1：扫描 DDL 获取数据库字段类型**
```sql
-- 从 schema.sql 扫描到的 config 表结构
CREATE TABLE config (
    id BIGINT PRIMARY KEY,       -- 数据库 INT 类型
    user_id INTEGER,             -- 数据库 INT 类型
    age INTEGER,                 -- 数据库 INT 类型
    score INT,                   -- 数据库 INT 类型
    name NVARCHAR2(50),          -- 数据库字符串类型
    settings JSON,               -- 数据库 JSON 类型
    metadata JSON                -- 数据库 JSON 类型
);
```

**步骤 2：扫描 Java 类获取参数类型**
```java
// ConfigDTO.java
public class ConfigDTO {
    private Long userId;         // Long 类型 → 数据库 INTEGER，不需要转换
    private Integer age;         // Integer 类型 → 数据库 INTEGER，不需要转换
    private String score;        // ⚠️ String 类型 → 数据库 INT，需要 ::int
    private String name;         // String 类型 → 数据库 VARCHAR，不需要转换
    private String settings;     // ⚠️ String 类型 → 数据库 JSON，需要 ::json
    private String metadata;     // ⚠️ String 类型 → 数据库 JSON，需要 ::json
}
```

**步骤 3：根据 Java 类型决定是否转换**
```xml
<!-- ========== MySQL 原始 Mapper ========== -->
<insert id="insertConfig">
    INSERT INTO config(user_id, age, score, name, settings, metadata)
    VALUES(#{userId}, #{age}, #{score}, #{name}, #{settings}, #{metadata})
</insert>

<update id="updateConfig">
    UPDATE config
    SET age = #{age},
        score = #{score},
        name = #{name},
        settings = #{settings},
        metadata = #{metadata}
    WHERE user_id = #{userId}
</update>

<!-- ========== GaussDB 转换后 Mapper ========== -->
<!-- 只转换 Java String 类型传给 INT/JSON 字段的情况 -->
<insert id="insertConfig">
    INSERT INTO config(user_id, age, score, name, settings, metadata)
    VALUES(#{userId}, #{age}, #{score}::int, #{name}, #{settings}::json, #{metadata}::json)
</insert>
<!--   ↑ Long不转换  ↑ Integer不转换  ↑ String→INT  ↑ String不转换  ↑ String→JSON -->

<update id="updateConfig">
    UPDATE config
    SET age = #{age},                  -- Integer 类型，不需要转换
        score = #{score}::int,         -- String → INT，需要转换
        name = #{name},                -- String → VARCHAR，不需要转换
        settings = #{settings}::json,  -- String → JSON，需要转换
        metadata = #{metadata}::json   -- String → JSON，需要转换
    WHERE user_id = #{userId}          -- WHERE 子句不需要类型转换
</update>
```
