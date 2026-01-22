# Claude Code 实践手册：从零创建自定义 Skill

本文记录了使用 Claude Code 创建一个 MySQL 到 GaussDB 数据库迁移 Skill 的完整过程，涵盖 Skill 开发规范、Git 操作和常见问题排查。

## 背景

项目需求：Java Spring Boot 服务要从 MySQL 切换到华为 GaussDB（OpenGauss），涉及大量重复性的代码修改工作。希望通过 Claude Code 的 Skill 功能，将迁移规则固化下来，方便后续项目复用。

## 一、Claude Code Skills 基础

### 1.1 什么是 Skills

Skills 是 Claude Code 的扩展机制，可以理解为"自定义指令集"。通过编写 Skill 文件，可以让 Claude 按照预设的规则执行特定任务。

### 1.2 Skills 目录结构

```
项目根目录/
└── .claude/
    └── skills/
        └── skill-name/        # 目录名即为 skill 命令名
            └── SKILL.md       # 必须是这个文件名
```

**关键点：**
- 必须是目录结构，不能直接放 `.md` 文件
- 文件名必须是 `SKILL.md`（大写）
- 目录名决定了调用时的命令名，如 `/mysql-to-gaussdb`

### 1.3 SKILL.md 文件格式

```markdown
---
name: skill-name
description: 这里写 skill 的功能描述，Claude 会根据这个判断何时自动调用
argument-hint: "[可选参数提示]"
---

# Skill 标题

具体的指令内容...
```

**YAML Frontmatter 字段说明：**

| 字段 | 必填 | 说明 |
|------|------|------|
| name | 否 | 默认使用目录名 |
| description | 推荐 | 描述 skill 功能，影响自动调用 |
| argument-hint | 否 | 参数提示，如 `[文件路径]` |
| disable-model-invocation | 否 | 设为 true 则仅手动调用 |

## 二、实战：创建数据库迁移 Skill

### 2.1 创建目录结构

```bash
mkdir -p .claude/skills/mysql-to-gaussdb
```

### 2.2 编写 SKILL.md

```markdown
---
name: mysql-to-gaussdb
description: 将 Spring Boot 项目从 MySQL 迁移到 GaussDB，自动转换依赖、配置和 SQL 语法
argument-hint: "[项目目录]"
---

# MySQL to GaussDB 迁移

## 执行步骤

### 1. 扫描文件
**/pom.xml, **/build.gradle
**/application*.yml, **/application*.properties
**/*Mapper.xml
**/*.sql

### 2. 修改依赖
...

### 3. SQL 语法转换规则
| MySQL | GaussDB |
|-------|---------|
| IFNULL(a, b) | COALESCE(a, b) |
| ... | ... |
```

内容要点：
- 明确执行步骤，Claude 会按顺序执行
- 转换规则用表格呈现，清晰易读
- 包含验证步骤，确保转换正确

### 2.3 验证 Skill

Skill 文件创建后，需要**重启 Claude Code 会话**才能加载：

```bash
# 退出当前会话
/exit

# 重新进入项目目录
claude

# 测试 skill
/mysql-to-gaussdb
```

## 三、Git 操作与 GitHub 推送

### 3.1 初始化仓库

```bash
git init
git remote add origin https://github.com/username/repo.git
```

### 3.2 GitHub 认证

GitHub 已不支持密码认证，需要使用 Personal Access Token：

```bash
# 方式一：直接在 URL 中使用 token
git remote set-url origin https://<TOKEN>@github.com/username/repo.git

# 方式二：使用 SSH
git remote set-url origin git@github.com:username/repo.git
```

### 3.3 提交与推送

```bash
git add .claude/skills/mysql-to-gaussdb/SKILL.md
git commit -m "Add mysql-to-gaussdb migration skill"
git branch -M main
git push -u origin main
```

### 3.4 忽略本地配置文件

`.claude/settings.local.json` 包含本地权限配置，不应提交到仓库：

```bash
# 从 git 中移除（保留本地文件）
git rm --cached .claude/settings.local.json

# 添加到 .gitignore
echo ".claude/settings.local.json" >> .gitignore

git add .gitignore
git commit -m "Add gitignore for local settings"
git push
```

## 四、踩坑记录

### 4.1 Skill 文件格式错误

