# 日期时间函数转换规则

## 格式符映射表

**DATE_FORMAT / TO_CHAR 格式符对照：**

| MySQL | GaussDB | 说明 |
|-------|---------|------|
| `%Y` | `YYYY` | 4 位年份 |
| `%y` | `YY` | 2 位年份 |
| `%m` | `MM` | 月份（01-12） |
| `%d` | `DD` | 日期（01-31） |
| `%H` | `HH24` | 小时（00-23） |
| `%h` | `HH12` | 小时（01-12） |
| `%i` | `MI` | 分钟（00-59） |
| `%s` | `SS` | 秒（00-59） |

---

## 1. 当前时间函数

| MySQL | GaussDB | 检测模式 |
|-------|---------|---------|
| `NOW()`, `SYSDATE()` | `NOW()` | `SYSDATE\s*\(` |
| `CURDATE()` | `CURRENT_DATE` | `CURDATE\s*\(` |
| `CURTIME()` | `CURRENT_TIME` | `CURTIME\s*\(` |

```sql
-- MySQL
SELECT NOW(), CURDATE(), CURTIME(), SYSDATE() FROM dual;

-- GaussDB
SELECT NOW(), CURRENT_DATE, CURRENT_TIME, NOW() FROM dual;
```

---

## 2. DATE_FORMAT 转换

**规则：`DATE_FORMAT(date, format)` → `TO_CHAR(date, format)`**

需要转换格式符：`%Y`→`YYYY`, `%m`→`MM`, `%d`→`DD`, `%H`→`HH24`, `%i`→`MI`, `%s`→`SS`

```sql
-- MySQL
SELECT DATE_FORMAT(create_time, '%Y-%m-%d') AS date_str FROM orders;
SELECT DATE_FORMAT(create_time, '%Y-%m-%d %H:%i:%s') AS datetime_str FROM orders;
SELECT DATE_FORMAT(create_time, '%Y-%m-%d %H:00:00') AS hour_str FROM orders;
SELECT DATE_FORMAT(create_time, '%Y-%m-%d %H:%i:00') AS minute_str FROM orders;

-- GaussDB
SELECT TO_CHAR(create_time, 'YYYY-MM-DD') AS date_str FROM orders;
SELECT TO_CHAR(create_time, 'YYYY-MM-DD HH24:MI:SS') AS datetime_str FROM orders;
SELECT TO_CHAR(create_time, 'YYYY-MM-DD HH24:00:00') AS hour_str FROM orders;
SELECT TO_CHAR(create_time, 'YYYY-MM-DD HH24:MI:00') AS minute_str FROM orders;
```

**检测模式：** `DATE_FORMAT\s*\(`

---

## 3. STR_TO_DATE 转换

**规则：`STR_TO_DATE(str, format)` → `TO_DATE(str, format)` 或 `TO_TIMESTAMP(str, format)`**

- 只有日期 → `TO_DATE`
- 包含时间 → `TO_TIMESTAMP`

```sql
-- MySQL
SELECT STR_TO_DATE('2024-01-15', '%Y-%m-%d') AS date_val;
SELECT STR_TO_DATE('2024-01-15 14:30:00', '%Y-%m-%d %H:%i:%s') AS datetime_val;

-- GaussDB
SELECT TO_DATE('2024-01-15', 'YYYY-MM-DD') AS date_val;
SELECT TO_TIMESTAMP('2024-01-15 14:30:00', 'YYYY-MM-DD HH24:MI:SS') AS datetime_val;
```

**检测模式：** `STR_TO_DATE\s*\(`

---

## 4. UNIX_TIMESTAMP 转换

**规则：`UNIX_TIMESTAMP()` → `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER`**
**规则：`UNIX_TIMESTAMP(date)` → `EXTRACT(EPOCH FROM date)::INTEGER`**

```sql
-- MySQL
SELECT UNIX_TIMESTAMP() AS current_ts;
SELECT UNIX_TIMESTAMP(create_time) AS create_ts FROM orders;

-- GaussDB
SELECT EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER AS current_ts;
SELECT EXTRACT(EPOCH FROM create_time)::INTEGER AS create_ts FROM orders;
```

**检测模式：** `UNIX_TIMESTAMP\s*\(`

