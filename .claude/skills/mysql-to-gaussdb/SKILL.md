---
name: mysql-to-gaussdb
description: 将 Spring Boot 项目从 MySQL 迁移到 GaussDB，自动转换依赖、配置和 SQL 语法
argument-hint: "[项目目录]"
---

# MySQL to GaussDB 迁移

自动扫描并转换 Spring Boot 项目中的 MySQL 代码为 GaussDB 兼容代码。

## 执行步骤

### 1. 扫描文件

```
**/pom.xml, **/build.gradle          # 依赖配置
**/application*.yml, **/application*.properties  # 数据库配置
**/*Mapper.xml                        # MyBatis 映射
**/*.sql                              # SQL 脚本
**/*Repository.java, **/*Dao.java    # @Query 原生 SQL
```

### 2. 修改依赖

**pom.xml:**
```xml
<!-- 删除 -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>

<!-- 添加 -->
<dependency>
    <groupId>org.opengauss</groupId>
    <artifactId>opengauss-jdbc</artifactId>
    <version>6.0.0-htrunk3.csi.gaussdb_kernel.opengaussjdbc.r1</version>
</dependency>
```

**build.gradle:**
```groovy
// 删除: implementation 'mysql:mysql-connector-java'
// 添加:
implementation 'org.opengauss:opengauss-jdbc:6.0.0-htrunk3.csi.gaussdb_kernel.opengaussjdbc.r1'
```

### 3. 修改配置

| 配置项 | MySQL | GaussDB |
|--------|-------|---------|
| driver-class-name | `com.mysql.cj.jdbc.Driver` | `com.huawei.opengauss.jdbc.Driver` |
| url | `jdbc:mysql://host:3306/db` | `jdbc:opengauss://host:5432/db?currentSchema=public` |
| database-platform | `MySQL8Dialect` | `org.hibernate.dialect.PostgreSQLDialect` |

**PageHelper 分页配置：**

application.yml 添加：
```yaml
pagehelper:
  helper-dialect: postgresql
  reasonable: true
  support-methods-argument: true
```

application.properties 添加：
```properties
pagehelper.helper-dialect=postgresql
pagehelper.reasonable=true
pagehelper.support-methods-argument=true
```

mybatis-config.xml（如配置了 plugins）添加：
```xml
<plugins>
    <plugin interceptor="com.github.pagehelper.PageInterceptor">
        <property name="helperDialect" value="postgresql"/>
    </plugin>
</plugins>
```

### 4. SQL 语法转换

**标识符引号转换（表名、字段名）：**

MySQL 使用反引号 `` ` `` 包裹表名和字段名，GaussDB 使用双引号 `"` 包裹：

```sql
-- MySQL 写法
SELECT `id`, `user_name` FROM `user` WHERE `status` = 1;
SELECT `User`.`Name` FROM `User`;

-- GaussDB 写法（反引号全部替换为双引号，标识符转小写）
SELECT "id", "user_name" FROM "user" WHERE "status" = 1;
SELECT "user"."name" FROM "user";
```

**转换规则：**
- `` `identifier` `` → `"identifier"` （反引号替换为双引号）
- `` `TableName` `` → `"tablename"` （大写必须转为小写）
- `` `Column_Name` `` → `"column_name"` （大写必须转为小写）

**通用函数转换：**

