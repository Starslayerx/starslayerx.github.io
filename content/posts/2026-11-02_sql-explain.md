+++
date = '2026-11-02T8:00:00+01:00'
draft = false
title = 'Using EXPLAIN'
categories = ['Blog']
tags = ['SQL', 'Postgres']
+++

[Source](https://www.postgresql.org/docs/current/using-explain.html#USING-EXPLAIN-ANALYZE)

Postgres 为收到的每条查询制定查询计划 (query plan)，要获得良好的性能，选择和查询结构 (query structure) 和数据特征 (properties of the data) 想匹配的正确计划至关重要，因此系统内置了一个复杂的规划器 planner。
可以使用 EXPLAIN 命令查看规划器生成的查询计划，本篇文章介绍相关知识。

### EXPLAIN Basics

查询计划的结构是一个查询节点的树 (tree of plan nodes)。
树的底层节点是扫描节点 (scan nodes)，它们从表中返回原始行。
对于不同的表访问方式，有不同的扫描节点类型：顺序扫描，索引扫描和 bitmap 位图索引扫描。
也有非表行来源，例如 `VALUES` 子句和 `FROM` 中的集合返回函数，他们各自有对应的扫描节点类型。
如果查询有 joining 连接，aggregation 聚合， sorting 排序 或其他操作，则扫描节点上面会有额外的节点来执行这些操作。
通常会有不止一个方法来执行这些操作，因此这里会出现不同类型的节点类型。

`EXPLAIN` 的输出会为计划树中的每个节点各显示一行，展示该节点的基本类型，以及规划器对该节点执行时所作的代价估算。
可能会出现一些附加行，它们相对于节点的摘要进行缩进，用来显示该节点的其他属性。
第一行是估计的计划总执行代价 (estimated total execution cost); 这是规划器尝试减小的数字。

例如：

```sql
EXPLAIN SELECT * FROM tenk1;

                         QUERY PLAN
-------------------------------------------------------------
 Seq Scan on tenk1  (cost=0.00..445.00 rows=10000 width=244)
```

由于该查询没有 `WHERE` 语句，因此规划器选择了单次的顺序扫描计划。

括号中的数字意思是：

- 估计的启动成本。这是输出阶段开始前，消耗的时间，例如排序节点中执行排序所需的时间
- 估计的总成本。该值是基于计划节点会执行完毕这一假设给出的。实际执行中，节点的父节点可能提前停止，不会读取全部可用行
- 预计的输出行数。同样该节点会假设执行完毕
- 预计的行输出平均宽度 (bytes)

成本以任意单位计量，具体单位由规划器的成本参数确定。

需要理解的很重要的一点是，上层节点的成本，包含其所有子节点的成本。
此外，成本只反映规划器关心的关心的部分。
特别是，不会考虑将结果转换为文本输出的成本或将其传递给客户端的成本，这点是实际消耗时间的重要影响因素；优化器忽略这些成本，是因为这无法通过修改计划改变。

行数 rows 有点棘手，因为它并不是处理或扫描的行数，而是该节点输出的行数。
由于节点上会应用 `WHERE` 子句条件进行过滤，这个值通常小于扫描行数。
理想情况下，顶层的 rows 估算值会按接近查询实际返回、更新或删除的行数。

例如之前的例子：

```sql
EXPLAIN SELECT * FROM tenk1;

                         QUERY PLAN
-------------------------------------------------------------
 Seq Scan on tenk1  (cost=0.00..445.00 rows=10000 width=244)
```

这些数字的推导过程非常直接，如果这样：

```sql
SELECT relpages, reltuples FROM pg_class WHERE relname = 'tenk1';
```

如果 `tenk1` 有 345 磁盘页 (disk pages) 和 10000 行。
则估计成本由 `(disk pages read * seq_page_cost) + (rows scanned * cpu_tuple_cost)` 计算得到。
默认，`seq_page_cost` 是 `1.0` 并 `cpu_tuple_cost` 是 `0.01`，因此估计的成本是 `345*1.0 + 10000 * 0.01 = 445`。

现在修改查询，并添加一个 `WHERE` 条件：

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 7000;

                         QUERY PLAN
------------------------------------------------------------
 Seq Scan on tenk1  (cost=0.00..470.00 rows=7000 width=244)
   Filter: (unique1 < 7000)
```

注意到 `EXPLAIN` 展示了 `WHERE` 子句被附加在 Seq Scan 计划节点上的 "Filter" 条件来引用。
这意味着计划节点检查扫描的每一行，并仅输出通过条件的行。
预计的输出行由于 `WHERE` 被减少，然而，扫描时仍然会访问所有 10000 行，因此成本并未因此下降。
实际上，它略微增加了一点(10000 \* cpu_operator_cost)，以反映检查 `WHERE` 条件时额外消耗的 CPU 时间。

改查询实际的行数是 7000，但行估计是预估。
如果重复该实验，可能得到一个有些不同的预估结果；
此外，每次执行 `ANALYZE` 命令后可能会有一些变化，因为 `ANALYZE` 产生的统计信息取自表的随机抽样样本。

现在，让条件更严格一些：

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100;

                                  QUERY PLAN
------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=5.06..224.98 rows=100 width=244)
   Recheck Cond: (unique1 < 100)
   ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=100 width=0)
         Index Cond: (unique1 < 100)
```

这里优化器决定使用一个两步计划：
子计划节点访问索引找到符合索引条件行的位置，然后上层计划节点通过表本身实际读取这些行。
单独读取每行比顺序读取昂贵的多，但是由于并不是所有表中的页都会被访问，因此它仍然比顺序扫描更加便宜。
（使用勇敢计划层级的原因是，上层计划节点会在读取索引所标识的位置之前，先将其按物理顺序排序，以尽量减少单独抓取的成本。节点中位图 Bitmap 就是执行排序的机制。）

下面再为 `WHERE` 语句添加一个条件：

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND stringu1 = 'xxx';

                                  QUERY PLAN
------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=5.04..225.20 rows=1 width=244)
   Recheck Cond: (unique1 < 100)
   Filter: (stringu1 = 'xxx'::name)
   ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=100 width=0)
         Index Cond: (unique1 < 100)
