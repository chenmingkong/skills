# 迁移完整性校验清单

转换完成后，执行以下检查确保没有遗漏。

---

## 1. 必须转换项（期望：无匹配）

### 1.1 标识符和字符串

```bash
# 检查反引号（普通字段去掉，关键字字段改为双引号）
grep -rn "\`" --include="*.xml" --include="*.java"

# 检查双引号字符串值（需区分字符串值和关键字字段）
grep -rnE "=\s*\"[^\"]+\"" --include="*.xml" --include="*.java"
```

### 1.2 通用函数

```bash
grep -rn "IFNULL" --include="*.xml" --include="*.java"
grep -rnE "\bIF\s*\(" --include="*.xml" --include="*.java"
grep -rn "GROUP_CONCAT" --include="*.xml" --include="*.java"
grep -rn "JSON_OBJECT" --include="*.xml" --include="*.java"
grep -rn "JSON_CONTAINS" --include="*.xml" --include="*.java"
grep -rn "ANY_VALUE" --include="*.xml" --include="*.java"
```

### 1.3 日期时间函数

```bash
grep -rn "DATE_FORMAT" --include="*.xml" --include="*.java"
grep -rn "STR_TO_DATE" --include="*.xml" --include="*.java"
grep -rn "UNIX_TIMESTAMP" --include="*.xml" --include="*.java"
grep -rn "FROM_UNIXTIME" --include="*.xml" --include="*.java"
grep -rn "CURDATE" --include="*.xml" --include="*.java"
grep -rn "CURTIME" --include="*.xml" --include="*.java"
grep -rn "SYSDATE" --include="*.xml" --include="*.java"
grep -rnE "DATE_ADD|DATE_SUB" --include="*.xml" --include="*.java"
grep -rn "DATEDIFF" --include="*.xml" --include="*.java"
grep -rn "TIMESTAMPDIFF" --include="*.xml" --include="*.java"
grep -rnE "\bDATE\s*\(" --include="*.xml" --include="*.java"
grep -rnE "\bTIME\s*\(" --include="*.xml" --include="*.java"
grep -rnE "\b(YEAR|MONTH|DAY|HOUR|MINUTE|SECOND)\s*\(" --include="*.xml" --include="*.java"
```

---

## 2. 需人工检查项

```bash
# 字段别名（需检查 <select> 的 resultType，只有 Map 类型需加双引号）
grep -rnE "\bAS\s+[a-zA-Z_][a-zA-Z0-9_]*\s*[,\s\n\r\)]" --include="*.xml"
grep -rn 'resultType="map"' --include="*.xml"  # 辅助：查找 map
grep -rn 'resultType="java.util.HashMap"' --include="*.xml"  # 辅助：查找 HashMap

grep -rn "LAST_INSERT_ID" --include="*.xml"  # 需扫描 DDL 获取序列名
grep -rn "ON DUPLICATE KEY" --include="*.xml"  # UPDATE 不能包含唯一索引字段
grep -rnE "ORDER\s+BY" --include="*.xml"  # 确认 NULLS FIRST/LAST
grep -rnE "GROUP\s+BY" --include="*.xml"  # 确认非聚合列用聚合函数
grep -rnE "INSERT\s+INTO|UPDATE\s+\w+\s+SET" --include="*.xml"  # 确认类型转换
```

---

## 3. 校验清单表

| 检查项 | 命令 | 期望结果 |
|--------|------|----------|
| 反引号 | `grep -rn "\`"` | 人工确认（关键字用双引号，普通字段去掉） |
| 双引号字符串 | `grep -rnE "=\s*\"[^\"]+\""` | 人工确认（区分字符串值和关键字字段） |
| IFNULL/IF/GROUP_CONCAT | `grep -rnE "IFNULL\|IF\(\|GROUP_CONCAT"` | 无匹配 |
| JSON_OBJECT/JSON_CONTAINS | `grep -rn "JSON_OBJECT\|JSON_CONTAINS"` | 无匹配 |
| DATE_FORMAT/STR_TO_DATE | `grep -rn "DATE_FORMAT\|STR_TO_DATE"` | 无匹配 |
| UNIX_TIMESTAMP/FROM_UNIXTIME | `grep -rn "UNIX_TIMESTAMP\|FROM_UNIXTIME"` | 无匹配 |
| CURDATE/CURTIME/SYSDATE | `grep -rn "CURDATE\|CURTIME\|SYSDATE"` | 无匹配 |
| DATE_ADD/DATE_SUB/DATEDIFF | `grep -rnE "DATE_ADD\|DATE_SUB\|DATEDIFF"` | 无匹配 |
| 字段别名 | `grep -rnE "\bAS\s+[a-zA-Z]"` | Map 类型加双引号 |
| ORDER BY | `grep -rnE "ORDER\s+BY"` | 确认 NULLS FIRST/LAST |
| GROUP BY | `grep -rnE "GROUP\s+BY"` | 确认聚合函数 |
| LAST_INSERT_ID | `grep -rn "LAST_INSERT_ID"` | 扫描 DDL 获取序列名 |
| INSERT/UPDATE 类型 | `grep -rn "INSERT\|UPDATE"` | 确认类型转换 |