| MySQL | GaussDB |
|-------|---------|
| 字符串值 `"text"` | 字符串值 `'text'` |
| `IFNULL(a, b)` | `COALESCE(a, b)` |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` |
| `ON DUPLICATE KEY UPDATE` | `ON CONFLICT (key) DO UPDATE SET` |
| `GROUP_CONCAT(col)` | `STRING_AGG(col::text, ',')` |
| SELECT 非聚合列不在 GROUP BY 中 | 对非聚合列使用 `MAX(col)` 包装 |
| `SELECT EXISTS(...)` 返回 0/1 | `SELECT (EXISTS(...))::int` |
| `JSON_OBJECT(key, value, ...)` | `json_build_object(key, value, ...)` |
| `ANY_VALUE(col)` | `MAX(col)` |

**日期时间函数转换（重要）：**

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `NOW()` | `NOW()` | 兼容，无需修改 |
| `CURDATE()` | `CURRENT_DATE` | 获取当前日期 |
| `CURTIME()` | `CURRENT_TIME` | 获取当前时间 |
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` | 获取当前时间戳 |
| `UNIX_TIMESTAMP(date)` | `EXTRACT(EPOCH FROM date)::INTEGER` | 日期转时间戳 |
| `FROM_UNIXTIME(ts)` | `TO_TIMESTAMP(ts)` | 时间戳转日期 |
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` | 日期格式化 |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:%s')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:SS')` | 日期时间格式化 |
| `DATE_FORMAT(d, '%Y%m%d')` | `TO_CHAR(d, 'YYYYMMDD')` | 日期格式化（无分隔符） |
| `DATE_FORMAT(d, '%H:%i:%s')` | `TO_CHAR(d, 'HH24:MI:SS')` | 时间格式化 |
| `DATE_ADD(date, INTERVAL n DAY)` | `TO_CHAR(CAST(date AS TIMESTAMP) + INTERVAL 'n DAY', 'YYYY-MM-DD')` | 日期加天数 |
| `DATE_ADD(date, INTERVAL n MONTH)` | `TO_CHAR(CAST(date AS TIMESTAMP) + INTERVAL 'n MONTH', 'YYYY-MM-DD')` | 日期加月数 |
| `DATE_ADD(date, INTERVAL n YEAR)` | `TO_CHAR(CAST(date AS TIMESTAMP) + INTERVAL 'n YEAR', 'YYYY-MM-DD')` | 日期加年数 |
| `DATE_SUB(date, INTERVAL n DAY)` | `TO_CHAR(CAST(date AS TIMESTAMP) - INTERVAL 'n DAY', 'YYYY-MM-DD')` | 日期减天数 |
| `DATE_SUB(date, INTERVAL n MONTH)` | `TO_CHAR(CAST(date AS TIMESTAMP) - INTERVAL 'n MONTH', 'YYYY-MM-DD')` | 日期减月数 |
| `DATE_SUB(date, INTERVAL n YEAR)` | `TO_CHAR(CAST(date AS TIMESTAMP) - INTERVAL 'n YEAR', 'YYYY-MM-DD')` | 日期减年数 |
| `DATEDIFF(date1, date2)` | `(date1::date - date2::date)` | 两日期相差天数 |
| `TIMESTAMPDIFF(DAY, start, end)` | `EXTRACT(DAY FROM (end - start))` | 时间差（天） |
| `TIMESTAMPDIFF(HOUR, start, end)` | `EXTRACT(EPOCH FROM (end - start))::INTEGER / 3600` | 时间差（小时） |
| `TIMESTAMPDIFF(MINUTE, start, end)` | `EXTRACT(EPOCH FROM (end - start))::INTEGER / 60` | 时间差（分钟） |
| `TIMESTAMPDIFF(SECOND, start, end)` | `EXTRACT(EPOCH FROM (end - start))::INTEGER` | 时间差（秒） |
| `DATE(datetime)` | `datetime::date` 或 `CAST(datetime AS DATE)` | 提取日期部分 |
| `TIME(datetime)` | `datetime::time` 或 `CAST(datetime AS TIME)` | 提取时间部分 |
| `YEAR(date)` | `EXTRACT(YEAR FROM date)` | 提取年份 |
| `MONTH(date)` | `EXTRACT(MONTH FROM date)` | 提取月份 |
| `DAY(date)` | `EXTRACT(DAY FROM date)` | 提取日 |
| `HOUR(time)` | `EXTRACT(HOUR FROM time)` | 提取小时 |
| `MINUTE(time)` | `EXTRACT(MINUTE FROM time)` | 提取分钟 |
| `SECOND(time)` | `EXTRACT(SECOND FROM time)` | 提取秒 |
| `STR_TO_DATE(str, '%Y-%m-%d')` | `TO_DATE(str, 'YYYY-MM-DD')` | 字符串转日期 |
| `STR_TO_DATE(str, '%Y-%m-%d %H:%i:%s')` | `TO_TIMESTAMP(str, 'YYYY-MM-DD HH24:MI:SS')` | 字符串转时间戳 |

