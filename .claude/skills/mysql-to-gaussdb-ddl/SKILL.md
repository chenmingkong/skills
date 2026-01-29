---
name: mysql-to-gaussdb-ddl
description: 将 MySQL DDL 语句转换为 GaussDB 兼容语法（建表、数据类型、索引、约束等）
argument-hint: "[SQL文件或目录]"
---

# MySQL to GaussDB DDL 转换

将 MySQL 的 DDL（数据定义语言）转换为 GaussDB 兼容语法，包括建表语句、数据类型、索引、约束等。

## 扫描文件

```
**/*.sql                              # SQL 脚本
**/schema.sql, **/data.sql           # 数据库初始化脚本
**/migration/**/*.sql                # 迁移脚本
```

**重要：必须对所有匹配的文件进行完整扫描，逐行检查并转换，确保文件从头到尾每一行都经过处理，不要遗漏文件后半部分的内容。**

## 输出文件

转换完成后，将转换后的 SQL 文件输出到原文件同一目录，文件名为原文件名 + `_gauss` 后缀：

| 原文件 | 转换后文件 |
|--------|-----------|
| `schema.sql` | `schema_gauss.sql` |
| `data.sql` | `data_gauss.sql` |
| `migration/V1__init.sql` | `migration/V1__init_gauss.sql` |

**注意：** 原文件保持不变，转换后的内容写入新文件。

## 1. 数据类型转换

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `BIGINT AUTO_INCREMENT` | `BIGINT` + 序列 | 自增大整数 |
| `INT AUTO_INCREMENT` | `INTEGER` + 序列 | 自增整数 |
| `TINYINT(1)` | `BOOLEAN` | 布尔类型 |
| `TINYINT` | `int2` | 小整数 |
| `SMALLINT` | `int2` | 小整数 |
| `MEDIUMINT` | `INTEGER` | 中整数 |
| `INT` / `INTEGER` | `INTEGER` | 整数 |
| `BIGINT` | `BIGINT` | 大整数 |
| `FLOAT` | `float8` | 浮点数 |
| `DOUBLE` | `float8` | 双精度浮点 |
| `DOUBLE(m,n)` | `numeric(m,n)` | 带精度双精度 |
| `DECIMAL(m,n)` | `numeric(m,n)` | 定点数 |
| `VARCHAR(n)` | `NVARCHAR2(n)` | 变长字符串 |
| `CHAR(n)` | `CHAR(n)` | 定长字符串 |
| `TEXT` | `TEXT` | 文本 |
| `LONGTEXT` | `TEXT` | 长文本 |
| `MEDIUMTEXT` | `TEXT` | 中文本 |
| `TINYTEXT` | `TEXT` | 短文本 |
| `BLOB` | `BYTEA` | 二进制 |
| `LONGBLOB` | `BYTEA` | 长二进制 |
| `MEDIUMBLOB` | `BYTEA` | 中二进制 |
| `TINYBLOB` | `BYTEA` | 短二进制 |
| `DATETIME` | `timestamp(0)` | 日期时间 |
| `TIMESTAMP` | `timestamp(0)` | 时间戳 |
| `DATE` | `DATE` | 日期 |
| `TIME` | `TIME` | 时间 |
| `YEAR` | `int2` | 年份 |
| `JSON` | `JSON` | JSON类型 |
| `ENUM(...)` | `VARCHAR(50)` + CHECK | 枚举（需手动处理） |
| `SET(...)` | `VARCHAR(255)` | 集合（需手动处理） |

## 2. AUTO_INCREMENT 转换为序列

GaussDB 不支持 AUTO_INCREMENT，需创建序列实现自增：

```sql
-- MySQL
CREATE TABLE user (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50)
);

-- GaussDB
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    name NVARCHAR2(50)
);

-- 创建序列
CREATE SEQUENCE user_id_seq;

-- 设置序列起始值为当前最大值+1（数据迁移后执行）
SELECT setval('user_id_seq', COALESCE((SELECT MAX(id) FROM user), 0) + 1, false);

-- 设置列默认值为序列
ALTER TABLE user ALTER COLUMN id SET DEFAULT nextval('user_id_seq');
```

## 3. 标识符转换（表名、字段名）

MySQL 使用反引号 `` ` `` 包裹标识符，GaussDB 使用双引号 `"`：

```sql
-- MySQL
CREATE TABLE `User` (
    `ID` BIGINT,
    `User_Name` VARCHAR(50)
);

-- GaussDB（标识符必须转小写）
CREATE TABLE "user" (
    "id" BIGINT,
    "user_name" NVARCHAR2(50)
);
```

**转换规则：**
- `` `identifier` `` → `"identifier"` （反引号替换为双引号）
- `` `TableName` `` → `"tablename"` （大写必须转为小写）
- `` `Column_Name` `` → `"column_name"` （大写必须转为小写）

## 4. COMMENT 注释转换

GaussDB 不支持在字段定义中声明 COMMENT，需单独声明：