```

添加的条件 `stringul = 'xxx'` 降低了输出的预期行数，但没有降低成本，因为我们仍然需要访问相同的行。
这是由于 `stringul` 语句不能被应用于索引条件，索引只在 `unique1` 列上。
相反，它被作为筛选条件应用于通过检索索引到的行上的。
因此，实际成本略有上升，以反映这项额外的检查。

在一些情况下，优化器倾向于更简单的索引扫描计划：

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 = 42;

                                 QUERY PLAN
-----------------------------------------------------------------------------
 Index Scan using tenk1_unique1 on tenk1  (cost=0.29..8.30 rows=1 width=244)
   Index Cond: (unique1 = 42)
```

在这种类型计划中，表行按索引顺序读取，这会进一步增加读取成本；
但由于行数太少，为行位置排序所付出的额外开销并不划算。
通常会看到单行的查询产生这种计划。
也通常在 `ORDER BY` 条件符合索引顺序的查询中看到，因为无需额外的排序步骤。
在这个例子中，添加 `ORDER BY unique1` 会使用同样的计划，因为索引已经提供了所请求的顺序。

优化器可以通过多种方式实现 `ORDER BY` 语句。
上面例子表明，这种排序子句可隐式实现。
优化器也可以添加一个显示的 Sort 步骤。

```sql
EXPLAIN SELECT * FROM tenk1 ORDER BY unique1;

                            QUERY PLAN
-------------------------------------------------------------------
 Sort  (cost=1109.39..1134.39 rows=10000 width=244)
   Sort Key: unique1
   ->  Seq Scan on tenk1  (cost=0.00..445.00 rows=10000 width=244)
```

如果计划中的某一部分能够保证按所需排序键的前缀有序，那么规划器可能会转而决定使用“增量排序” (Incremental Sort) 步骤：

```sql
EXPLAIN SELECT * FROM tenk1 ORDER BY hundred, ten LIMIT 100;

                                              QUERY PLAN
------------------------------------------------------------------------------------------------
 Limit  (cost=19.35..39.49 rows=100 width=244)
   ->  Incremental Sort  (cost=19.35..2033.39 rows=10000 width=244)
         Sort Key: hundred, ten
         Presorted Key: hundred
         ->  Index Scan using tenk1_hundred on tenk1  (cost=0.29..1574.20 rows=10000 width=244)
```

对比其他排序，增量排序可以在整个结果集完成排序之前就返回元组，这尤其有利于 `LIMIT` 查询进行优化。
这同样会降低内存使用量，并减少排序益处到磁盘的可能性。
但代价是：将结果集拆分成多个排序批次会带来更高的额外开销。

如果 `WHERE` 中引用的多个列上分别建有独立索引，规划期可能会选择将这些索引以 `AND` 或 `OR` 组合使用：

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 900;

                                     QUERY PLAN
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=25.07..60.11 rows=10 width=244)
   Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))
   ->  BitmapAnd  (cost=25.07..25.07 rows=10 width=0)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=100 width=0)
               Index Cond: (unique1 < 100)
         ->  Bitmap Index Scan on tenk1_unique2  (cost=0.00..19.78 rows=999 width=0)
               Index Cond: (unique2 > 9000)
```

但是这个查询要求访问两个索引，因此对比使用一个索引并将另一个作为过滤条件，不一定更好。
如果调整涉及的范围，就会看到执行计划也随之改变。

```sql
EXPLAIN SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000 LIMIT 2;

                                     QUERY PLAN
-------------------------------------------------------------------------------------
 Limit  (cost=0.29..14.28 rows=2 width=244)
   ->  Index Scan using tenk1_unique2 on tenk1  (cost=0.29..70.27 rows=10 width=244)
         Index Cond: (unique2 > 9000)
         Filter: (unique1 < 100)

```

这和上面的查询一样，但是添加了 `LIMIT` 因此并不需要遍历所有行了，优化器故修改了执行计划。
注意到 Index Scan 节点的总成本和行数，显示得好像它已经执行完毕一样。
不过，Limit 节点预计只取到这些行的五分之一就会停止，所以它的总成本也只有五分之一，而这才是实际查询的估算成本。
之所以选择这个方案，而不是在之前方案上增加一个 Limit 节点，是因为 Limit 无法避免位图扫描的启动开销，因此采用那种方式的总成本会略高于 25 个单位。

现在尝试用刚才讨论的这些列来连接两张表：

```sql
EXPLAIN SELECTT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;

                                      QUERY PLAN