**DATE_FORMAT 格式符对照：**

| MySQL 格式符 | GaussDB 格式符 | 含义 |
|-------------|---------------|------|
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

**GROUP BY 非聚合列处理说明：**
MySQL 默认允许 SELECT 中包含不在 GROUP BY 子句中的非聚合列（`ONLY_FULL_GROUP_BY` 关闭时），GaussDB 严格遵循 SQL 标准，不允许此行为。

```sql
-- MySQL (允许)
SELECT user_id, username, COUNT(*) FROM orders GROUP BY user_id;

-- GaussDB (需要修改)
SELECT user_id, MAX(username) AS username, COUNT(*) FROM orders GROUP BY user_id;
```

**GaussDB 兼容的 MySQL 语法（无需转换）：**
- `LIMIT offset, count` - 分页语法直接兼容
- `LIMIT count` - 直接兼容

### 5. DDL 类型转换

| MySQL | GaussDB |
|-------|---------|
| `BIGINT AUTO_INCREMENT` | `BIGINT` + 序列 |
| `INT AUTO_INCREMENT` | `INTEGER` + 序列 |
| `TINYINT(1)` | `BOOLEAN` |
| `TINYINT` | `int2` |
| `DATETIME` / `TIMESTAMP` | `timestamp(0)` |
| `DOUBLE(m,n)` | `numeric(m,n)` |
| `DOUBLE` / `FLOAT` | `float8` |
| `BLOB` / `LONGBLOB` | `BYTEA` |
| `LONGTEXT` / `MEDIUMTEXT` | `TEXT` |
| `VARCHAR(n)` | `NVARCHAR2(n)` |

**AUTO_INCREMENT 转换为序列：**

GaussDB 需要创建序列实现自增，并设置起始值为当前最大值+1：

```sql
-- 1. 创建序列
CREATE SEQUENCE table_name_id_seq;

-- 2. 设置序列起始值为当前最大值+1
SELECT setval('table_name_id_seq', COALESCE((SELECT MAX(id) FROM table_name), 0) + 1, false);

-- 3. 设置列默认值为序列
ALTER TABLE table_name ALTER COLUMN id SET DEFAULT nextval('table_name_id_seq');
```

**COMMENT 注释转换：**

GaussDB 不支持在字段定义中声明 COMMENT，需要单独使用 `COMMENT ON COLUMN`：

```sql
-- MySQL
CREATE TABLE user (
    id BIGINT COMMENT '用户ID',
    name VARCHAR(50) COMMENT '用户名'
);

-- GaussDB
CREATE TABLE user (
    id BIGINT,
    name VARCHAR(50)
);
COMMENT ON COLUMN user.id IS '用户ID';
COMMENT ON COLUMN user.name IS '用户名';
```

**ON UPDATE CURRENT_TIMESTAMP 转换：**

GaussDB 不支持 `ON UPDATE CURRENT_TIMESTAMP`，需使用触发器实现：

```sql
-- MySQL
CREATE TABLE user (
    id BIGINT,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- GaussDB: 创建触发器函数
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.update_time = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 创建触发器
CREATE TRIGGER trg_user_update_time
    BEFORE UPDATE ON user
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();
```

**索引名称转换：**

GaussDB 同一 schema 中不允许索引名重复，需将索引名改为"表名_索引名"格式：

```sql
-- MySQL
CREATE INDEX idx_name ON user(name);
CREATE INDEX idx_name ON order(name);

-- GaussDB
CREATE INDEX user_idx_name ON user(name);
CREATE INDEX order_idx_name ON order(name);
```

**时间字段默认值转换：**