```sql
-- MySQL
CREATE TABLE user (
    id BIGINT COMMENT '用户ID',
    name VARCHAR(50) COMMENT '用户名',
    status TINYINT COMMENT '状态：0-禁用，1-启用'
) COMMENT='用户表';

-- GaussDB
CREATE TABLE user (
    id BIGINT,
    name NVARCHAR2(50),
    status int2
);
COMMENT ON TABLE user IS '用户表';
COMMENT ON COLUMN user.id IS '用户ID';
COMMENT ON COLUMN user.name IS '用户名';
COMMENT ON COLUMN user.status IS '状态：0-禁用，1-启用';
```

## 5. 索引名称转换

GaussDB 同一 schema 中不允许索引名重复，需添加表名前缀：

```sql
-- MySQL
CREATE INDEX idx_name ON user(name);
CREATE INDEX idx_name ON product(name);
CREATE UNIQUE INDEX uk_email ON user(email);

-- GaussDB（索引名添加表名前缀）
CREATE INDEX user_idx_name ON user(name);
CREATE INDEX product_idx_name ON product(name);
CREATE UNIQUE INDEX user_uk_email ON user(email);
```

## 6. 时间字段默认值转换

GaussDB 不支持零值时间和 ON UPDATE CURRENT_TIMESTAMP：

```sql
-- MySQL
CREATE TABLE user (
    id BIGINT,
    create_time DATETIME DEFAULT '0000-00-00 00:00:00',
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- GaussDB
CREATE TABLE user (
    id BIGINT,
    create_time timestamp(0) DEFAULT CURRENT_TIMESTAMP,
    update_time timestamp(0) DEFAULT CURRENT_TIMESTAMP
);

-- ON UPDATE CURRENT_TIMESTAMP 需使用触发器实现
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.update_time = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_user_update_time
    BEFORE UPDATE ON user
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();
```

## 7. 删除 MySQL 特有语法

以下 MySQL 特有语法需删除：

| 需删除的语法 | 说明 |
|-------------|------|
| `ENGINE=InnoDB` | 存储引擎 |
| `ENGINE=MyISAM` | 存储引擎 |
| `DEFAULT CHARSET=utf8mb4` | 字符集 |
| `COLLATE=utf8mb4_unicode_ci` | 排序规则 |
| `UNSIGNED` | 无符号（GaussDB 不支持） |
| `AUTO_INCREMENT=N` | 自增起始值 |
| `ON UPDATE CURRENT_TIMESTAMP` | 自动更新时间 |
| `ROW_FORMAT=DYNAMIC` | 行格式 |

**示例：**

```sql
-- MySQL
CREATE TABLE user (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    age TINYINT UNSIGNED,
    balance DECIMAL(10,2) UNSIGNED
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 AUTO_INCREMENT=1000;

-- GaussDB
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    age int2,
    balance numeric(10,2)
);
-- 创建序列并设置起始值为 1000
CREATE SEQUENCE user_id_seq START WITH 1000;
ALTER TABLE user ALTER COLUMN id SET DEFAULT nextval('user_id_seq');
```

## 8. 主键和约束转换

```sql
-- MySQL
CREATE TABLE order_item (
    order_id BIGINT,
    product_id BIGINT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(id)
);

-- GaussDB（基本兼容，注意标识符大小写）
CREATE TABLE order_item (
    order_id BIGINT,
    product_id BIGINT,
    quantity INTEGER,
    PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

## 9. 迁移完整性校验

转换完成后，执行以下检查确保没有遗漏。

### 9.1 扫描残留 MySQL 语法

```bash
# ========== DDL 语法检查 ==========
# 检查 AUTO_INCREMENT
grep -rE "AUTO_INCREMENT" --include="*.sql"

# 检查存储引擎
grep -rE "ENGINE\s*=" --include="*.sql"

# 检查 UNSIGNED
grep -rE "\bUNSIGNED\b" --include="*.sql"

# 检查字符集和排序规则
grep -rE "CHARSET\s*=|COLLATE\s*=" --include="*.sql"

# 检查 ON UPDATE CURRENT_TIMESTAMP
grep -rE "ON\s+UPDATE\s+CURRENT_TIMESTAMP" --include="*.sql"

# 检查反引号（MySQL 特有）
grep -r "\`" --include="*.sql"

# 检查零值时间默认值
grep -rE "DEFAULT\s*'0+(-0+){2}" --include="*.sql"

# 检查内联 COMMENT（需转为 COMMENT ON COLUMN）
grep -rE "COMMENT\s*'" --include="*.sql"

# 检查 VARCHAR（应转为 NVARCHAR2）
grep -rE "\bVARCHAR\s*\(" --include="*.sql"

# 检查索引名是否包含表名前缀
grep -rE "CREATE\s+(UNIQUE\s+)?INDEX\s+idx_" --include="*.sql"