--------------------------------------------------------------------------------------
 Nested Loop  (cost=4.65..118.50 rows=10 width=488)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.38 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..7.90 rows=1 width=244)
         Index Cond: (unique2 = t1.unique2)
```

在这个计划中，有一个嵌套循环连接节点 nested-loop join node，以及两个表的扫描作为输入（子节点）。
节点的总结行缩进反应了树结构，join 的第一个获外层 (outer)，子节点是一个位图扫描 Bitmap Heap Scan，类似之前看到的。
其成本和 `SELECT ... WHERE unique1 < 10` 一样，因为在这个节点中我们要执行 `WHERE unique1 < 10` 语句。
`t1.unique1 = t2.unique2` 语句此时还未涉及到，因此不会影响扫描的行数。
nested-loop join node 会对外层子节点的每行都运行第二个或内 (inner) 子节点，外层行的列值可以被插入内层扫描；
这里来自外层行的 `t1.unique2` 值是可获得的，因此计划成本类似前面看到的 `SELECT ... WHERE t2.unique2 = constant` 语句。
（预估的成本实际上比之前的低一点，因为预计在对 t2 进行重复扫描期间会发生缓存。）
循环节点的成本就会被设置成：outer 外层节点的成本，加上重复次数乘以内层扫描 (`10 * 7.90`)，加上处理 join 的 CPU 时间。`(outer 代价 + outer 行数 × inner 单次代价 + 连接 CPU)`

在这个例子中，join 的输出行数和两个扫描各自行数的乘积相同；
但并非所有情况都是如此，因为可能还有额外的 `WHERE` 子句同时涉及这两张表，因而只能在连接点应用，而不能应用到任意输入扫描上。
例如：

```sql
SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 10 AND t2.unique2 < 10 AND t1.hundred < t2.hundred;

                                         QUERY PLAN
---------------------------------------------------------------------------------------------
 Nested Loop  (cost=4.65..49.36 rows=33 width=488)
   Join Filter: (t1.hundred < t2.hundred)
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.38 rows=10 width=244)
         Recheck Cond: (unique1 < 10)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0)
               Index Cond: (unique1 < 10)
   ->  Materialize  (cost=0.29..8.51 rows=10 width=244)
         ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..8.46 rows=10 width=244)
               Index Cond: (unique2 < 10)

```

条件 `t1.hundred < t2.hundred` 无法在 `tenk2_unique2` 索引测试，因此它在连接节点处应用。
这降低了连接节点的估计输出行数，但不能改变任一输入扫描。

注意这里规划器选择 “物化 (materralize)” 连接的内关系，使在它上面放置一个 `Materialize` 计划节点。
这意味着 t2 索引扫描只执行一次，即使嵌套循环连接节点需要读取该数据十次，即对外关系的每一行读取一次。
`Materialize` 节点在读取数据时将其保存在内存中，然后在后续每次传递时从内层返回数据。

当处理外层连接时，可能会看到 join plan nodes 同时附加 "Join Filter" 和 "Filter" 条件。
Join Filter 条件来自外层连接点 `ON` 语句，如果一行不满足 Join Filter 条件仍然可能作为空扩展行发出。
但普通 Filter 条件是在外连接规则之后应用到，因此会无条件地移除行。
在内连接中，这些类型的过滤器在语义上没有区别。

如果稍微修改一下查询的选择性，可能会得到一个非常不同的查询连接计划：

```sql
EXPLAIN SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;

                                        QUERY PLAN
------------------------------------------------------------------------------------------
 Hash Join  (cost=226.23..709.73 rows=100 width=488)
   Hash Cond: (t2.unique2 = t1.unique2)
   ->  Seq Scan on tenk2 t2  (cost=0.00..445.00 rows=10000 width=244)
   ->  Hash  (cost=224.98..224.98 rows=100 width=244)
         ->  Bitmap Heap Scan on tenk1 t1  (cost=5.06..224.98 rows=100 width=244)
               Recheck Cond: (unique1 < 100)
               ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=100 width=0)
                     Index Cond: (unique1 < 100)

```

这里规划器选择使用哈希连接 (hash join)，其中一个表的行被放入内存中的哈希表，之后扫描另一个表，并针对每一行在哈希中探测匹配项。
再次注意缩进如何反应计划结构：tenk1 上的位图扫描是哈希节点 (hash node) 的输入，哈希节点构建哈希表。
然后该结果返回给 Hash Join 节点从其外部自计划读取行，并为每一行搜索哈希表。

另一种可能的类型是 merge join，例如：

```sql
EXPLAIN SELECT *
FROM tenk1 t1, onek t2
WHER t1.unique1 < 100 AND t1.unique2 = t2.unique2;

                                        QUERY PLAN
------------------------------------------------------------------------------------------
 Merge Join  (cost=0.56..233.49 rows=10 width=488)
   Merge Cond: (t1.unique2 = t2.unique2)
   ->  Index Scan using tenk1_unique2 on tenk1 t1  (cost=0.29..643.28 rows=100 width=244)
         Filter: (unique1 < 100)
   ->  Index Scan using onek_unique2 on onek t2  (cost=0.28..166.28 rows=1000 width=244)
