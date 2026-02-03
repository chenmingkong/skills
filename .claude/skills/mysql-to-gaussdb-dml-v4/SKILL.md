---
name: mysql-to-gaussdb-dml-v4
description: 将 MySQL DML 语句转换为 GaussDB 兼容语法（模块化版本 v4）
argument-hint: "[Mapper文件或目录]"
---

# MySQL to GaussDB DML 转换（v4 - 模块化版本）

MySQL DML 转 GaussDB 兼容语法。**版本 505.2.1**

本版本采用模块化 reference 结构，规则更清晰、更易维护。

---

## 转换流程

### 步骤 1：扫描文件

扫描以下文件范围：

```
**/*Mapper.xml                     # MyBatis 映射文件
**/*Repository.java, **/*Dao.java  # @Query 原生 SQL
**/*.java                          # 其他 Java 文件中的 SQL
**/*.sql                           # DDL 文件（用于类型映射）
```

**重要：必须完整扫描所有匹配文件，逐行检查并转换。**

### 步骤 2：依次应用转换规则

按以下顺序检查每个 SQL 语句：

1. **标识符和字符串转换** → 参考 `references/01-identifiers.md`
   - 反引号处理（普通字段去掉，关键字用双引号）
   - 字段别名（Map 类型加双引号）
   - 字符串值用单引号

2. **通用函数转换** → 参考 `references/02-common-functions.md`
   - IFNULL → COALESCE
   - IF → CASE WHEN
   - GROUP_CONCAT → STRING_AGG
   - JSON_OBJECT → json_build_object
   - 等等...

3. **日期时间函数转换** → 参考 `references/03-datetime-functions.md`
   - DATE_FORMAT → TO_CHAR
   - DATE_ADD/DATE_SUB → INTERVAL
   - UNIX_TIMESTAMP → EXTRACT
   - 等等...

4. **特殊场景处理** → 参考 `references/04-special-cases.md`
   - LAST_INSERT_ID（需扫描 DDL）
   - ON DUPLICATE KEY UPDATE（需扫描唯一索引）
   - INSERT/UPDATE 类型转换（需扫描 Java 类）
   - GROUP BY 非聚合列
   - ORDER BY NULL 处理

### 步骤 3：执行校验

完成转换后，执行完整性校验 → 参考 `references/05-validation.md`

- 运行 grep 命令检查遗漏项
- 生成校验报告

---

## 核心原则

### 1. 匹配即转换，无匹配则通过

- 依次检查每个转换规则的检测模式
- 如果匹配到，应用对应的转换规则
- 如果没有匹配项，保持原样，不做修改

### 2. 最小化修改

- 只修改需要转换的语法问题
- 保持文件其他内容不变
- 不做多余的格式化或优化

### 3. 基于用例验证

- 每个转换规则都有明确的示例
- 转换前后对比清晰
- 可快速验证转换正确性

---

## 转换规则参考

### 📋 01 - 标识符与字符串转换

详见：`references/01-identifiers.md`

**关键规则：**
- 普通标识符：去掉反引号
- SQL 关键字：用双引号（order, desc, group, key, value 等）
- 字段别名：只有 resultType="map/HashMap" 时加双引号
- 字符串值：必须用单引号

**快速参考：**
| 场景 | MySQL | GaussDB |
|------|-------|---------|
| 普通字段 | `` `user` `` | `user` |
| 关键字字段 | `` `order` `` | `"order"` |
| Map 别名 | `AS userName` | `AS "userName"` |
| 实体类别名 | `AS userName` | `AS userName` |
| 字符串值 | `"张三"` | `'张三'` |

---

### ⚙️ 02 - 通用函数转换

详见：`references/02-common-functions.md`

