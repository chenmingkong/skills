---
name: mysql-to-gaussdb-dml-v2
description: 将 MySQL DML 语句转换为 GaussDB 兼容语法（优化版）
argument-hint: "[Mapper文件或目录]"
---

# MySQL to GaussDB DML 转换（v2）

## 快速参考表

### 引号转换
| 场景 | MySQL | GaussDB |
|------|-------|---------|
| 表名/字段名 | `` `name` `` | `name`（去掉反引号） |
| 字段别名 | `AS userName` | `AS "userName"`（加双引号） |
| 字符串值 | `"张三"` | `'张三'`（用单引号） |

### 常用函数转换
| MySQL | GaussDB |
|-------|---------|
| `IFNULL(a, b)` | `COALESCE(a, b)` |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` |
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` |
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` |
| `LAST_INSERT_ID()` | `currval('序列名')` |

### 类型转换（仅 Java String → DB INT/JSON）
| 数据库类型 | Java 类型 | 转换 |
|-----------|----------|------|
| INT/BIGINT | Integer/Long | 不需要 |
| INT/BIGINT | String | `#{val}::int` |
| JSON | String | `#{val}::json` |

---

## 扫描范围

```
**/*Mapper.xml          # MyBatis 映射文件
**/*Repository.java     # @Query 原生 SQL
**/*.java               # 其他 Java 文件中的 SQL
**/*.sql                # DDL 文件（用于获取字段类型）
```

**重要：必须完整扫描每个文件的所有行，不要遗漏！**

---

## 一、基础语法转换

### 1.1 标识符（表名、字段名）

**规则：去掉反引号，不加双引号**

```sql
-- MySQL
SELECT `id`, `user_name` FROM `user` WHERE `status` = 1;

-- GaussDB
SELECT id, user_name FROM user WHERE status = 1;
```

### 1.2 字段别名

**规则：别名必须加双引号**

```sql
-- MySQL
SELECT user_name AS userName, COUNT(*) AS totalCount FROM user;

-- GaussDB
SELECT user_name AS "userName", COUNT(*) AS "totalCount" FROM user;
```

> 表别名（如 `FROM user u`）不需要加引号

### 1.3 字符串值

**规则：双引号改为单引号（WHERE、LIKE、IN 等所有位置）**

```sql
-- MySQL
WHERE status = "active" AND name LIKE "%张%" AND code IN ("A", "B")

-- GaussDB
WHERE status = 'active' AND name LIKE '%张%' AND code IN ('A', 'B')
```

---

## 二、函数转换

### 2.1 通用函数

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `IFNULL(a, b)` | `COALESCE(a, b)` | 空值处理 |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` | 条件判断 |
| `JSON_OBJECT(k, v)` | `json_build_object(k, v)` | 构建 JSON |
| `JSON_CONTAINS(col, val)` | `col::jsonb @> val::jsonb` | JSON 包含 |
| `ANY_VALUE(col)` | `MAX(col)` | 任意值 |
| `SELECT EXISTS(...)` | `SELECT (EXISTS(...))::int` | EXISTS 返回 0/1 |

### 2.2 日期时间函数

#### 获取当前时间
| MySQL | GaussDB |
|-------|---------|
| `NOW()` | `NOW()`（兼容） |
| `CURDATE()` | `CURRENT_DATE` |
| `CURTIME()` | `CURRENT_TIME` |
| `SYSDATE()` | `NOW()` |

#### 时间戳转换
| MySQL | GaussDB |
|-------|---------|
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` |
| `UNIX_TIMESTAMP(date)` | `EXTRACT(EPOCH FROM date)::INTEGER` |
| `FROM_UNIXTIME(ts)` | `TO_TIMESTAMP(ts)` |

#### 日期格式化
| MySQL | GaussDB |
|-------|---------|
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:%s')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:SS')` |
| `DATE_FORMAT(d, '%Y-%m-%d %H:00:00')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:00:00')` |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:00')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:00')` |
| `STR_TO_DATE(str, '%Y-%m-%d')` | `TO_DATE(str, 'YYYY-MM-DD')` |

