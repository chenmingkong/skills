# 标识符与字符串转换规则

## 1. 普通标识符（表名、字段名）

**规则：去掉反引号**

```sql
-- MySQL
SELECT `id`, `user_name` FROM `user` WHERE `status` = 1;
SELECT `User`.`Name` FROM `User`;

-- GaussDB
SELECT id, user_name FROM user WHERE status = 1;
SELECT User.Name FROM User;
```

**转换规则：**
- `` `普通字段` `` → `普通字段`（去反引号）
- 如果不是 SQL 关键字，直接去掉反引号即可

---

## 2. SQL 关键字字段（重要）

**规则：关键字必须用双引号包裹**

表名或字段名是 SQL 关键字时，MySQL 用反引号，GaussDB 用双引号。

**常见关键字：**
`order`, `desc`, `asc`, `group`, `key`, `value`, `type`, `comment`, `index`, `rank`, `level`, `user`, `position`, `option`, `select`, `insert`, `update`, `delete`, `where`, `from`, `table`, `column`, `check`, `default`

```sql
-- MySQL
SELECT `id`, `order`, `desc`, `key`, `value` FROM `order`;
SELECT t.`id`, t.`group`, t.`type` FROM `config` t WHERE t.`level` = 1;

-- GaussDB
SELECT id, "order", "desc", "key", "value" FROM "order";
SELECT t.id, t."group", t."type" FROM config t WHERE t."level" = 1;
```

**转换规则：**
- `` `关键字字段` `` → `"关键字字段"`（改为双引号）
- 检查常见关键字列表进行匹配

---

## 3. 字段别名（重要：根据 resultType 决定）

**规则：只有 resultType 为 Map/HashMap 时，别名才需要加双引号**

### 3.1 resultType 为 Map/HashMap - 别名加双引号

**必须加双引号的类型：**
- `resultType="map"`
- `resultType="java.util.Map"`
- `resultType="java.util.HashMap"`

```xml
<!-- GaussDB（✅ resultType="map"，别名加双引号）-->
<select id="getUserStats" resultType="map">
    SELECT user_name AS "userName", COUNT(*) AS "totalCount" FROM user;
</select>

<!-- GaussDB（✅ resultType="java.util.HashMap"，别名加双引号）-->
<select id="getUserMap" resultType="java.util.HashMap">
    SELECT user_id AS "userId", user_name AS "userName" FROM user;
</select>
```

**原因：** 别名作为 Map 的 key，需双引号保持驼峰命名。

### 3.2 其他情况 - 别名不加引号

当 resultType 为实体类或使用 resultMap 时，别名不需要加引号：

```xml
<!-- GaussDB（❌ resultType 为实体类，别名不加引号）-->
<select id="getUser" resultType="com.example.entity.User">
    SELECT user_id AS userId, user_name AS userName FROM user;
</select>

<!-- GaussDB（❌ 使用 resultMap，别名不加引号）-->
<select id="getUserList" resultMap="userResultMap">
    SELECT user_id AS userId, user_name AS userName FROM user;
</select>
```

**原因：** MyBatis 通过反射或映射配置，不依赖别名的大小写。

**转换规则：**
1. 查找 `<select>` 标签的 `resultType` 属性
2. 如果是 `map`、`java.util.Map`、`java.util.HashMap`，则所有别名加双引号
3. 否则，别名不加引号

---

## 4. 表别名

**规则：表别名不需要引号**

```sql
-- MySQL & GaussDB（都不需要引号）
FROM user u
FROM orders o
LEFT JOIN user_info ui ON u.id = ui.user_id
```

---

## 5. 字符串值（必须用单引号）

**规则：GaussDB 中字符串值必须用单引号**

```sql
-- MySQL
WHERE name = "张三" AND status = "active"
WHERE name LIKE "%test%"
WHERE code IN ("A", "B", "C")

-- GaussDB
WHERE name = '张三' AND status = 'active'
WHERE name LIKE '%test%'
WHERE code IN ('A', 'B', 'C')
```

**转换规则：**
- 字符串值的双引号 `"..."` → 单引号 `'...'`
- 注意区分字符串值和关键字字段（关键字字段用双引号）

---

## 快速参考表

| 场景 | MySQL | GaussDB | 说明 |
|------|-------|---------|------|
| 表名/字段名（普通） | `` `user` ``, `` `name` `` | `user`, `name` | 去掉反引号 |
| 表名/字段名（关键字） | `` `order` ``, `` `desc` `` | `"order"`, `"desc"` | 关键字用双引号 |
| 字段别名（Map） | `AS userName` | `AS "userName"` | resultType="map/HashMap" 加双引号 |
| 字段别名（实体类） | `AS userName` | `AS userName` | resultType 为实体类不加引号 |
| 表别名 | `FROM user u` | `FROM user u` | 不需要引号 |
| 字符串值 | `"张三"`, `"%test%"` | `'张三'`, `'%test%'` | 必须用单引号 |

---

## 处理流程

1. **扫描 SQL 语句**
2. **处理反引号**：
   - 判断是否为 SQL 关键字
   - 是 → 改为双引号 `"关键字"`
   - 否 → 直接去掉
3. **处理双引号**：
   - 判断是字符串值还是关键字字段
   - 字符串值 → 改为单引号 `'值'`
   - 关键字字段 → 保留双引号 `"字段"`
4. **处理别名**：
   - 查找所在 `<select>` 的 `resultType`
   - 如果是 Map/HashMap → 别名加双引号
   - 否则 → 别名不加引号
5. **表别名不处理**
