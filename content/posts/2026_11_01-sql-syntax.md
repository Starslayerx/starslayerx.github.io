+++
date = '2026-11-01T8:00:00+01:00'
draft = False
title = 'SQL Lexical Order and Syntax Order'
categories = ['Blog']
tags = ['SQL']
+++

这篇文章简单总结一下 SQL 的编写顺序与执行顺序

### SQL Lexical Order

一般编写 SQL 时需要遵循以下顺序：

```sql
WITH cte_name (...) AS ...
SELECT [DISTINCT] 选择列
FROM 表1
[JOIN 表2 ON 连接条件]
WHERE 行过滤条件
GROUP BY 分组列
HAVING 分组后过滤条件
[WINDOW 窗口定义]
ORDER BY 排序列
LIMII n OFFSET m;
```

1. 首先 `WITH` 定义 CTE (Common Table Expression) 公共表表达式，这为子查询取一个名字，供后面的主查询使用，可以定义多个，使用逗号分隔

例如：

```sql
WITH totals AS (
    SELECT facid, SUM(slots) AS total_slots
    FROM cd.bookings
    GROUP BY facid
)
SELECT * FROM totals
```

2. `SELECT` 选择需要输出的列、表达式、聚合结果、窗口函数，可以添加 `DISTINCT` 来过滤重复列

例如：

```sql
SELECT facid, SUM(slots) AS total_slots
```

3. `FROM` 确定数据源，后面指定主表，通过 `JOIN` 指定需要连接的表，`ON` 指定连接条件; 写在 `SELECT` 后，`WHERE` 前

例如：

```sql
FROM cd.bookings bks
JOIN cd.facilities facs ON bks.facid = facs.facid
```

4. `WHERE` 对 `FROM/JOIN` 的结果按行过滤，常用条件有：
   `=`、`<>`、`>`、`<`、`BETWEEN`、`IN`、`LIKE`、`IS NULL`、`AND`、`OR`、`NOT` 等

5. `GROUP BY` 把行按指定的列进行分组，每组变成一行，通常配合聚合函数使用，可以分多组

例如：

```sql
GROUP BY col1, col2
```

6. `HAVING` 对 `GROUP BY` 产生的分组/聚合结果进行过滤

例如：

```sql
HAVING sum(slots) > 1000
```

7. `WINDOW` 给窗口函数定义命名窗口，避免在多个窗口函数重复写 `OVER`

例如：

```sql
SELECT facid, SUM(slots) OVER (PARTITION BY facid) AS s
FROM cd.bookings
WINDOW w AS (PARTITION BY facid)
```

8. `ORDER BY` 对结果进行排序，默认升序 `ASC`，也可以指定降序 `DESC`

例如：

```sql
ORDER BY total_slots DESC
```

9. `LIMIT / OFFSET / FETCH` 限制最终返回的行数，或跳过若干行

例如：

```sql
LIMIT 10 OFFSET 20;
-- 或标准 SQL：
OFFSET 20 ROWS FETCH FIRST 10 ROWS ONLY;
```

上面讲的是 SQL 编写的顺序，但实际执行的逻辑顺序并非如此，也不易于理解 SQL，下面就将一下 SQL 的逻辑执行顺序

### SQL Syntax Order

SQL 执行顺序大致如下：

```
FROM / JOIN / ON
→ WHERE
→ GROUP BY
→ HAVING
→ SELECT [DISTINCT]
→ ORDER BY
→ LIMIT / OFFSET / FETCH
```

1. 首先通过 `FROM [JOIN ... ON ...]` 确定数据来源
2. 然后通过 `WHERE` 对数据按行进行过滤
3. 之后可以使用 `GROUP BY` 对数据进行分组、聚合
4. 得到分组结果，可以使用 `HAVING` 对分组或聚合结果再进行过滤
5. 此时使用 `SELECT` 选择需要展示的数据，可以添加 `DISTINCT` 过滤重复行
6. 再然后可通过 `ORDER BY` 对数据进行排序
7. 最后限制数据的返回行数 `LIMIT / OFFSET / FETCH`

这里给一个完整示例：

```sql
WITH totals AS (                          -- 1. WITH：定义 CTE
    SELECT facid, SUM(slots) AS total_slots
    FROM cd.bookings
    GROUP BY facid
)
SELECT facid, total_slots                 -- 2. SELECT：选择输出列
FROM totals                               -- 3. FROM：数据源是 CTE totals
WHERE total_slots > 100                   -- 4. WHERE：过滤行
GROUP BY facid                            -- 5. GROUP BY：分组
HAVING SUM(total_slots) > 200             -- 6. HAVING：过滤组
ORDER BY total_slots DESC                 -- 8. ORDER BY：排序
LIMIT 10;                                 -- 9. LIMIT：限制返回行数
```

编写 SQL 时，要按照语法写，而分析问题时，则按照语义思考，这样 SQL 便容易理解多了