**格式符对照：** `%Y`→`YYYY`, `%m`→`MM`, `%d`→`DD`, `%H`→`HH24`, `%i`→`MI`, `%s`→`SS`

#### 日期加减
| MySQL | GaussDB |
|-------|---------|
| `DATE_ADD(d, INTERVAL n DAY)` | `d + INTERVAL 'n DAY'` |
| `DATE_SUB(d, INTERVAL n DAY)` | `d - INTERVAL 'n DAY'` |
| `DATEDIFF(d1, d2)` | `(d1::date - d2::date)` |
| `TIMESTAMPDIFF(DAY, start, end)` | `EXTRACT(DAY FROM (end - start))` |

#### 提取日期部分
| MySQL | GaussDB |
|-------|---------|
| `DATE(datetime)` | `TO_CHAR(datetime, 'YYYY-MM-DD')` |
| `TIME(datetime)` | `datetime::time` |
| `YEAR/MONTH/DAY(d)` | `EXTRACT(YEAR/MONTH/DAY FROM d)` |

---

## 三、特殊场景处理

### 3.1 LAST_INSERT_ID（需扫描 DDL）

**步骤：**
1. 扫描 DDL 获取真实序列名：`grep -rn "nextval" --include="*.sql"`
2. 使用真实序列名替换

```sql
-- MySQL
SELECT LAST_INSERT_ID();

-- GaussDB（使用扫描到的真实序列名）
SELECT currval('user_id_seq');
```

### 3.2 ON DUPLICATE KEY UPDATE（需扫描 DDL）

**规则：UPDATE 子句中不能包含唯一索引字段**

**步骤：**
1. 扫描唯一索引：`grep -rn "UNIQUE\|PRIMARY KEY" --include="*.sql"`
2. 从 UPDATE 子句中移除唯一索引字段

```sql
-- MySQL（username 是唯一索引）
ON DUPLICATE KEY UPDATE username = VALUES(username), status = VALUES(status)

-- GaussDB（移除唯一索引字段）
ON DUPLICATE KEY UPDATE status = VALUES(status)
```

### 3.3 INSERT/UPDATE 类型转换（需扫描 Java 类 + DDL）

**规则：只有 Java String 传给 DB INT/JSON 字段时才需要转换**

**步骤：**
1. 扫描 DDL 获取 INT/JSON 字段：`grep -rnE "\b(INT|JSON)\b" --include="*.sql"`
2. 扫描 Java 类获取参数类型：`grep -rn "private String" --include="*.java"`
3. 对照后添加类型转换

```java
// Java 类
private Long userId;      // Long → DB INT，不需要转换
private String score;     // String → DB INT，需要 ::int
private String settings;  // String → DB JSON，需要 ::json
```

```sql
-- GaussDB（只转换 String→INT/JSON）
INSERT INTO config(user_id, score, settings)
VALUES(#{userId}, #{score}::int, #{settings}::json)
--    Long不转换   String需转换    String需转换

-- WHERE 子句不需要类型转换
WHERE user_id = #{userId}
```

### 3.4 GROUP BY 非聚合列

```sql
-- MySQL
SELECT user_id, username, COUNT(*) FROM orders GROUP BY user_id;

-- GaussDB（非聚合列需用聚合函数包装）
SELECT user_id, MAX(username) AS username, COUNT(*) FROM orders GROUP BY user_id;
```

### 3.5 ORDER BY NULL 值

```sql
-- MySQL
ORDER BY age ASC, create_time DESC

-- GaussDB（需指定 NULL 排序位置）
ORDER BY age ASC NULLS FIRST, create_time DESC NULLS LAST
```

---

## 四、兼容语法（无需转换）

- `LIMIT offset, count` / `LIMIT count`
- `NOW()`, `COALESCE()`, `CONCAT()`
- `GROUP_CONCAT(col)`, `GROUP_CONCAT(col SEPARATOR ';')`
- `TRIM()`, `UPPER()`, `LOWER()`
- `ABS()`, `ROUND()`, `CEIL()`, `FLOOR()`