```

Merge join 要求输入数据根据连接键 join keys 排序。
在这个例子中，每个输入通过 index scan 排序，从而访问正确的顺序；
但是顺序扫描和排序也能被使用。
（顺序扫描和排序通常在排序很多行的时候比索引扫描好，因为所以扫描需要非顺序的磁盘访问）

查看其他计划的一种方法是，强制规划器忽略它认为最便宜的方案。
例如，我们不相信合并连接时当前示例的最佳连接类型，可以尝试：

```sql
SET enable_mergjoin=off;

EXPLAIN SELECT *
FROM tenk1 t1, onek t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2;

                                        QUERY PLAN
------------------------------------------------------------------------------------------
 Hash Join  (cost=226.23..344.08 rows=10 width=488)
   Hash Cond: (t2.unique2 = t1.unique2)
   ->  Seq Scan on onek t2  (cost=0.00..114.00 rows=1000 width=244)
   ->  Hash  (cost=224.98..224.98 rows=100 width=244)
         ->  Bitmap Heap Scan on tenk1 t1  (cost=5.06..224.98 rows=100 width=244)
               Recheck Cond: (unique1 < 100)
               ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=100 width=0)
                     Index Cond: (unique1 < 100)
```

这说明规划器认为 hash join 比 merge join 几乎贵了 50%。
当然，下一个问题就是这个判断是否正确。
可以使用 `EXPLAIN ANALYZE` 分析，下面会介绍。

当通过 enable/disable flags 标签来关闭计划节点类型时，许多标志指示劝阻使用相应的计划节点，而不会完全禁止规划器使用该计划节点类型的能力。
这是有意设计的，一边规划器仍然能够给出查询形成计划。
让结果包含被禁用的节点时，`EXPLAIN` 输出会明确指出。

```sql
SET enable_seqscan = off;
EXPLAIN SELECT * FROM unit;

                       QUERY PLAN
---------------------------------------------------------
 Seq Scan on unit  (cost=0.00..21.30 rows=1130 width=44)
   Disabled: true
```

由于 `unit` 表没有索引，因此没有其他方式来读取表中数据，故规划器只能使用顺序扫描。

一些查询计划涉及子计划 _subplans_，这些子计划来源于原始查询中的子 `SELECT`。
这类查询优势可以转换为普通连接计划，但当无法转换时，会得到这样的计划：

```sql
EXPLAIN VERBOSE SELECT unique1
FROM tenk1 t
WHERE t.ten < ALL (SELECT o.ten FROM onek o WHERE o.four = t.four);

                               QUERY PLAN
-------------------------------------------------------------------------
 Seq Scan on public.tenk1 t  (cost=0.00..586095.00 rows=5000 width=4)
   Output: t.unique1
   Filter: (ALL (t.ten < (SubPlan 1).col1))
   SubPlan 1
     ->  Seq Scan on public.onek o  (cost=0.00..116.50 rows=250 width=4)
           Output: o.ten
           Filter: (o.four = t.four)
```

这个相当人为的示例用于说明几点：
来自外部计划层级的值可以传递到子计划中，并且子查询的结果可供外部计划使用。（这里 `t.four` 向下传递）
这些结果通过 `EXPLAIN` 带注释，以类似 `(subplan_name).colN` 的记号显示，它指的是子 `SELECT` 的第 N 个输出列。

在上面示例中，`ALL` 运算符会为外部查询的每一次再次运行子计划，有些查询可以使用哈希子计划来避免这种情况：

```sql
EXPLAIN SELECT *
FROM tenk1 t
WHERE t.unique1 NOT IN (SELECT o.unique1 FROM onek o);

                                         QUERY PLAN
--------------------------------------------------------------------------------------------
 Seq Scan on tenk1 t  (cost=61.77..531.77 rows=5000 width=244)
   Filter: (NOT (ANY (unique1 = (hashed SubPlan 1).col1)))
   SubPlan 1
     ->  Index Only Scan using onek_unique1 on onek o  (cost=0.28..59.27 rows=1000 width=4)
(4 rows)
```

这里，子计划只运行一次，其输出被加载到内存哈希表中，然后由外部的 `ANY` 运算符探测。
这要求子 `SELECT` 不引用外部查询的任何变量，并且 `ANY` 的比较运算符适合哈希处理。

如果除了不引用外部查询的任何变量之外，子 `SELECT` 还不能返回多于一行，那么它可能改为实现为初始化计划：

```sql
EXPLAIN VERBOSE SELECT unique1
FROM tenk1 t1 WHERE t1.ten = (SELECT (random() * 10)::integer);

                             QUERY PLAN
--------------------------------------------------------------------
 Seq Scan on public.tenk1 t1  (cost=0.02..470.02 rows=1000 width=4)
   Output: t1.unique1
   Filter: (t1.ten = (InitPlan 1).col1)
   InitPlan 1
     ->  Result  (cost=0.00..0.02 rows=1 width=4)
           Output: ((random() * '10'::double precision))::integer