GaussDB 不支持零值时间作为默认值：

| MySQL | GaussDB |
|-------|---------|
| `DEFAULT '0000-00-00 00:00:00'` | `DEFAULT CURRENT_TIMESTAMP` |
| `DEFAULT '00-00-00 00:00:00'` | `DEFAULT CURRENT_TIMESTAMP` |

删除 MySQL 特有语法: `ENGINE=InnoDB`, `DEFAULT CHARSET=utf8mb4`, `UNSIGNED`, `AUTO_INCREMENT=N`, `ON UPDATE CURRENT_TIMESTAMP`

### 6. 迁移完整性校验

转换完成后，执行以下检查确保没有遗漏：

#### 6.1 扫描残留 MySQL 语法

```bash
# ========== 依赖和配置检查 ==========
# 检查是否还有 MySQL 驱动引用
grep -r "mysql-connector-java" --include="pom.xml" --include="build.gradle"
grep -r "com.mysql" --include="*.java" --include="*.xml" --include="*.yml" --include="*.properties"

# 检查是否还有 MySQL JDBC URL
grep -r "jdbc:mysql" --include="*.yml" --include="*.properties" --include="*.xml"

# 检查是否还有 MySQL 方言
grep -r "MySQLDialect\|MySQL5Dialect\|MySQL8Dialect" --include="*.yml" --include="*.properties" --include="*.java"

# 检查 PageHelper 方言配置（应为 postgresql）
grep -r "helper-dialect" --include="*.yml" --include="*.properties" --include="*.xml"

# ========== DML 语法检查（查询、函数） ==========
# 检查是否还有反引号（MySQL 特有）
grep -r "\`" --include="*.xml" --include="*.sql" --include="*.java"

# 检查双引号包围字符串值（GaussDB 双引号仅用于标识符）
grep -rE "=\s*\"[^\"]+\"" --include="*.xml" --include="*.sql" --include="*.java"

# 检查未转换的通用函数
grep -rE "IFNULL|GROUP_CONCAT|JSON_OBJECT|JSON_CONTAINS|ANY_VALUE" --include="*.xml" --include="*.sql" --include="*.java"
grep -rE "\bIF\s*\(" --include="*.xml" --include="*.sql" --include="*.java"

# 检查未转换的 MySQL 日期函数（重点检查）
grep -rE "DATE_FORMAT|STR_TO_DATE" --include="*.xml" --include="*.sql" --include="*.java"
grep -rE "UNIX_TIMESTAMP|FROM_UNIXTIME" --include="*.xml" --include="*.sql" --include="*.java"
grep -rE "CURDATE|CURTIME|SYSDATE" --include="*.xml" --include="*.sql" --include="*.java"
grep -rE "DATE_ADD|DATE_SUB" --include="*.xml" --include="*.sql" --include="*.java"
grep -rE "DATEDIFF|TIMESTAMPDIFF" --include="*.xml" --include="*.sql" --include="*.java"
grep -rE "\b(YEAR|MONTH|DAY|HOUR|MINUTE|SECOND)\s*\(" --include="*.xml" --include="*.sql" --include="*.java"

# 检查 ON DUPLICATE KEY
grep -r "ON DUPLICATE KEY" --include="*.xml" --include="*.sql" --include="*.java"

# 检查 ORDER BY 语句（需人工确认 NULLS FIRST/LAST）
grep -rE "ORDER\s+BY" --include="*.xml" --include="*.sql" --include="*.java"

# 检查 GROUP BY 语句（需人工确认非聚合列是否都已处理）
grep -rE "GROUP\s+BY" --include="*.xml" --include="*.sql" --include="*.java"

# 检查 EXISTS 返回值（如需返回 0/1 需转换为 (EXISTS(...))::int）
grep -rE "SELECT\s+EXISTS" --include="*.xml" --include="*.sql" --include="*.java"

# ========== DDL 语法检查（建表、索引） ==========
# 检查 MySQL 特有的 DDL 语法
grep -rE "AUTO_INCREMENT" --include="*.sql"
grep -rE "ENGINE\s*=" --include="*.sql"
grep -rE "\bUNSIGNED\b" --include="*.sql"
grep -rE "CHARSET\s*=|COLLATE\s*=" --include="*.sql"
grep -rE "ON\s+UPDATE\s+CURRENT_TIMESTAMP" --include="*.sql"
grep -rE "ROW_FORMAT\s*=" --include="*.sql"

# 检查零值时间默认值
grep -rE "DEFAULT\s*'0+(-0+){2}" --include="*.sql"

# 检查字段定义中的 COMMENT（需转为 COMMENT ON COLUMN）
grep -rE "COMMENT\s*'" --include="*.sql"

# 检查 VARCHAR（应转为 NVARCHAR2）
grep -rE "\bVARCHAR\s*\(" --include="*.sql"

# 检查索引名是否包含表名前缀（避免同 schema 索引名重复）
grep -rE "CREATE\s+(UNIQUE\s+)?INDEX\s+idx_" --include="*.sql"
```