**快速参考：**
| MySQL | GaussDB | 检测模式 |
|-------|---------|---------|
| `IFNULL(a, b)` | `COALESCE(a, b)` | `IFNULL\s*\(` |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` | `\bIF\s*\(` |
| `GROUP_CONCAT(col)` | `STRING_AGG(col::text, ',')` | `GROUP_CONCAT\s*\(` |
| `JSON_OBJECT(k, v)` | `json_build_object(k, v)` | `JSON_OBJECT\s*\(` |
| `JSON_CONTAINS(col, val)` | `col::jsonb @> val::jsonb` | `JSON_CONTAINS\s*\(` |
| `ANY_VALUE(col)` | `MAX(col)` | `ANY_VALUE\s*\(` |

---

### 📅 03 - 日期时间函数转换

详见：`references/03-datetime-functions.md`

**格式符映射：**
- `%Y` → `YYYY`（4位年份）
- `%m` → `MM`（月份）
- `%d` → `DD`（日期）
- `%H` → `HH24`（24小时制）
- `%i` → `MI`（分钟）
- `%s` → `SS`（秒）

**快速参考：**
| MySQL | GaussDB | 检测模式 |
|-------|---------|---------|
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` | `DATE_FORMAT\s*\(` |
| `DATE_ADD(d, INTERVAL n DAY)` | `TO_CHAR(CAST(d AS TIMESTAMP) + INTERVAL 'n DAY', 'YYYY-MM-DD HH24:MI:SS')` | `DATE_ADD\s*\(` |
| `DATEDIFF(d1, d2)` | `(d1::date - d2::date)` | `DATEDIFF\s*\(` |
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` | `UNIX_TIMESTAMP\s*\(` |
| `CURDATE()` | `CURRENT_DATE` | `CURDATE\s*\(` |
| `YEAR(date)` | `EXTRACT(YEAR FROM date)` | `\bYEAR\s*\(` |

---

### 🔧 04 - 特殊场景处理

详见：`references/04-special-cases.md`

**需要扫描 DDL 的场景：**

1. **LAST_INSERT_ID** - 需扫描序列定义
   ```bash
   grep -rn "CREATE SEQUENCE" --include="*.sql"
   ```

2. **ON DUPLICATE KEY UPDATE** - 需扫描唯一索引
   ```bash
   grep -rn "UNIQUE" --include="*.sql"
   ```

3. **INSERT/UPDATE 类型转换** - 需扫描 Java 类和 DDL
   ```bash
   grep -rn "private String" --include="*.java"
   grep -rnE "\b(INT|INTEGER|JSON)\b" --include="*.sql"
   ```

**其他特殊场景：**

- **GROUP BY 非聚合列** - SELECT 非聚合列需用 MAX/MIN 包装
- **ORDER BY NULL 处理** - ASC 添加 NULLS FIRST，DESC 添加 NULLS LAST

---

### ✅ 05 - 完整性校验

详见：`references/05-validation.md`

**必须转换项检查（期望：无匹配）：**

```bash
# 标识符
grep -rn "\`" --include="*.xml" --include="*.java"

# 通用函数
grep -rn "IFNULL\|JSON_OBJECT\|GROUP_CONCAT" --include="*.xml"

# 日期时间函数
grep -rn "DATE_FORMAT\|UNIX_TIMESTAMP\|CURDATE" --include="*.xml"
```

**人工检查项：**

```bash
# Map 类型别名
grep -rnE "\bAS\s+[a-zA-Z]" --include="*.xml"
grep -rn 'resultType="map"' --include="*.xml"

# 特殊场景
grep -rn "LAST_INSERT_ID\|ON DUPLICATE KEY" --include="*.xml"
grep -rnE "ORDER\s+BY\|GROUP\s+BY" --include="*.xml"
```

**生成校验报告：**

输出到 `dml_完整性校验报告.txt`，包含：
- 已转换项统计
- 待人工检查项
- 扫描文件数和修改文件数

---

## 完整转换示例

### 示例 1：基础查询转换（包含多种转换）

```xml
<!-- ========== MySQL ========== -->
<select id="getUserStats" resultType="map">
    SELECT
        `user_id`,
        `username`,
        DATE_FORMAT(`create_time`, '%Y-%m-%d') AS createDate,
        IFNULL(`nickname`, `username`) AS displayName,
        IF(`status` = 1, '启用', '禁用') AS statusText,
        TIMESTAMPDIFF(DAY, `create_time`, NOW()) AS daysSinceCreated
    FROM `user` u
    WHERE `status` = "active"
      AND `create_time` &gt;= DATE_SUB(NOW(), INTERVAL 30 DAY)
    ORDER BY `create_time` DESC
</select>

<!-- ========== GaussDB ========== -->
<select id="getUserStats" resultType="map">
    SELECT
        user_id AS "userId",
        username AS "username",
        TO_CHAR(create_time, 'YYYY-MM-DD') AS "createDate",
        COALESCE(nickname, username) AS "displayName",
        CASE WHEN status = 1 THEN '启用' ELSE '禁用' END AS "statusText",
        EXTRACT(DAY FROM (NOW() - create_time)) AS "daysSinceCreated"
    FROM user u
    WHERE status = 'active'
      AND create_time &gt;= TO_CHAR(CAST(NOW() AS TIMESTAMP) - INTERVAL '30 DAY', 'YYYY-MM-DD HH24:MI:SS')
    ORDER BY create_time DESC NULLS LAST
</select>
```

**转换说明：**
1. ✅ 去掉反引号（普通字段）
2. ✅ resultType="map"，所有别名加双引号
3. ✅ DATE_FORMAT → TO_CHAR + 格式符转换
4. ✅ IFNULL → COALESCE
5. ✅ IF → CASE WHEN
6. ✅ TIMESTAMPDIFF → EXTRACT
7. ✅ DATE_SUB → INTERVAL
8. ✅ 字符串值双引号 → 单引号
9. ✅ ORDER BY 添加 NULLS LAST

---

### 示例 2：SQL 关键字字段转换

```xml
<!-- ========== MySQL ========== -->
<select id="getOrderList" resultType="map">
    SELECT
        `id`,
        `order`,           -- order 是 SQL 关键字
        `desc`,            -- desc 是 SQL 关键字
        `key`,             -- key 是 SQL 关键字
        `value`,           -- value 是 SQL 关键字
        `type`             -- type 是 SQL 关键字
    FROM `order`
    WHERE `type` = "normal"
    ORDER BY `order` ASC
</select>

<!-- ========== GaussDB ========== -->
<select id="getOrderList" resultType="map">
    SELECT
        id AS "id",
        "order" AS "order",        -- 关键字用双引号
        "desc" AS "desc",          -- 关键字用双引号
        "key" AS "key",            -- 关键字用双引号
        "value" AS "value",        -- 关键字用双引号
        "type" AS "type"           -- 关键字用双引号
    FROM "order"                   -- 表名也是关键字
    WHERE "type" = 'normal'        -- 关键字字段 + 字符串值单引号
    ORDER BY "order" ASC NULLS FIRST
</select>
```

**转换说明：**
1. ✅ SQL 关键字字段用双引号包裹
2. ✅ resultType="map"，别名加双引号
3. ✅ 字符串值用单引号
4. ✅ ORDER BY 添加 NULLS FIRST

---

### 示例 3：INSERT/UPDATE 类型转换

```java
// Java 实体类
public class ConfigDTO {
    private Long userId;         // Long → DB BIGINT，不需要转换
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

**转换说明：**
1. ✅ 扫描 Java 类获取字段类型
2. ✅ 扫描 DDL 获取数据库字段类型
3. ✅ 只对 Java String → DB INT/JSON 添加类型转换
4. ✅ Java Integer/Long 不需要转换

---

## 注意事项

### XML 转义字符保留

- `&lt;` 不需要改成 `<`
- `&gt;` 不需要改成 `>`
- `&amp;` 不需要改成 `&`

这些是 XML 的标准转义，在 MyBatis Mapper XML 中用于比较运算符，必须保留。

### 兼容的 MySQL 语法（无需转换）

以下语法 GaussDB 直接兼容：

- `LIMIT offset, count` / `LIMIT count` - 分页语法
- `NOW()` - 当前时间
- `COALESCE()` - 空值处理
- `CONCAT()` - 字符串拼接
- `TRIM()`, `UPPER()`, `LOWER()` - 字符串函数
- `ABS()`, `ROUND()`, `CEIL()`, `FLOOR()` - 数学函数

---

## 使用说明

1. **启动转换**：`/mysql-to-gaussdb-dml-v4 [目录或文件]`
2. **查看规则**：需要详细规则时，查看对应的 reference 文件
3. **执行校验**：转换完成后，运行 `references/05-validation.md` 中的检查命令
4. **生成报告**：输出完整性校验报告到 `dml_完整性校验报告.txt`

---

## 版本说明

**v4 改进：**
- ✅ 模块化 reference 结构，规则更清晰
- ✅ 每个 reference 专注单一职责
- ✅ 检测模式明确，易于匹配
- ✅ 匹配即转换，无匹配则通过
- ✅ 完整的示例和验证方法

**与 v3 的区别：**
- v3：所有规则在一个文件中，难以维护
- v4：规则拆分成独立 reference，易于查找和更新