```

初始化计划在外部计划的每次执行中只运行一次，并且其结果会被保存下来，以便在外部计划的后续行中重复使用。
因此在这个示例中，`random()` 只求值一次，并且 `t1.ten` 的所有值都与同一个随机选择的整数进行比较。
这与没有子 `SELECT` 结构时会发生的情况大不相同。

### EXPLAIN ANALYZE

可以使用 `EXPLAIN ANALYZE` 来检查规划器的准确度，该参数会真实执行查询，并展示实际的行数和每个计划节点的运行时间。

例如这样：

```sql
SELECT ANALYZE SELECT *
FROM tenk1 t2, tenk2 t2
WHERE t1.unique1 < 10 AND t1.unique2 = t2.unique2;

                                                           QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=4.65..118.50 rows=10 width=488) (actual time=0.017..0.051 rows=10.00 loops=1)
   Buffers: shared hit=36 read=6
   ->  Bitmap Heap Scan on tenk1 t1  (cost=4.36..39.38 rows=10 width=244) (actual time=0.009..0.017 rows=10.00 loops=1)
         Recheck Cond: (unique1 < 10)
         Heap Blocks: exact=10
         Buffers: shared hit=3 read=5 written=4
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..4.36 rows=10 width=0) (actual time=0.004..0.004 rows=10.00 loops=1)
               Index Cond: (unique1 < 10)
               Index Searches: 1
               Buffers: shared hit=2
   ->  Index Scan using tenk2_unique2 on tenk2 t2  (cost=0.29..7.90 rows=1 width=244) (actual time=0.003..0.003 rows=1.00 loops=10)
         Index Cond: (unique2 = t1.unique2)
         Index Searches: 10
         Buffers: shared hit=24 read=6
 Planning:
   Buffers: shared hit=15 dirtied=9
 Planning Time: 0.485 ms
 Execution Time: 0.073 ms
```

注意实际时间以毫秒为单位，而成本估计是任意单元表示的；因此他们并不想匹配。
最重要的事情是看，预估行数和实际结果是否一致。
在这个例子中，各项都分毫不差，在实际操作中，这种情况相当少见。

在一些查询计划中，子查询计划节点可能执行不止一次。
例如，上述嵌套循环计划中，内存索引扫描针对外层的每一行执行一次。
这样做是为了让这些数字与成本估算的显示方式具有可比性。
将结果乘以 loops 值，即可得到该节点实际消耗的总时间。
在上面的示例中，tenk2 上执行索引扫描总共花费了 0.030 毫秒。

在一些情况下 `EXPLAIN ANALYZE` 展示计划节点执行次数和行数额外的统计信息。
例如，排序 Sort 和 Hash 节点提供了额外的信息：

```sql
EXPLAIN ANALYZE SELECT *
FROM tenk1 t1, tenk2 t2
WHERE t1.unique1 < 100 AND t1.unique2 = t2.unique2 ORDER BY t1.fivethous;

                                                                 QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=713.05..713.30 rows=100 width=488) (actual time=2.995..3.002 rows=100.00 loops=1)
   Sort Key: t1.fivethous
   Sort Method: quicksort  Memory: 74kB
   Buffers: shared hit=440
   ->  Hash Join  (cost=226.23..709.73 rows=100 width=488) (actual time=0.515..2.920 rows=100.00 loops=1)
         Hash Cond: (t2.unique2 = t1.unique2)
         Buffers: shared hit=437
         ->  Seq Scan on tenk2 t2  (cost=0.00..445.00 rows=10000 width=244) (actual time=0.026..1.790 rows=10000.00 loops=1)
               Buffers: shared hit=345
         ->  Hash  (cost=224.98..224.98 rows=100 width=244) (actual time=0.476..0.477 rows=100.00 loops=1)
               Buckets: 1024  Batches: 1  Memory Usage: 35kB
               Buffers: shared hit=92
               ->  Bitmap Heap Scan on tenk1 t1  (cost=5.06..224.98 rows=100 width=244) (actual time=0.030..0.450 rows=100.00 loops=1)
                     Recheck Cond: (unique1 < 100)
                     Heap Blocks: exact=90
                     Buffers: shared hit=92
                     ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=100 width=0) (actual time=0.013..0.013 rows=100.00 loops=1)
                           Index Cond: (unique1 < 100)
                           Index Searches: 1
                           Buffers: shared hit=2
 Planning:
   Buffers: shared hit=12
 Planning Time: 0.187 ms
 Execution Time: 3.036 ms
```

Sort 节点会显示所使用的排序方法（特别是排序是在内存中还是在磁盘上进行的），以及所需的内存或磁盘空间量。
Hash 节点会显示哈希桶数和批次数，以及哈希表使用的峰值内存量。
（如果批次数超过 1，还会涉及磁盘空间的使用，但此处未显示。）

Index Scan 节点（以及 Bitmap Index Scan 和 Index-Only Scan 节点）会显示一行“Index Searches”，报告所有节点执行/循环中的搜索总次数：

```sql
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE thousand IN (1, 500, 700, 999);
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=9.45..73.44 rows=40 width=244) (actual time=0.012..0.028 rows=40.00 loops=1)
   Recheck Cond: (thousand = ANY ('{1,500,700,999}'::integer[]))
   Heap Blocks: exact=39
   Buffers: shared hit=47
   ->  Bitmap Index Scan on tenk1_thous_tenthous  (cost=0.00..9.44 rows=40 width=0) (actual time=0.009..0.009 rows=40.00 loops=1)
         Index Cond: (thousand = ANY ('{1,500,700,999}'::integer[]))
         Index Searches: 4
         Buffers: shared hit=8
 Planning Time: 0.029 ms
 Execution Time: 0.034 ms
