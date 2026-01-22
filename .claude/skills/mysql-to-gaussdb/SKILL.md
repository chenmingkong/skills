---
name: mysql-to-gaussdb
description: 查看 MySQL 到 GaussDB 的迁移指南，包含 SQL 转换规则、数据类型映射和配置示例
---

# MySQL to GaussDB 迁移指南

## Maven 依赖

移除 MySQL:
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

添加 GaussDB:
```xml
<dependency>
    <groupId>org.opengauss</groupId>
    <artifactId>opengauss-jdbc</artifactId>
    <version>5.0.0</version>
</dependency>
```

## 配置修改

### application.yml
```yaml
spring:
  datasource:
    driver-class-name: org.opengauss.Driver
    url: jdbc:opengauss://localhost:5432/mydb?currentSchema=public
    username: gaussdb_user
    password: password
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
```

## SQL 语法转换

| MySQL | GaussDB |
|-------|---------|
| `` `column` `` | `"column"` 或直接 column |
| `AUTO_INCREMENT` | `SERIAL` / `BIGSERIAL` |
| `IFNULL(a, b)` | `COALESCE(a, b)` |
| `IF(cond, a, b)` | `CASE WHEN cond THEN a ELSE b END` |
| `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` |
| `LIMIT 5, 10` | `LIMIT 10 OFFSET 5` |
| `ON DUPLICATE KEY UPDATE` | `ON CONFLICT (key) DO UPDATE SET` |
| `TINYINT(1)` | `BOOLEAN` |
| `DATETIME` | `TIMESTAMP` |
| `LONGTEXT` | `TEXT` |
| `BLOB` | `BYTEA` |

## 驱动类名映射

- `com.mysql.jdbc.Driver` → `org.opengauss.Driver`
- `com.mysql.cj.jdbc.Driver` → `org.opengauss.Driver`

## Hibernate 方言

- `org.hibernate.dialect.MySQL5Dialect` → `org.hibernate.dialect.PostgreSQLDialect`
- `org.hibernate.dialect.MySQL8Dialect` → `org.hibernate.dialect.PostgreSQLDialect`
