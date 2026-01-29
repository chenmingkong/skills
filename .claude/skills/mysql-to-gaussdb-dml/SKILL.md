---
name: mysql-to-gaussdb-dml
description: 将 MySQL DML 语句转换为 GaussDB 兼容语法（查询、函数、日期处理等）
argument-hint: "[Mapper文件或目录]"
---

# MySQL to GaussDB DML 转换

将 MySQL 的 DML（数据操作语言）转换为 GaussDB 兼容语法，包括 SELECT、INSERT、UPDATE、DELETE 语句中的函数和语法。

## 扫描文件

```
**/*Mapper.xml                        # MyBatis 映射文件
**/*Repository.java, **/*Dao.java    # @Query 原生 SQL
**/*.java                            # 其他 Java 文件中的 SQL
```

**重要：必须对所有匹配的文件进行完整扫描，逐行检查并转换，确保文件从头到尾每一行都经过处理，不要遗漏文件后半部分的内容。**

## 1. 标识符引号转换（表名、字段名）

MySQL 使用反引号 `` ` `` 包裹表名和字段名，GaussDB 使用双引号 `"`：

```sql
-- MySQL 写法（有反引号）
SELECT `id`, `user_name` FROM `user` WHERE `status` = 1;
SELECT `User`.`Name` FROM `User`;

-- GaussDB 写法（反引号替换为双引号，标识符转小写）
SELECT "id", "user_name" FROM "user" WHERE "status" = 1;
SELECT "user"."name" FROM "user";

-- MySQL 写法（无反引号）
SELECT id, user_name FROM user WHERE status = 1;

-- GaussDB 写法（保持原样，不加双引号）
SELECT id, user_name FROM user WHERE status = 1;
```

**转换规则：**
- `` `identifier` `` → `"identifier"` （反引号替换为双引号）
- `` `TableName` `` → `"tablename"` （大写必须转为小写）
- `` `table`.`column` `` → `"table"."column"` （表名.字段名同样处理）
- `identifier` → `identifier` （无反引号则保持原样，不加双引号）

## 2. 字符串值引号转换

GaussDB 中双引号用于标识符，字符串必须用单引号：

```sql
-- MySQL（双引号和单引号都可用于字符串）
SELECT * FROM user WHERE name = "张三";
INSERT INTO user(name) VALUES("李四");

-- GaussDB（字符串必须用单引号）
SELECT * FROM user WHERE name = '张三';
INSERT INTO user(name) VALUES('李四');
```

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

## 7. 兼容的 MySQL 语法（无需转换）

以下语法 GaussDB 直接兼容：
- `LIMIT offset, count` - 分页语法
- `LIMIT count` - 限制条数
- `NOW()` - 当前时间
- `COALESCE()` - 空值处理
- `CONCAT()` - 字符串拼接
- `TRIM()`, `UPPER()`, `LOWER()` - 字符串函数
- `ABS()`, `ROUND()`, `CEIL()`, `FLOOR()` - 数学函数

## 8. 注意事项

**XML 转义字符保留：**
- `&lt;` 不需要改成 `<`
- `&gt;` 不需要改成 `>`
- `&amp;` 不需要改成 `&`

这些是 XML 的标准转义，在 MyBatis Mapper XML 中用于比较运算符，必须保留。

## 9. 迁移完整性校验

转换完成后，执行以下检查确保没有遗漏。

### 10.1 扫描残留 MySQL 语法