```

这里我们看到一个 Bitmap Index Scan 节点，它需要进行 4 次独立的索引查找。
该扫描必须针对谓词 IN 结构中的每个整数值，从 `tenk1_thous_tenthous` 索引根页面开始各搜索一次索引。
然而，索引查找的次数通常不会与查询谓词存在如此简单的对应关系：

```sql
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE thousand IN (1, 2, 3, 4);
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=9.45..73.44 rows=40 width=244) (actual time=0.009..0.019 rows=40.00 loops=1)
   Recheck Cond: (thousand = ANY ('{1,2,3,4}'::integer[]))
   Heap Blocks: exact=38
   Buffers: shared hit=40
   ->  Bitmap Index Scan on tenk1_thous_tenthous  (cost=0.00..9.44 rows=40 width=0) (actual time=0.005..0.005 rows=40.00 loops=1)
         Index Cond: (thousand = ANY ('{1,2,3,4}'::integer[]))
         Index Searches: 1
         Buffers: shared hit=2
 Planning Time: 0.029 ms
 Execution Time: 0.026 ms
```

我们这个 IN 查询的变体只执行了 1 次索引搜索。
与原始查询相比，它花在遍历索引上的时间更少，因为它的 IN 构造所使用的值所匹配的索引元组彼此相邻，都存储在同一个 `tenk1_thous_tenthous` 索引叶子页上。

“Index Searches” 这一行对于应用 skip scan 优化来更高效地遍历索引的 B-tree 索引扫描也同样有用：

```sql
EXPLAIN ANALYZE SELECT four, unique1 FROM tenk1 WHERE four BETWEEN 1 AND 3 AND unique1 = 42;
                                                              QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using tenk1_four_unique1_idx on tenk1  (cost=0.29..6.90 rows=1 width=8) (actual time=0.006..0.007 rows=1.00 loops=1)
   Index Cond: ((four >= 1) AND (four <= 3) AND (unique1 = 42))
   Heap Fetches: 0
   Index Searches: 3
   Buffers: shared hit=7
 Planning Time: 0.029 ms
 Execution Time: 0.012 ms
```

这里我们看到一个 `Index-Only Scan` 节点使用 `tenk1_four_unique1_idx`，这是 `tenk1` 表的 `four` 和 `unique1` 列上的多列索引。该扫描执行 3 次搜索，每次读取一个索引叶子页面：“four = 1 AND unique1 = 42”、“four = 2 AND unique1 = 42”和“four = 3 AND unique1 = 42”。该索引通常是跳过扫描的良好目标，因为如第 11.3 节所述，其前导列（`four` 列）只包含 4 个不同值，而第二列/最后一列（`unique1` 列）包含许多不同值。

另一种额外信息是被过滤条件移除的行数：

```sql
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE ten < 7;

                                               QUERY PLAN
-------------------------------------------------------------------​--------------------------------------
 Seq Scan on tenk1  (cost=0.00..470.00 rows=7000 width=244) (actual time=0.030..1.995 rows=7000.00 loops=1)
   Filter: (ten < 7)
   Rows Removed by Filter: 3000
   Buffers: shared hit=345
 Planning Time: 0.102 ms
 Execution Time: 2.145 ms
```

这些计数对于在连接节点处应用的过滤条件尤其有价值。“Rows Removed”行仅在至少一行被扫描的行（对于连接节点，则是潜在连接对）被过滤条件拒绝时出现。

与过滤条件类似的情况出现在“有损”索引扫描中。例如，考虑这个搜索包含特定点的多边形：

```sql
EXPLAIN ANALYZE SELECT * FROM polygon_tbl WHERE f1 @> polygon '(0.5,2.0)';

                                              QUERY PLAN
-------------------------------------------------------------------​-----------------------------------
 Seq Scan on polygon_tbl  (cost=0.00..1.09 rows=1 width=85) (actual time=0.023..0.023 rows=0.00 loops=1)
   Filter: (f1 @> '((0.5,2))'::polygon)
   Rows Removed by Filter: 7
   Buffers: shared hit=1
 Planning Time: 0.039 ms
 Execution Time: 0.033 ms
```

规划器认为（非常正确）这个示例表太小，不值得使用索引扫描，因此我们有一个普通的顺序扫描，其中所有行都被过滤条件拒绝。但如果我们强制使用索引扫描，会看到：

```sql
SET enable_seqscan TO off;

EXPLAIN ANALYZE SELECT * FROM polygon_tbl WHERE f1 @> polygon '(0.5,2.0)';

                                                        QUERY PLAN