---

## 5. FROM_UNIXTIME 转换

**规则：`FROM_UNIXTIME(timestamp)` → `TO_TIMESTAMP(timestamp)`**

```sql
-- MySQL
SELECT FROM_UNIXTIME(1704063600) AS date_val;
SELECT FROM_UNIXTIME(unix_ts) AS created_at FROM logs;

-- GaussDB
SELECT TO_TIMESTAMP(1704063600) AS date_val;
SELECT TO_TIMESTAMP(unix_ts) AS created_at FROM logs;
```

**检测模式：** `FROM_UNIXTIME\s*\(`

---

## 6. DATE_ADD / DATE_SUB 转换

**规则：`DATE_ADD(date, INTERVAL n unit)` → `TO_CHAR(CAST(date AS TIMESTAMP) + INTERVAL 'n unit', 'YYYY-MM-DD')`**
**规则：`DATE_SUB(date, INTERVAL n unit)` → `TO_CHAR(CAST(date AS TIMESTAMP) - INTERVAL 'n unit', 'YYYY-MM-DD')`**

```sql
-- MySQL
SELECT DATE_ADD(create_time, INTERVAL 7 DAY) AS next_week FROM orders;
SELECT DATE_SUB(NOW(), INTERVAL 30 DAY) AS last_month FROM orders;
SELECT DATE_ADD(NOW(), INTERVAL 1 YEAR) AS next_year FROM orders;

-- GaussDB
SELECT TO_CHAR(CAST(create_time AS TIMESTAMP) + INTERVAL '7 DAY', 'YYYY-MM-DD') AS next_week FROM orders;
SELECT TO_CHAR(CAST(NOW() AS TIMESTAMP) - INTERVAL '30 DAY', 'YYYY-MM-DD') AS last_month FROM orders;
SELECT TO_CHAR(CAST(NOW() AS TIMESTAMP) + INTERVAL '1 YEAR', 'YYYY-MM-DD') AS next_year FROM orders;
```

**检测模式：** `DATE_ADD\s*\(`, `DATE_SUB\s*\(`

---

## 7. DATEDIFF 转换

**规则：`DATEDIFF(date1, date2)` → `(date1::date - date2::date)`**

```sql
-- MySQL
SELECT DATEDIFF(end_date, start_date) AS days_diff FROM events;
SELECT DATEDIFF(NOW(), create_time) AS days_ago FROM orders;

-- GaussDB
SELECT (end_date::date - start_date::date) AS days_diff FROM events;
SELECT (NOW()::date - create_time::date) AS days_ago FROM orders;
```

**检测模式：** `DATEDIFF\s*\(`

---

## 8. TIMESTAMPDIFF 转换

**规则：根据单位选择不同的转换方式**

```sql
-- MySQL
SELECT TIMESTAMPDIFF(DAY, start_time, end_time) AS days_diff FROM events;
SELECT TIMESTAMPDIFF(HOUR, start_time, end_time) AS hours_diff FROM events;
SELECT TIMESTAMPDIFF(MINUTE, start_time, end_time) AS minutes_diff FROM events;
SELECT TIMESTAMPDIFF(SECOND, start_time, end_time) AS seconds_diff FROM events;

-- GaussDB
SELECT EXTRACT(DAY FROM (end_time - start_time)) AS days_diff FROM events;
SELECT EXTRACT(EPOCH FROM (end_time - start_time))::INTEGER / 3600 AS hours_diff FROM events;
SELECT EXTRACT(EPOCH FROM (end_time - start_time))::INTEGER / 60 AS minutes_diff FROM events;
SELECT EXTRACT(EPOCH FROM (end_time - start_time))::INTEGER AS seconds_diff FROM events;
```

**转换规则：**
- `TIMESTAMPDIFF(DAY, ...)` → `EXTRACT(DAY FROM (...))`
- `TIMESTAMPDIFF(HOUR, ...)` → `EXTRACT(EPOCH FROM (...))::INTEGER / 3600`
- `TIMESTAMPDIFF(MINUTE, ...)` → `EXTRACT(EPOCH FROM (...))::INTEGER / 60`
- `TIMESTAMPDIFF(SECOND, ...)` → `EXTRACT(EPOCH FROM (...))::INTEGER`