```bash
# ========== 标识符和字符串检查 ==========
# 检查反引号（MySQL 特有）
grep -r "\`" --include="*.xml" --include="*.java"

# 检查双引号字符串值（GaussDB 双引号仅用于标识符）
grep -rE "=\s*\"[^\"]+\"" --include="*.xml" --include="*.java"

# ========== 通用函数检查 ==========
# 检查 IFNULL（应为 COALESCE）
grep -r "IFNULL" --include="*.xml" --include="*.java"

# 检查 IF(（应为 CASE WHEN）
grep -rE "\bIF\s*\(" --include="*.xml" --include="*.java"

# 检查 GROUP_CONCAT（应为 STRING_AGG）
grep -r "GROUP_CONCAT" --include="*.xml" --include="*.java"

# 检查 JSON_OBJECT（应为 json_build_object）
grep -r "JSON_OBJECT" --include="*.xml" --include="*.java"

# 检查 JSON_CONTAINS（应为 ::jsonb @> ::jsonb）
grep -r "JSON_CONTAINS" --include="*.xml" --include="*.java"

# 检查 ANY_VALUE（应为 MAX）
grep -r "ANY_VALUE" --include="*.xml" --include="*.java"

# ========== 日期函数检查（重点） ==========
# 检查 DATE_FORMAT（应为 TO_CHAR）
grep -r "DATE_FORMAT" --include="*.xml" --include="*.java"

# 检查 STR_TO_DATE（应为 TO_DATE/TO_TIMESTAMP）
grep -r "STR_TO_DATE" --include="*.xml" --include="*.java"

# 检查 UNIX_TIMESTAMP
grep -r "UNIX_TIMESTAMP" --include="*.xml" --include="*.java"

# 检查 FROM_UNIXTIME（应为 TO_TIMESTAMP）
grep -r "FROM_UNIXTIME" --include="*.xml" --include="*.java"

# 检查 CURDATE（应为 CURRENT_DATE）
grep -r "CURDATE" --include="*.xml" --include="*.java"

# 检查 CURTIME（应为 CURRENT_TIME）
grep -r "CURTIME" --include="*.xml" --include="*.java"

# 检查 SYSDATE（应为 NOW）
grep -r "SYSDATE" --include="*.xml" --include="*.java"

# 检查 DATE_ADD/DATE_SUB
grep -rE "DATE_ADD|DATE_SUB" --include="*.xml" --include="*.java"

# 检查 DATEDIFF
grep -r "DATEDIFF" --include="*.xml" --include="*.java"

# 检查 TIMESTAMPDIFF
grep -r "TIMESTAMPDIFF" --include="*.xml" --include="*.java"

# 检查日期提取函数 YEAR/MONTH/DAY/HOUR/MINUTE/SECOND
grep -rE "\b(YEAR|MONTH|DAY|HOUR|MINUTE|SECOND)\s*\(" --include="*.xml" --include="*.java"

# 检查 DAYOFWEEK/DAYOFMONTH/DAYOFYEAR
grep -rE "\bDAYOF(WEEK|MONTH|YEAR)\s*\(" --include="*.xml" --include="*.java"

# ========== ORDER BY 检查 ==========
# 检查 ORDER BY 是否添加了 NULLS FIRST/LAST
grep -rE "ORDER\s+BY" --include="*.xml" --include="*.java"

# ========== 需人工检查 ==========
# 检查 GROUP BY 语句（确认非聚合列已处理）
grep -rE "GROUP\s+BY" --include="*.xml" --include="*.java"

# 检查 SELECT EXISTS（如需返回 0/1 需转换）
grep -rE "SELECT\s+EXISTS" --include="*.xml" --include="*.java"
```

### 10.2 校验清单

**标识符和字符串：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| 反引号 | `grep -r "\`"` | 无匹配 |
| 双引号字符串 | `grep -rE "=\s*\"[^\"]+\""` | 无匹配 |