-------------------------------------------------------------------​-------------------------------------------------------
 Index Scan using gpolygonind on polygon_tbl  (cost=0.13..8.15 rows=1 width=85) (actual time=0.074..0.074 rows=0.00 loops=1)
   Index Cond: (f1 @> '((0.5,2))'::polygon)
   Rows Removed by Index Recheck: 1
   Index Searches: 1
   Buffers: shared hit=1
 Planning Time: 0.039 ms
 Execution Time: 0.098 ms
```

这里我们可以看到，索引返回了一行候选行，随后该行被索引条件的重新检查拒绝。发生这种情况是因为 GiST 索引对于多边形包含测试是“有损”的：它实际上返回与目标重叠的多边形所在的行，然后我们必须对这些行执行精确的包含测试。

`EXPLAIN` 有一个 `BUFFERS` 选项，它提供有关在给定查询的规划和执行期间执行的 I/O 操作的额外细节。显示的缓冲区数字表示给定节点及其所有子节点命中的、读取的、弄脏的和写入的非重复缓冲区数量。`ANALYZE` 选项隐式启用 `BUFFERS` 选项。如果不希望这样，可以显式禁用 `BUFFERS`：

```sql
EXPLAIN (ANALYZE, BUFFERS OFF) SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000;

                                                           QUERY PLAN
-------------------------------------------------------------------​--------------------------------------------------------------
 Bitmap Heap Scan on tenk1  (cost=25.07..60.11 rows=10 width=244) (actual time=0.105..0.114 rows=10.00 loops=1)
   Recheck Cond: ((unique1 < 100) AND (unique2 > 9000))
   Heap Blocks: exact=10
   ->  BitmapAnd  (cost=25.07..25.07 rows=10 width=0) (actual time=0.100..0.101 rows=0.00 loops=1)
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=100 width=0) (actual time=0.027..0.027 rows=100.00 loops=1)
               Index Cond: (unique1 < 100)
               Index Searches: 1
         ->  Bitmap Index Scan on tenk1_unique2  (cost=0.00..19.78 rows=999 width=0) (actual time=0.070..0.070 rows=999.00 loops=1)
               Index Cond: (unique2 > 9000)
               Index Searches: 1
 Planning Time: 0.162 ms
 Execution Time: 0.143 ms
```

请记住，由于 `EXPLAIN ANALYZE` 实际运行查询，任何副作用都会照常发生，即使查询可能输出的任何结果都会被丢弃，以改为打印 `EXPLAIN` 数据。如果你想分析一个修改数据的查询而不改变表，可以在之后回滚命令，例如：

```sql
BEGIN;

EXPLAIN ANALYZE UPDATE tenk1 SET hundred = hundred + 1 WHERE unique1 < 100;

                                                           QUERY PLAN
-------------------------------------------------------------------​-------------------------------------------------------------
 Update on tenk1  (cost=5.06..225.23 rows=0 width=0) (actual time=1.634..1.635 rows=0.00 loops=1)
   ->  Bitmap Heap Scan on tenk1  (cost=5.06..225.23 rows=100 width=10) (actual time=0.065..0.141 rows=100.00 loops=1)
         Recheck Cond: (unique1 < 100)
         Heap Blocks: exact=90
         Buffers: shared hit=4 read=2
         ->  Bitmap Index Scan on tenk1_unique1  (cost=0.00..5.04 rows=100 width=0) (actual time=0.031..0.031 rows=100.00 loops=1)
               Index Cond: (unique1 < 100)
               Index Searches: 1
               Buffers: shared read=2
 Planning Time: 0.151 ms
 Execution Time: 1.856 ms

ROLLBACK;
```

如这个示例所示，当查询是 `INSERT`、`UPDATE`、`DELETE` 或 `MERGE` 命令时，应用表更改的实际工作由顶层的 `Insert`、`Update`、`Delete` 或 `Merge` 计划节点完成。此节点下方的计划节点执行定位旧行和/或计算新数据的工作。因此，上面我们看到已经见过的同类位图表扫描，其输出被送入一个 `Update` 节点，该节点存储更新后的行。值得注意的是，尽管修改数据的节点可能占用相当多的运行时间（这里它占用了大部分时间），规划器目前不会为此工作向代价估算中添加任何内容。这是因为对于每个正确的查询计划，要完成的工作都是相同的，因此它不影响规划决策。

当 `UPDATE`、`DELETE` 或 `MERGE` 命令影响分区表或继承层次结构时，输出可能如下所示：

```sql
EXPLAIN UPDATE gtest_parent SET f1 = CURRENT_DATE WHERE f2 = 101;

                                       QUERY PLAN
----------------------------------------------------------------------------------------
 Update on gtest_parent  (cost=0.00..3.06 rows=0 width=0)
   Update on gtest_child gtest_parent_1
   Update on gtest_child2 gtest_parent_2
   Update on gtest_child3 gtest_parent_3
   ->  Append  (cost=0.00..3.06 rows=3 width=14)
         ->  Seq Scan on gtest_child gtest_parent_1  (cost=0.00..1.01 rows=1 width=14)
               Filter: (f2 = 101)
         ->  Seq Scan on gtest_child2 gtest_parent_2  (cost=0.00..1.01 rows=1 width=14)
               Filter: (f2 = 101)
         ->  Seq Scan on gtest_child3 gtest_parent_3  (cost=0.00..1.01 rows=1 width=14)
               Filter: (f2 = 101)
