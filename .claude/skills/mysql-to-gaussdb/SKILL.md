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
| `` `column` `` | `column` 或 `"column"` |
| `IFNULL(a, b)` | `COALESCE(a, b)` |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` |
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` |
| `DATE_FORMAT(d, '%Y-%m-%d %H:%i:%s')` | `TO_CHAR(d, 'YYYY-MM-DD HH24:MI:SS')` |
| `LIMIT 5, 10` | `LIMIT 10 OFFSET 5` |
| `ON DUPLICATE KEY UPDATE` | `ON CONFLICT (key) DO UPDATE SET` |
| `GROUP_CONCAT(col)` | `STRING_AGG(col::text, ',')` |
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` |
| `FROM_UNIXTIME(ts)` | `TO_TIMESTAMP(ts)` |

### 5. DDL 类型转换

| MySQL | GaussDB |
|-------|---------|
| `BIGINT AUTO_INCREMENT` | `BIGSERIAL` |
| `INT AUTO_INCREMENT` | `SERIAL` |
| `TINYINT(1)` | `BOOLEAN` |
| `TINYINT` | `SMALLINT` |
| `DATETIME` | `TIMESTAMP` |
| `DOUBLE` | `DOUBLE PRECISION` |
| `BLOB` / `LONGBLOB` | `BYTEA` |
| `LONGTEXT` / `MEDIUMTEXT` | `TEXT` |

删除 MySQL 特有语法: `ENGINE=InnoDB`, `DEFAULT CHARSET=utf8mb4`, `UNSIGNED`, `AUTO_INCREMENT=N`

### 6. 验证

```bash
mvn clean compile  # 检查编译
mvn test           # 运行测试
```