**通用函数：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| IFNULL | `grep -r "IFNULL"` | 无匹配 |
| IF( | `grep -rE "\bIF\s*\("` | 无匹配 |
| GROUP_CONCAT | `grep -r "GROUP_CONCAT"` | 无匹配 |
| JSON_OBJECT | `grep -r "JSON_OBJECT"` | 无匹配 |
| JSON_CONTAINS | `grep -r "JSON_CONTAINS"` | 无匹配 |
| ANY_VALUE | `grep -r "ANY_VALUE"` | 无匹配 |

**日期时间函数：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| DATE_FORMAT | `grep -r "DATE_FORMAT"` | 无匹配 |
| STR_TO_DATE | `grep -r "STR_TO_DATE"` | 无匹配 |
| UNIX_TIMESTAMP | `grep -r "UNIX_TIMESTAMP"` | 无匹配 |
| FROM_UNIXTIME | `grep -r "FROM_UNIXTIME"` | 无匹配 |
| CURDATE | `grep -r "CURDATE"` | 无匹配 |
| CURTIME | `grep -r "CURTIME"` | 无匹配 |
| SYSDATE | `grep -r "SYSDATE"` | 无匹配 |
| DATE_ADD | `grep -r "DATE_ADD"` | 无匹配 |
| DATE_SUB | `grep -r "DATE_SUB"` | 无匹配 |
| DATEDIFF | `grep -r "DATEDIFF"` | 无匹配 |
| TIMESTAMPDIFF | `grep -r "TIMESTAMPDIFF"` | 无匹配 |
| YEAR/MONTH/DAY | `grep -rE "\b(YEAR\|MONTH\|DAY)\s*\("` | 无匹配 |
| HOUR/MINUTE/SECOND | `grep -rE "\b(HOUR\|MINUTE\|SECOND)\s*\("` | 无匹配 |
| DAYOFWEEK/DAYOFMONTH/DAYOFYEAR | `grep -rE "\bDAYOF(WEEK\|MONTH\|YEAR)\s*\("` | 无匹配 |

**需人工检查：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| ORDER BY | `grep -rE "ORDER\s+BY"` | 需人工检查 NULLS FIRST/LAST |
| GROUP BY | `grep -rE "GROUP\s+BY"` | 需人工检查非聚合列 |
| SELECT EXISTS | `grep -rE "SELECT\s+EXISTS"` | 需人工检查返回值 |

### 10.3 生成校验报告

扫描完成后，将报告输出到项目根目录下的 `dml_完整性校验报告.txt` 文件：

```
============ DML 迁移校验报告 ============

✅ 标识符和字符串
   - 反引号已替换为双引号
   - 字符串值已使用单引号

✅ 通用函数
   - IFNULL 已转为 COALESCE
   - IF() 已转为 CASE WHEN
   - GROUP_CONCAT 已转为 STRING_AGG
   - JSON_OBJECT 已转为 json_build_object

✅ 日期时间函数
   - DATE_FORMAT 已转为 TO_CHAR
   - STR_TO_DATE 已转为 TO_DATE/TO_TIMESTAMP
   - CURDATE 已转为 CURRENT_DATE
   - DATE_ADD/DATE_SUB 已转为 INTERVAL 运算
   - TIMESTAMPDIFF 已转为 EXTRACT
   - YEAR/MONTH/DAY 已转为 EXTRACT

⚠️ 待人工检查项（如有）
   - GROUP BY 语句：确认非聚合列已用 MAX() 包装
   - SELECT EXISTS：如需返回 0/1 需转为 (EXISTS(...))::int
   - UserMapper.xml:45 - 发现反引号
   - OrderMapper.xml:78 - 发现 DATE_FORMAT

❌ 未通过项（如有）
   - 仍存在 IFNULL 函数
   - 仍存在 DATE_FORMAT 函数

============ 统计 ============
扫描文件: XX 个
已转换: XX 处
待处理: XX 处
```

## 10. 完整转换示例

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
<select id="getUserStats" resultType="map">
    SELECT
        "user_id",
        "username",
        TO_CHAR("create_time", 'YYYY-MM-DD') AS create_date,
        COALESCE("nickname", "username") AS display_name,
        CASE WHEN "status" = 1 THEN '启用' ELSE '禁用' END AS status_text,
        EXTRACT(DAY FROM (NOW() - "create_time")) AS days_since_created,
        (SELECT (EXISTS(SELECT 1 FROM orders WHERE user_id = u.id))::int) AS has_orders
    FROM "user" u
    WHERE "status" = 1
      AND "create_time" &gt;= TO_CHAR(CAST(NOW() AS TIMESTAMP) - INTERVAL '30 DAY', 'YYYY-MM-DD')
</select>
```