```

在这个示例中，`Update` 节点需要考虑三个子表，但不包括最初提到的分区表（因为它从不存储任何数据）。因此有三个输入扫描子计划，每个表一个。为清晰起见，`Update` 节点被标注为显示将要更新的具体目标表，顺序与相应子计划相同。

`EXPLAIN ANALYZE` 显示的 `Planning time` 是从解析后的查询生成查询计划并优化它所花费的时间。它不包括解析或重写。

`EXPLAIN ANALYZE` 显示的 `Execution time` 包括执行器启动和关闭时间，以及运行任何被触发的触发器的时间，但不包括解析、重写或规划时间。执行 `BEFORE` 触发器（如果有）所花费的时间包含在相关 `Insert`、`Update` 或 `Delete` 节点的时间中；但执行 `AFTER` 触发器所花费的时间不计入其中，因为 `AFTER` 触发器是在整个计划完成后触发的。每个触发器（无论是 `BEFORE` 还是 `AFTER`）所花费的总时间也会单独显示。注意，延迟约束触发器要到事务结束时才会执行，因此 `EXPLAIN ANALYZE` 完全不考虑它们。

顶层节点显示的时间不包括将查询输出数据转换为可显示形式或发送给客户端所需的任何时间。虽然 `EXPLAIN ANALYZE` 从不将数据发送给客户端，但可以通过指定 `SERIALIZE` 选项，让它将查询输出数据转换为可显示形式并测量所需时间。该时间将单独显示，并且也包含在总 `Execution time` 中。

### Caveats 注意事项

`EXPLAIN ANALYZE` 测量的运行时间可能在两个重要方面偏离同一查询的正常执行。
首先，由于不会向客户端交付输出行，因此不包括网络传输成本。
除非指定了 `SERIALIZE`，否则也不包括 I/O 转换成本。
其次，`EXPLAIN ANALYZE` 增加的测量开销可能很显著，尤其是在 `gettimeofday()` 操作系统调用较慢的机器上。
可以使用 `pg_test_timing` 工具来测量你系统上的计时开销。

`EXPLAIN` 的结果不应外推到与你实际测试的情况差别很大的场景；
例如，不能假定玩具大小的表上的结果适用于大表。
规划器的代价估算不是线性的，因此它可能会为更大或更小的表选择不同的计划。
一个极端的例子是，对于只占用一个磁盘页的表，无论是否有可用索引，你几乎总是会得到顺序扫描计划。
规划器意识到无论如何处理该表都需要读取一个磁盘页，因此额外花费页读取去查看索引没有价值。
（我们在上面的 `polygon_tbl` 示例中看到了这种情况。）

有些情况下，实际值和估计值不会很好地匹配，但并没有什么真正出错。
一种这样的情况是计划节点执行因 `LIMIT` 或类似效果而提前停止。
例如，在我们之前使用的 `LIMIT` 查询中：

```sql
EXPLAIN ANALYZE SELECT * FROM tenk1 WHERE unique1 < 100 AND unique2 > 9000 LIMIT 2;

                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.29..14.33 rows=2 width=244) (actual time=0.051..0.071 rows=2.00 loops=1)
   Buffers: shared hit=16
   ->  Index Scan using tenk1_unique2 on tenk1  (cost=0.29..70.50 rows=10 width=244) (actual time=0.051..0.070 rows=2.00 loops=1)
         Index Cond: (unique2 > 9000)
         Filter: (unique1 < 100)
         Rows Removed by Filter: 287
         Index Searches: 1
         Buffers: shared hit=16
 Planning Time: 0.077 ms
 Execution Time: 0.086 ms
```

`Index Scan` 节点的估计代价和行数显示得就好像它运行至完成一样。
但实际上，`Limit` 节点在获得两行后就停止请求行，因此实际行数只有 2，运行时间也低于代价估算所暗示的值。
这不是估算错误，只是估算值和真实值的显示方式存在差异。

合并连接也有测量假象，可能会让不谨慎的人困惑。
如果合并连接已经耗尽一个输入，并且另一个输入中的下一个键值大于另一个输入的最后一个键值，那么它就会停止读取那个输入；
在这种情况下不可能再有匹配，因此无需扫描第一个输入的剩余部分。
这会导致没有读取某个子节点的全部内容，结果类似于前面提到的 `LIMIT` 情况。
此外，如果外部（第一个）子节点包含重复键值的行，内部（第二个）子节点会被回退并重新扫描其匹配该键值的那部分行。
`EXPLAIN ANALYZE` 将这些同一内部行的重复发出计为真实的额外行。
当外部重复很多时，内部子计划节点报告的实际行数可能会显著大于内部关系中实际存在的行数。

由于实现限制，`BitmapAnd` 和 `BitmapOr` 节点总是将其实际行数报告为零。

通常，`EXPLAIN` 会显示规划器创建的每个计划节点。
然而，在某些情况下，执行器可以根据规划时不可用的参数值确定某些节点不需要执行，因为它们无法产生任何行。
（目前这只会发生在正在扫描分区表的 `Append` 或 `MergeAppend` 节点的子节点上。）
发生这种情况时，这些计划节点会从 `EXPLAIN` 输出中省略，并改为显示 `Subplans Removed: N` 注解。
