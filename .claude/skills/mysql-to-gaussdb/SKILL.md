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
    <version>5.0.0</version>
</dependency>
```

**build.gradle:**
```groovy
// 删除: implementation 'mysql:mysql-connector-java'
// 添加:
implementation 'org.opengauss:opengauss-jdbc:5.0.0'
```

### 3. 修改配置

| 配置项 | MySQL | GaussDB |
|--------|-------|---------|
| driver-class-name | `com.mysql.cj.jdbc.Driver` | `org.opengauss.Driver` |
| url | `jdbc:mysql://host:3306/db` | `jdbc:opengauss://host:5432/db?currentSchema=public` |
| database-platform | `MySQL8Dialect` | `org.hibernate.dialect.PostgreSQLDialect` |

### 4. SQL 语法转换

| MySQL | GaussDB |
|-------|---------|
| `` `Column` `` | `"column"` (反引号转双引号，大写转小写) |
| 字符串值 `"text"` | 字符串值 `'text'` |
| `IFNULL(a, b)` | `COALESCE(a, b)` |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` |
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:%s')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:SS')` |
| `ON DUPLICATE KEY UPDATE` | `ON CONFLICT (key) DO UPDATE SET` |
| `GROUP_CONCAT(col)` | `STRING_AGG(col::text, ',')` |
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` |
| `FROM_UNIXTIME(ts)` | `TO_TIMESTAMP(ts)` |
| `CURDATE()` | `CURRENT_DATE` |
| `DATE(CURDATE()+INTERVAL n DAY)` | `TO_CHAR(CURRENT_DATE + INTERVAL 'n DAY', 'YYYY-MM-DD')` |
| `DATE_ADD(date, INTERVAL n DAY)` | `TO_CHAR(CAST(date AS TIMESTAMP) + INTERVAL 'n DAY', 'YYYY-MM-DD')` |
| `DATE_SUB(date, INTERVAL n DAY)` | `TO_CHAR(CAST(date AS TIMESTAMP) - INTERVAL 'n DAY', 'YYYY-MM-DD')` |
| SELECT 非聚合列不在 GROUP BY 中 | 对非聚合列使用 `MAX(col)` 包装 |
| `SELECT EXISTS(...)` 返回 0/1 | `SELECT (EXISTS(...))::int` |
| `JSON_OBJECT(key, value, ...)` | `json_build_object(key, value, ...)` |
| `ANY_VALUE(col)` | `MAX(col)` |

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

删除 MySQL 特有语法: `ENGINE=InnoDB`, `DEFAULT CHARSET=utf8mb4`, `UNSIGNED`, `AUTO_INCREMENT=N`

### 6. 迁移完整性校验

转换完成后，执行以下检查确保没有遗漏：

#### 6.1 扫描残留 MySQL 语法

```bash
# 检查是否还有 MySQL 驱动引用
grep -r "mysql-connector-java" --include="pom.xml" --include="build.gradle"
grep -r "com.mysql" --include="*.java" --include="*.xml" --include="*.yml" --include="*.properties"

# 检查是否还有 MySQL JDBC URL
grep -r "jdbc:mysql" --include="*.yml" --include="*.properties" --include="*.xml"

# 检查是否还有 MySQL 方言
grep -r "MySQLDialect\|MySQL5Dialect\|MySQL8Dialect" --include="*.yml" --include="*.properties" --include="*.java"

# 检查是否还有反引号（MySQL 特有）
grep -r "\`" --include="*.xml" --include="*.sql"

# 检查 DML 中使用双引号包围字符串值（GaussDB 双引号仅用于标识符）
grep -rE "\"[^\"]+\"" --include="*.xml" --include="*.sql"

# 检查未转换的 MySQL 函数
grep -rE "IFNULL|DATE_FORMAT|GROUP_CONCAT|UNIX_TIMESTAMP|FROM_UNIXTIME|CURDATE|DATE_ADD|DATE_SUB|JSON_OBJECT|ANY_VALUE" --include="*.xml" --include="*.sql" --include="*.java"

# 检查 MySQL 特有的 DDL 语法
grep -rE "AUTO_INCREMENT|ENGINE=|UNSIGNED|CHARSET=" --include="*.sql"

# 检查 ON DUPLICATE KEY
grep -r "ON DUPLICATE KEY" --include="*.xml" --include="*.sql" --include="*.java"

# 检查 GROUP BY 语句（需人工确认非聚合列是否都已处理）
grep -rE "GROUP BY" --include="*.xml" --include="*.sql" --include="*.java"

# 检查 EXISTS 返回值（如需返回 0/1 需转换为 (EXISTS(...))::int）
grep -rE "SELECT\s+EXISTS" --include="*.xml" --include="*.sql" --include="*.java"
```

#### 6.2 校验清单

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| MySQL 驱动 | `grep -r "mysql-connector-java"` | 无匹配 |
| MySQL 驱动类 | `grep -r "com.mysql"` | 无匹配 |
| JDBC URL | `grep -r "jdbc:mysql"` | 无匹配 |
| MySQL 方言 | `grep -r "MySQLDialect"` | 无匹配 |
| 反引号 | `grep -r "\\\`" *.xml *.sql` | 无匹配 |
| IFNULL 函数 | `grep -r "IFNULL"` | 无匹配 |
| DATE_FORMAT | `grep -r "DATE_FORMAT"` | 无匹配 |
| AUTO_INCREMENT | `grep -r "AUTO_INCREMENT"` | 无匹配 |
| ON DUPLICATE KEY | `grep -r "ON DUPLICATE KEY"` | 无匹配 |

#### 6.3 生成校验报告

扫描完成后输出报告：

```
============ MySQL to GaussDB 迁移校验报告 ============

✅ 依赖配置
   - pom.xml: 已切换到 opengauss-jdbc
   - 无残留 MySQL 驱动引用

✅ 数据库配置
   - driver-class-name: org.opengauss.Driver
   - url: jdbc:opengauss://...
   - dialect: PostgreSQLDialect

⚠️ 待检查项（如有）
   - src/main/resources/mapper/UserMapper.xml:45 - 发现反引号
   - src/main/resources/db/init.sql:23 - 发现 AUTO_INCREMENT

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

### 8. 常见遗漏项

| 遗漏类型 | 常见位置 | 解决方法 |
|----------|----------|----------|
| GROUP BY 非聚合列 | 所有包含 GROUP BY 的查询 | 用 `MAX(col)` 包装非聚合列 |
| 硬编码 SQL | `@Query` 注解 | 搜索 `nativeQuery = true` |
| 动态 SQL | MyBatis `<if>` 标签内 | 逐个检查 Mapper.xml |
| 测试代码 | `src/test/**` | 同步修改测试配置 |
| 多环境配置 | `application-dev.yml` 等 | 检查所有 profile 配置 |
| 数据库初始化 | `schema.sql`, `data.sql` | Flyway/Liquibase 脚本 |
| 存储过程 | `*.sql` | 需手动重写，语法差异大 |