---

## 4. 校验报告模板

扫描完成后，输出到 `dml_完整性校验报告.txt`：

```
============ DML 迁移校验报告 ============

【标识符和字符串】
✅ 普通字段反引号已去掉
✅ SQL 关键字字段已用双引号（order, desc, group, key, value, type 等）
✅ 字符串值已使用单引号
✅ 字段别名（Map 类型加双引号：map/HashMap）

【通用函数】
✅ IFNULL → COALESCE
✅ IF() → CASE WHEN
✅ GROUP_CONCAT → STRING_AGG
✅ JSON_OBJECT → json_build_object
✅ JSON_CONTAINS → ::jsonb @> ::jsonb
✅ ANY_VALUE → MAX

【日期时间函数】
✅ DATE_FORMAT → TO_CHAR
✅ STR_TO_DATE → TO_DATE/TO_TIMESTAMP
✅ UNIX_TIMESTAMP → EXTRACT(EPOCH FROM ...)
✅ FROM_UNIXTIME → TO_TIMESTAMP
✅ CURDATE/CURTIME/SYSDATE → CURRENT_DATE/CURRENT_TIME/NOW()
✅ DATE_ADD/DATE_SUB → INTERVAL
✅ DATEDIFF/TIMESTAMPDIFF → EXTRACT
✅ DATE/TIME/YEAR/MONTH/DAY → TO_CHAR/::time/EXTRACT

【类型转换】
✅ Java String → DB INT/JSON 已添加 ::int/::json
✅ Java Integer/Long → DB INT 无需转换

【特殊语法】
✅ LAST_INSERT_ID → currval('真实序列名')
✅ ON DUPLICATE KEY UPDATE（支持,不支持CONFLICT）
✅ ORDER BY 已添加 NULLS FIRST/LAST
✅ GROUP BY 非聚合列已用聚合函数包装

【待人工检查】
⚠️ SQL 关键字字段：X 处（需用双引号）
⚠️ 字段别名：X 处（Map 类型加双引号：map/HashMap）
⚠️ ORDER BY：X 处（需确认 NULLS）
⚠️ GROUP BY：X 处（需确认聚合函数）
⚠️ LAST_INSERT_ID：X 处（需获取序列名）
⚠️ ON DUPLICATE KEY：X 处（需确认无唯一索引）
⚠️ INSERT/UPDATE 类型转换：X 处（需对照 Java 类）

【统计】
扫描文件数：X 个 | 修改文件数：X 个 | 已转换项：X 处 | 待检查：X 处

============================================
```

---

## 5. 兼容的 MySQL 语法（无需检查）

以下语法 GaussDB 直接兼容，无需转换：

- `LIMIT offset, count` / `LIMIT count` - 分页语法
- `NOW()` - 当前时间
- `COALESCE()` - 空值处理
- `CONCAT()` - 字符串拼接
- `TRIM()`, `UPPER()`, `LOWER()` - 字符串函数
- `ABS()`, `ROUND()`, `CEIL()`, `FLOOR()` - 数学函数

---

## 6. 注意事项

### XML 转义字符保留

- `&lt;` 不需要改成 `<`
- `&gt;` 不需要改成 `>`
- `&amp;` 不需要改成 `&`

这些是 XML 的标准转义，在 MyBatis Mapper XML 中用于比较运算符，必须保留。

### 最小化修改原则

只修改需要转换的语法问题，保持文件其他内容不变，不要做多余的修改。