**检测模式：** `TIMESTAMPDIFF\s*\(`

---

## 9. DATE / TIME 提取函数

**规则：`DATE(datetime)` → `TO_CHAR(datetime, 'YYYY-MM-DD')`**
**规则：`TIME(datetime)` → `datetime::time`**

```sql
-- MySQL
SELECT DATE(create_time) AS date_part FROM orders;
SELECT TIME(create_time) AS time_part FROM orders;

-- GaussDB
SELECT TO_CHAR(create_time, 'YYYY-MM-DD') AS date_part FROM orders;
SELECT create_time::time AS time_part FROM orders;
```

**检测模式：** `\bDATE\s*\(`, `\bTIME\s*\(`

---

## 10. YEAR / MONTH / DAY 提取函数

**规则：`YEAR(date)` → `EXTRACT(YEAR FROM date)`**

```sql
-- MySQL
SELECT YEAR(create_time) AS year FROM orders;
SELECT MONTH(create_time) AS month FROM orders;
SELECT DAY(create_time) AS day FROM orders;
SELECT HOUR(create_time) AS hour FROM orders;
SELECT MINUTE(create_time) AS minute FROM orders;
SELECT SECOND(create_time) AS second FROM orders;

-- GaussDB
SELECT EXTRACT(YEAR FROM create_time) AS year FROM orders;
SELECT EXTRACT(MONTH FROM create_time) AS month FROM orders;
SELECT EXTRACT(DAY FROM create_time) AS day FROM orders;
SELECT EXTRACT(HOUR FROM create_time) AS hour FROM orders;
SELECT EXTRACT(MINUTE FROM create_time) AS minute FROM orders;
SELECT EXTRACT(SECOND FROM create_time) AS second FROM orders;
```

**检测模式：** `\b(YEAR|MONTH|DAY|HOUR|MINUTE|SECOND)\s*\(`

---

## 快速参考表

| MySQL | GaussDB | 检测模式 |
|-------|---------|---------|
| `SYSDATE()` | `NOW()` | `SYSDATE\s*\(` |
| `CURDATE()` | `CURRENT_DATE` | `CURDATE\s*\(` |
| `CURTIME()` | `CURRENT_TIME` | `CURTIME\s*\(` |
| `DATE_FORMAT(d, fmt)` | `TO_CHAR(d, fmt)` | `DATE_FORMAT\s*\(` |
| `STR_TO_DATE(s, fmt)` | `TO_DATE/TO_TIMESTAMP(s, fmt)` | `STR_TO_DATE\s*\(` |
| `UNIX_TIMESTAMP()` | `EXTRACT(EPOCH FROM CURRENT_TIMESTAMP)::INTEGER` | `UNIX_TIMESTAMP\s*\(` |
| `FROM_UNIXTIME(ts)` | `TO_TIMESTAMP(ts)` | `FROM_UNIXTIME\s*\(` |
| `DATE_ADD(d, INTERVAL n u)` | `TO_CHAR(CAST(d AS TIMESTAMP) + INTERVAL 'n u', 'YYYY-MM-DD')` | `DATE_ADD\s*\(` |
| `DATE_SUB(d, INTERVAL n u)` | `TO_CHAR(CAST(d AS TIMESTAMP) - INTERVAL 'n u', 'YYYY-MM-DD')` | `DATE_SUB\s*\(` |
| `DATEDIFF(d1, d2)` | `(d1::date - d2::date)` | `DATEDIFF\s*\(` |
| `TIMESTAMPDIFF(DAY, s, e)` | `EXTRACT(DAY FROM (e - s))` | `TIMESTAMPDIFF\s*\(` |
| `DATE(dt)` | `TO_CHAR(dt, 'YYYY-MM-DD')` | `\bDATE\s*\(` |
| `TIME(dt)` | `dt::time` | `\bTIME\s*\(` |
| `YEAR(d)` | `EXTRACT(YEAR FROM d)` | `\bYEAR\s*\(` |

---

## 处理流程

1. **扫描 SQL 语句**
2. **依次匹配检测模式**
3. **如果匹配到，应用对应的转换规则**
4. **如果没有匹配项，保持原样**