#### 6.2 校验清单

**依赖和配置：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| MySQL 驱动 | `grep -r "mysql-connector-java"` | 无匹配 |
| MySQL 驱动类 | `grep -r "com.mysql"` | 无匹配 |
| JDBC URL | `grep -r "jdbc:mysql"` | 无匹配 |
| MySQL 方言 | `grep -r "MySQLDialect"` | 无匹配 |
| PageHelper 方言 | `grep -r "helper-dialect"` | postgresql |

**DML 语法（查询、函数）：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| 反引号 | `grep -r "\`"` | 无匹配 |
| 双引号字符串 | `grep -rE "=\"[^\"]+\""` | 无匹配 |
| IFNULL | `grep -r "IFNULL"` | 无匹配 |
| IF( | `grep -rE "\bIF\s*\("` | 无匹配 |
| GROUP_CONCAT | `grep -r "GROUP_CONCAT"` | 无匹配 |
| JSON_OBJECT | `grep -r "JSON_OBJECT"` | 无匹配 |
| JSON_CONTAINS | `grep -r "JSON_CONTAINS"` | 无匹配 |
| ANY_VALUE | `grep -r "ANY_VALUE"` | 无匹配 |
| ON DUPLICATE KEY | `grep -r "ON DUPLICATE KEY"` | 无匹配 |
| DATE_FORMAT | `grep -r "DATE_FORMAT"` | 无匹配 |
| STR_TO_DATE | `grep -r "STR_TO_DATE"` | 无匹配 |
| UNIX_TIMESTAMP | `grep -r "UNIX_TIMESTAMP"` | 无匹配 |
| FROM_UNIXTIME | `grep -r "FROM_UNIXTIME"` | 无匹配 |
| CURDATE | `grep -r "CURDATE"` | 无匹配 |
| CURTIME | `grep -r "CURTIME"` | 无匹配 |
| DATE_ADD/DATE_SUB | `grep -rE "DATE_ADD\|DATE_SUB"` | 无匹配 |
| DATEDIFF | `grep -r "DATEDIFF"` | 无匹配 |
| TIMESTAMPDIFF | `grep -r "TIMESTAMPDIFF"` | 无匹配 |
| YEAR/MONTH/DAY | `grep -rE "\b(YEAR\|MONTH\|DAY)\s*\("` | 无匹配 |
| HOUR/MINUTE/SECOND | `grep -rE "\b(HOUR\|MINUTE\|SECOND)\s*\("` | 无匹配 |
| ORDER BY | `grep -rE "ORDER\s+BY"` | 需人工检查 NULLS FIRST/LAST |
| GROUP BY | `grep -rE "GROUP\s+BY"` | 需人工检查非聚合列 |
| SELECT EXISTS | `grep -rE "SELECT\s+EXISTS"` | 需人工检查返回值 |

**DDL 语法（建表、索引）：**

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| AUTO_INCREMENT | `grep -r "AUTO_INCREMENT"` | 无匹配 |
| ENGINE= | `grep -rE "ENGINE\s*="` | 无匹配 |
| UNSIGNED | `grep -r "UNSIGNED"` | 无匹配 |
| CHARSET= | `grep -rE "CHARSET\s*="` | 无匹配 |
| COLLATE= | `grep -rE "COLLATE\s*="` | 无匹配 |
| ON UPDATE CURRENT_TIMESTAMP | `grep -r "ON UPDATE CURRENT_TIMESTAMP"` | 无匹配 |
| 零值时间默认值 | `grep -rE "DEFAULT.*0000-00-00"` | 无匹配 |
| 内联 COMMENT | `grep -rE "COMMENT\s*'"` | 无匹配 |
| VARCHAR | `grep -rE "\bVARCHAR\s*\("` | 无匹配（应为 NVARCHAR2） |
| 索引名前缀 | `grep -rE "CREATE\s+INDEX\s+idx_"` | 应含表名前缀 |

#### 6.3 生成校验报告

扫描完成后输出报告：

```
============ MySQL to GaussDB 迁移校验报告 ============

✅ 依赖配置
   - pom.xml: 已切换到 opengauss-jdbc 6.0.0
   - 无残留 MySQL 驱动引用

✅ 数据库配置
   - driver-class-name: com.huawei.opengauss.jdbc.Driver
   - url: jdbc:opengauss://...
   - dialect: PostgreSQLDialect
   - pagehelper.helper-dialect: postgresql

✅ SQL 语法
   - 无反引号
   - 无双引号字符串值
   - 无未转换的 MySQL 函数

✅ DDL 语法
   - 无 AUTO_INCREMENT（已转为序列）
   - 无内联 COMMENT（已转为 COMMENT ON COLUMN）
   - VARCHAR 已转为 NVARCHAR2
   - 索引名已添加表名前缀

⚠️ 待人工检查项（如有）
   - GROUP BY 语句：确认非聚合列已用 MAX() 包装
   - SELECT EXISTS：如需返回 0/1 需转为 (EXISTS(...))::int
   - src/main/resources/mapper/UserMapper.xml:45 - 发现反引号

❌ 未通过项（如有）
   - 仍存在 MySQL 驱动依赖
   - 仍使用 jdbc:mysql URL

============ 统计 ============
扫描文件: XX 个
已转换: XX 处
待处理: XX 处
```

### 7. 编译与测试

```bash
mvn clean compile  # 检查编译
mvn test           # 运行测试
```

### 8. 注意事项

**反引号必须替换为双引号：**
MySQL 使用反引号 `` ` `` 包裹标识符，GaussDB 使用双引号 `"`，转换规则：
- `` `user` `` → `"user"` （反引号改双引号）
- `` `order` `` → `"order"` （关键字必须用双引号）
- `` `UserName` `` → `"username"` （大写必须转小写）
- `` `table`.`column` `` → `"table"."column"` （表名.字段名同样处理）

**XML 转义字符保留：**
- `&lt;` 不需要改成 `<`
- `&gt;` 不需要改成 `>`
- `&amp;` 不需要改成 `&`

这些是 XML 的标准转义，在 MyBatis Mapper XML 中用于比较运算符，必须保留。

### 9. 常见遗漏项

| 遗漏类型 | 常见位置 | 解决方法 |
|----------|----------|----------|
| GROUP BY 非聚合列 | 所有包含 GROUP BY 的查询 | 用 `MAX(col)` 包装非聚合列 |
| 硬编码 SQL | `@Query` 注解 | 搜索 `nativeQuery = true` |
| 动态 SQL | MyBatis `<if>` 标签内 | 逐个检查 Mapper.xml |
| 测试代码 | `src/test/**` | 同步修改测试配置 |
| 多环境配置 | `application-dev.yml` 等 | 检查所有 profile 配置 |
| 数据库初始化 | `schema.sql`, `data.sql` | Flyway/Liquibase 脚本 |
| 存储过程 | `*.sql` | 需手动重写，语法差异大 |
