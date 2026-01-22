---
name: convert-mysql-gaussdb
description: 自动扫描并转换 Spring Boot 项目中的 MySQL 代码为 GaussDB 兼容代码
argument-hint: "[项目目录路径]"
---

# Spring Boot MySQL to GaussDB 代码转换器

将当前项目或指定目录的 Spring Boot 项目从 MySQL 转换为 GaussDB。

## 执行步骤

### 1. 扫描项目文件

扫描以下类型文件：
- `**/pom.xml` - Maven 依赖
- `**/build.gradle` - Gradle 依赖
- `**/application*.yml` `**/application*.properties` - 配置文件
- `**/*Mapper.xml` - MyBatis 映射文件
- `**/*.sql` - SQL 脚本
- `**/*Repository.java` `**/*Dao.java` - 检查 @Query 原生 SQL

### 2. 修改依赖

**pom.xml**: 将 `mysql:mysql-connector-java` 替换为 `org.opengauss:opengauss-jdbc:5.0.0`

**build.gradle**: 将 `mysql:mysql-connector-java` 替换为 `org.opengauss:opengauss-jdbc:5.0.0`

### 3. 修改配置

- `driver-class-name`: `com.mysql.cj.jdbc.Driver` → `org.opengauss.Driver`
- `url`: `jdbc:mysql://host:3306/db` → `jdbc:opengauss://host:5432/db?currentSchema=public`
- `database-platform`: `MySQL*Dialect` → `org.hibernate.dialect.PostgreSQLDialect`

### 4. SQL 语法转换

对 Mapper.xml 和 .sql 文件执行：

```
反引号 `name` → name 或 "name"
IFNULL(a, b) → COALESCE(a, b)
IF(cond, a, b) → CASE WHEN cond THEN a ELSE b END
DATE_FORMAT(d, '%Y-%m-%d') → TO_CHAR(d, 'YYYY-MM-DD')
DATE_FORMAT(d, '%Y-%m-%d %H:%i:%s') → TO_CHAR(d, 'YYYY-MM-DD HH24:MI:SS')
LIMIT offset, count → LIMIT count OFFSET offset
GROUP_CONCAT(col) → STRING_AGG(col::text, ',')
UNIX_TIMESTAMP() → EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER
FROM_UNIXTIME(ts) → TO_TIMESTAMP(ts)
```

### 5. DDL 类型转换

```
BIGINT AUTO_INCREMENT → BIGSERIAL
INT AUTO_INCREMENT → SERIAL
TINYINT(1) → BOOLEAN
TINYINT → SMALLINT
DATETIME → TIMESTAMP
DOUBLE → DOUBLE PRECISION
BLOB/LONGBLOB → BYTEA
LONGTEXT/MEDIUMTEXT → TEXT
```

删除: `ENGINE=InnoDB`, `DEFAULT CHARSET=utf8mb4`, `UNSIGNED`, `AUTO_INCREMENT=N`

### 6. 验证

完成转换后运行:
- `mvn clean compile` 检查编译
- `mvn test` 运行测试