---

## 五、注意事项

**XML 转义字符保留：** `&lt;` `&gt;` `&amp;` 不需要修改

---

## 六、校验清单

### 必须转换（期望：grep 无匹配）

```bash
# 标识符
grep -rn "\`" --include="*.xml" --include="*.java"

# 双引号字符串
grep -rnE '=\s*"[^"]+"' --include="*.xml" --include="*.java"

# 函数
grep -rn "IFNULL\|JSON_OBJECT\|ANY_VALUE" --include="*.xml"
grep -rnE "\bIF\s*\(" --include="*.xml"

# 日期函数
grep -rn "DATE_FORMAT\|STR_TO_DATE\|UNIX_TIMESTAMP\|FROM_UNIXTIME" --include="*.xml"
grep -rn "CURDATE\|CURTIME\|SYSDATE\|DATEDIFF\|TIMESTAMPDIFF" --include="*.xml"
grep -rnE "\b(DATE|TIME|YEAR|MONTH|DAY|HOUR|MINUTE|SECOND)\s*\(" --include="*.xml"
```

### 需人工检查

| 检查项 | 命令 | 要点 |
|--------|------|------|
| 别名 | `grep -rnE "\bAS\s+\w+" --include="*.xml"` | 确认加了双引号 |
| LAST_INSERT_ID | `grep -rn "LAST_INSERT_ID"` | 需扫描 DDL 获取序列名 |
| ON DUPLICATE KEY | `grep -rn "ON DUPLICATE KEY"` | 需扫描 DDL 确认唯一索引 |
| ORDER BY | `grep -rnE "ORDER\s+BY"` | 确认加了 NULLS FIRST/LAST |
| GROUP BY | `grep -rnE "GROUP\s+BY"` | 确认非聚合列已处理 |
| INSERT/UPDATE | `grep -rnE "INSERT\|UPDATE"` | 需对照 Java 类确认类型转换 |

### 转换报告

转换完成后，输出以下统计信息：

```
=== 转换报告 ===

扫描文件数：X 个
扫描文件列表：
  - src/main/resources/mapper/UserMapper.xml
  - src/main/resources/mapper/OrderMapper.xml
  - ...

修改文件数：Y 个
修改文件列表：
  - src/main/resources/mapper/UserMapper.xml（5处修改）
  - src/main/resources/mapper/OrderMapper.xml（3处修改）
  - ...
```

---

## 七、完整示例

```xml
<!-- ========== MySQL ========== -->
<select id="getUserStats" resultType="map">
    SELECT `user_id`, `username`,
           DATE_FORMAT(`create_time`, '%Y-%m-%d') AS createDate,
           IFNULL(`nickname`, `username`) AS displayName,
           IF(`status` = 1, '启用', '禁用') AS statusText
    FROM `user` u
    WHERE `status` = "active"
    ORDER BY `create_time` DESC
</select>

<!-- ========== GaussDB ========== -->
<select id="getUserStats" resultType="map">
    SELECT user_id, username,
           TO_CHAR(create_time, 'YYYY-MM-DD') AS "createDate",
           COALESCE(nickname, username) AS "displayName",
           CASE WHEN status = 1 THEN '启用' ELSE '禁用' END AS "statusText"
    FROM user u
    WHERE status = 'active'
    ORDER BY create_time DESC NULLS LAST
</select>
```

```xml
<!-- INSERT/UPDATE 类型转换示例 -->
<!-- Java: Long userId, Integer age, String score, String settings -->

<!-- MySQL -->
<insert id="insert">
    INSERT INTO config(user_id, age, score, settings)
    VALUES(#{userId}, #{age}, #{score}, #{settings})
</insert>

<!-- GaussDB（只转换 String→INT/JSON） -->
<insert id="insert">
    INSERT INTO config(user_id, age, score, settings)
    VALUES(#{userId}, #{age}, #{score}::int, #{settings}::json)
</insert>
```