# 检查 ROW_FORMAT
grep -rE "ROW_FORMAT\s*=" --include="*.sql"
```

### 9.2 校验清单

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| AUTO_INCREMENT | `grep -r "AUTO_INCREMENT"` | 无匹配 |
| ENGINE= | `grep -rE "ENGINE\s*="` | 无匹配 |
| UNSIGNED | `grep -r "UNSIGNED"` | 无匹配 |
| CHARSET= | `grep -rE "CHARSET\s*="` | 无匹配 |
| COLLATE= | `grep -rE "COLLATE\s*="` | 无匹配 |
| ROW_FORMAT= | `grep -rE "ROW_FORMAT\s*="` | 无匹配 |
| ON UPDATE CURRENT_TIMESTAMP | `grep -r "ON UPDATE CURRENT_TIMESTAMP"` | 无匹配 |
| 反引号 | `grep -r "\`"` | 无匹配 |
| 零值时间 | `grep -rE "DEFAULT.*0000-00-00"` | 无匹配 |
| 内联 COMMENT | `grep -rE "COMMENT\s*'"` | 无匹配 |
| VARCHAR | `grep -rE "\bVARCHAR\s*\("` | 无匹配（应为 NVARCHAR2） |
| 索引名前缀 | `grep -rE "CREATE\s+(UNIQUE\s+)?INDEX\s+idx_"` | 应含表名前缀 |

### 9.3 生成校验报告

扫描完成后，将报告输出到项目根目录下的 `ddl_完整性校验报告.txt` 文件：

```
============ DDL 迁移校验报告 ============

✅ 数据类型
   - VARCHAR 已转为 NVARCHAR2
   - DATETIME/TIMESTAMP 已转为 timestamp(0)
   - TINYINT 已转为 int2/BOOLEAN
   - DOUBLE 已转为 float8/numeric

✅ 自增列
   - AUTO_INCREMENT 已转为序列
   - 序列起始值已正确设置

✅ 约束和索引
   - 索引名已添加表名前缀
   - 主键和外键语法正确

✅ 注释
   - 内联 COMMENT 已转为 COMMENT ON COLUMN
   - 表注释已转为 COMMENT ON TABLE

✅ 已删除 MySQL 特有语法
   - 无 ENGINE=
   - 无 CHARSET=
   - 无 UNSIGNED
   - 无 ON UPDATE CURRENT_TIMESTAMP

⚠️ 待人工检查项（如有）
   - schema.sql:45 - 发现反引号
   - migration/V1.sql:23 - 发现 AUTO_INCREMENT

❌ 未通过项（如有）
   - 仍存在 VARCHAR（应为 NVARCHAR2）
   - 仍存在内联 COMMENT

============ 统计 ============
扫描文件: XX 个
已转换: XX 处
待处理: XX 处
```

## 10. 完整转换示例

```sql
-- ========== MySQL 原始 DDL ==========
CREATE TABLE `User` (
    `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY COMMENT '用户ID',
    `user_name` VARCHAR(50) NOT NULL COMMENT '用户名',
    `email` VARCHAR(100) COMMENT '邮箱',
    `age` TINYINT UNSIGNED COMMENT '年龄',
    `balance` DOUBLE(10,2) DEFAULT 0.00 COMMENT '余额',
    `status` TINYINT(1) DEFAULT 1 COMMENT '状态',
    `create_time` DATETIME DEFAULT '0000-00-00 00:00:00' COMMENT '创建时间',
    `update_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    UNIQUE KEY `uk_email` (`email`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表' AUTO_INCREMENT=1000;

-- ========== GaussDB 转换后 DDL ==========
CREATE TABLE "user" (
    "id" BIGINT PRIMARY KEY,
    "user_name" NVARCHAR2(50) NOT NULL,
    "email" NVARCHAR2(100),
    "age" int2,
    "balance" numeric(10,2) DEFAULT 0.00,
    "status" BOOLEAN DEFAULT true,
    "create_time" timestamp(0) DEFAULT CURRENT_TIMESTAMP,
    "update_time" timestamp(0) DEFAULT CURRENT_TIMESTAMP
);

-- 创建序列
CREATE SEQUENCE user_id_seq START WITH 1000;
ALTER TABLE "user" ALTER COLUMN "id" SET DEFAULT nextval('user_id_seq');

-- 创建索引（名称添加表名前缀）
CREATE UNIQUE INDEX user_uk_email ON "user"("email");
CREATE INDEX user_idx_status ON "user"("status");

-- 添加注释
COMMENT ON TABLE "user" IS '用户表';
COMMENT ON COLUMN "user"."id" IS '用户ID';
COMMENT ON COLUMN "user"."user_name" IS '用户名';
COMMENT ON COLUMN "user"."email" IS '邮箱';
COMMENT ON COLUMN "user"."age" IS '年龄';
COMMENT ON COLUMN "user"."balance" IS '余额';
COMMENT ON COLUMN "user"."status" IS '状态';
COMMENT ON COLUMN "user"."create_time" IS '创建时间';
COMMENT ON COLUMN "user"."update_time" IS '更新时间';

-- 创建更新时间触发器
CREATE OR REPLACE FUNCTION update_user_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW."update_time" = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_user_update_time
    BEFORE UPDATE ON "user"
    FOR EACH ROW
    EXECUTE FUNCTION update_user_timestamp();
```