**错误做法：**
```
.claude/skills/mysql-to-gaussdb.md  # 直接放 md 文件
```

**正确做法：**
```
.claude/skills/mysql-to-gaussdb/SKILL.md  # 目录 + SKILL.md
```

### 4.2 Skill 不生效

现象：创建了 Skill 文件但 `/skill-name` 命令无效

排查步骤：
1. 检查目录结构是否正确
2. 检查 SKILL.md 文件名大小写
3. 检查 YAML frontmatter 格式（`---` 开头和结尾）
4. **重启 Claude Code 会话**

### 4.3 GitHub 推送认证失败

```
remote: Invalid username or token.
fatal: Authentication failed
```

解决：使用 Personal Access Token 替代密码，在 GitHub Settings → Developer settings → Personal access tokens 生成。

### 4.4 Skills 功能重复

最初创建了两个 Skill：
- `mysql-to-gaussdb`：迁移指南
- `convert-mysql-gaussdb`：代码转换

问题：内容大量重复，维护成本高

解决：合并为单个 Skill，既包含规则说明又能执行转换

## 五、最佳实践

### 5.1 Skill 设计原则

1. **单一职责**：一个 Skill 解决一类问题
2. **步骤清晰**：按顺序列出执行步骤
3. **规则明确**：转换规则用表格，避免歧义
4. **包含验证**：最后加上验证步骤

### 5.2 项目结构建议

```
project/
├── .claude/
│   ├── settings.local.json  # 本地配置，不提交
│   └── skills/
│       └── your-skill/
│           └── SKILL.md
├── .gitignore               # 忽略 settings.local.json
└── src/
```

### 5.3 .gitignore 配置

```gitignore
# Claude Code 本地配置
.claude/settings.local.json
```

## 六、完整 Skill 示例

项目地址：https://github.com/chenmingkong/skills

```markdown
---
name: mysql-to-gaussdb
description: 将 Spring Boot 项目从 MySQL 迁移到 GaussDB，自动转换依赖、配置和 SQL 语法
argument-hint: "[项目目录]"
---

# MySQL to GaussDB 迁移

自动扫描并转换 Spring Boot 项目中的 MySQL 代码为 GaussDB 兼容代码。

## 执行步骤

### 1. 扫描文件
**/pom.xml, **/build.gradle
**/application*.yml, **/application*.properties
**/*Mapper.xml
**/*.sql
**/*Repository.java, **/*Dao.java

### 2. 修改依赖
pom.xml: mysql:mysql-connector-java → org.opengauss:opengauss-jdbc:5.0.0

### 3. 修改配置
driver-class-name: com.mysql.cj.jdbc.Driver → org.opengauss.Driver
url: jdbc:mysql://host:3306/db → jdbc:opengauss://host:5432/db?currentSchema=public
database-platform: MySQL8Dialect → org.hibernate.dialect.PostgreSQLDialect

### 4. SQL 语法转换
| MySQL | GaussDB |
|-------|---------|
| `column` | column 或 "column" |
| IFNULL(a, b) | COALESCE(a, b) |
| IF(cond, a, b) | CASE WHEN cond THEN a ELSE b END |
| DATE_FORMAT(d, '%Y-%m-%d') | TO_CHAR(d, 'YYYY-MM-DD') |
| LIMIT 5, 10 | LIMIT 10 OFFSET 5 |
| ON DUPLICATE KEY UPDATE | ON CONFLICT (key) DO UPDATE SET |

### 5. DDL 类型转换
| MySQL | GaussDB |
|-------|---------|
| BIGINT AUTO_INCREMENT | BIGSERIAL |
| INT AUTO_INCREMENT | SERIAL |
| TINYINT(1) | BOOLEAN |
| DATETIME | TIMESTAMP |
| BLOB | BYTEA |
| LONGTEXT | TEXT |

删除: ENGINE=InnoDB, DEFAULT CHARSET=utf8mb4, UNSIGNED

### 6. 验证
mvn clean compile
mvn test
```

## 参考资料

- [Claude Code 官方文档](https://docs.anthropic.com/claude-code)
- [GaussDB 官方文档](https://support.huaweicloud.com/gaussdb)
- [OpenGauss JDBC 驱动](https://opengauss.org)
