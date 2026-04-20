+++
date = '2026-04-16T8:00:00+01:00'
draft = false
title = 'SQL Performance Explained: The Where Clause'
categories = ['Note']
tags = ['SQL']
+++

在前面一篇文章，介绍了索引的结构，并解释了慢索引的原因。
下面会介绍如何发现并避免这些问题，先从 `where` 语句开始。

`where` 语句定义了 SQL 的搜索条件，因此它属于索引的核心功能范畴：快速寻找数据。
即使 `where` 语句对性能有很大影响，但它常常被随意使用，导致数据库不得不扫描索引的大部分内容。
结果导致：一个编写不当的 `where` 子句是导致查询缓慢的首要因素。

本章解释不同运算符如何影响索引的使用，以及如何确保索引能适用于尽可能多的查询。

### The Equality Operator

等号是 SQL 中最简单也最常用的符号。
影响性能的索引错误仍然非常普遍，尤其是结合多个条件的 `where` 子句中，这种错误尤为常见。

本章展示如何验证索引的使用，并解释如何通过连接索引 _combined conditions_ 来优化组合条件。

#### Primary Keys

下面从最简单的 `where` 子句开始：主键查询。
这里使用 `EMPLOYEES` 表做示例：

```sql
CREATE TABLE employees (
    employee_id   NUMBER        NOT NULL,
    first_name    VARCHAR(1000) NOT NULL,
    last_name     VARCHAR(1000) NOT NULL,
    date_of_birth DATE          NOT NULL,
    phone_number  VARCHAR(1000) NOT NULL,
    CONSTRAINT    employess_pk  PRIMARY KEY (employee_id)
);
```

数据库会自动为主键创建索引，这意味着 `employee_id` 列会有一个索引，即使没有 `create index` 语句。

下面这个查询使用主键检索雇员名称：

```sql
SELECT first_name, last_name
FROM employees
WHERE employee_id = 123
```

这里的 `where` 子句不会匹配到多列，因为主键约束了 `employee_id` 的唯一性。
数据库无需遵循索引叶节点，遍历索引树就足够了。

可以使用执行计划 _execution plan_ 来验证：

```ascii
------------------------------------------------------------------------------
| Id  | Operation                        | Name         | Rows | Cost (%CPU) |
------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |              |    1 |     2   (0) |
|   1 | TABLE ACCESS BY INDEX ROWID      | EMPLOYEES    |    1 |     2   (0) |
| * 2 | INDEX UNIQUE SCAN                | EMPLOYEES_PK |    1 |     1   (0) |
------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
2 - access("EMPLOYEE_ID"=123)
```

Oracle 执行计划展示了一个 `INDEX UNIQUE SCAN`，该操作只检索索引树。
它充分利用了索引的对数可扩展性，几乎和表的大小独立。

在访问索引后，数据库必须进行一个额外的步骤，从表存储中获取查询的数据 `(FIRST_NAME, LAST_NAME)`：
即 `TABLE ACCESS BY INDEX ROWID` 操作。
该操作可能成为性能瓶颈，但在使用 `INDEX QUERY SCAN` 时并不存在此风险。
由于此操作最多只能返回一条记录，因此它最多触发一次表访问。
这意味着导致编码的因素不会在 `INDEX QUERY SCAN` 中出现。

> Primary Keys Without Unique Index 不使用唯一索引的主键

主键并不一定需要一个唯一索引，也可以使用非唯一索引。
这样 Oracle 数据库就不使用索引唯一扫描 `INDEX UNIQUE SCAN`，但使用索引范围扫描 `INDEX RANGE SCAN` 操作。
尽管如此，该约束仍然会保持键值的唯一性，因此索引查询最多只会返回一条记录。

为主键使用非唯一索引的原因之一是可延迟约束 _Deferrable Constraints_。
在与语句执行期间进行验证的普通索引不同，数据库会将可延迟约束的验证推迟到事务提交时进行。
在向具有循环依赖 _Circular Dependencies_ 的表中插入数据时，必须使用延迟约束。

#### Concatenated Indexes

尽管数据库会自动为主键创建索引，但如果主键由多个列组成，仍有手动优化的空间。
在这种情况下，数据库会针对所有主键列创建一个索引，即所谓的级联索引 _Concatenated Index_，也称为多列索引，复合索引和组合索引。
要注意，级联索引的列顺序对可用性有很大影响，这里必须谨慎选择。

为了演示的目的，现在假设有一次公司合并。
其他公司的雇员都加入了 `employees` 表，因此变成了原来的 10 倍大。
这里只有一个问题：`employee_id` 在不同的公司内部不是唯一的。
我们需要通过额外的标识符，例如子公司 ID 来扩展主键。
因此，新的主键有两列：`employee_id` 员工与之前相同，以及用于重新确立唯一性的子公司 ID `subsidiary_id`。

因此主键的索引通过下面这种方式去新建：

```sql
CREATE UNIQUE INDEX employee_pk
ON employees (employee_id, subsidiary_id)
```

对于特定员工的查询必须考虑完整的主键，也就是说，必须使用 `subsidiary_id` 列：

```sql
SELECT first_name, last_name
FROM employees
WHERE employee_id == 123
AND subsidiary_id == 30;
```

每当查询使用完整的主键时，无论索引包含多少列，数据库都可以执行索引唯一扫描。
如果只使用其中一个关键列，去搜索所有员工会怎么样呢？

```sql
SELECT first_name, last_name
FROM employees
WHERE subsidiary_id == 20;
```

有下面输出

```
----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           | 106  | 478  |
| *1 | TABLE ACCESS FULL | EMPLOYEES | 106  | 478  |
----------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
 1 - filter("SUBSIDIARY_ID"=20)
```

该查询执行过程没有使用索引，反而它执行了一次全表扫描 `FULL TABLE SCAN`。
数据库查找了整个数据库表，并根据 `where` 子句对每一行进行评估。
该执行直接会随着表的大小增长，如果表变为原来 10 倍大，则全表扫描耗时也会变为 10 倍。
这种情况的危险在于，对于足够小的开发环境而言，速度足够块，但是在生产环境就会有严重的性能问题。

> Full Table Scan

全表扫描操作，在某些情况下，尤其是检索表的大部分数据时，仍然可能是最高效的操作方式。

这部分是由于索引查询 _Index Lookup_ 本身的开销导致，而全表扫描操作避开了这一点。
这主要是因为索引查找会逐个读取数据块，因为数据库在处理完当前块之前，并不知道接下来要读取哪个块。
相比之下，全表扫描必须获取整个表的数据，因此数据库可以一次型读取更大的数据块。
虽然数据库读取的总量增加了，但实际的读取操作（I/O 次数）反而可能更少。

---

数据库不使用索引，因为它无法随意使用符合索引中的单个列。

级联索引 _concatenated index_ 与其他 B 树索引一样，都索引数据存储在有序列表中。
数据库根据在索引定义中的顺序来排序索引条目。
第一列是主要的排序标准，第二列仅在第一列中有相同的值时决定顺序，以此类推。

我们当然可以额外为 `subsidiary_id` 增加一个索引来提升速度。
然而，存在一个更好的解决方案，即假设仅凭 `employee_id` 进行搜索没有意义。
可以颠倒索引列的顺序，将 `subsidiary_id` 放在首位：

```sql
CREATE UNIQUE INDEX employee_pk
ON employees (subsidiary_id, employee_id);
```

这样的索引列仍然是唯一的，主键仍然可以使用唯一索引扫描 `INDEX UNIQUE SCAN`，只是索引条目的顺序完全不同。
这时 `subsidiary_id` 会变成主要的排序标准。
这意味着子公司 _subsidiary_ 的所有条目在索引中是连续排列的，因此数据库可以利用 B 树来确定他们的位置。

> 定义级联索引最重要的考虑因素是如何选择列的顺序，以便尽可能频繁的使用它。

执行计划确定数据库使用了 “反向” 索引，由于仅凭 `subsidiary_id` 已不再具有唯一性。
数据库必须遍历叶子节点以查找所有匹配的条目，因此，它采用了索引范围扫描操作 _INDEX RANGE SCAN_。

```ascii
----------------------------------------------------------------
| Id | Operation                   | Name        | Rows | Cost |
----------------------------------------------------------------
|  0 | SELECT STATEMENT            |             | 106  | 75   |
|  1 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES   | 106  | 75   |
| *2 | INDEX RANGE SCAN            | EMPLOYEE_PK | 106  | 2    |
----------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
 2 - access("SUBSIDIARY_ID"=20)
```

通常情况下，当搜索条件包含前导列 _leading columns_ 时，数据库可以使用级联索引。
一个包含三列的索引可以用于搜索第一列、同时搜索前两列，以及搜索所有列的场景。

尽管双索引解决方案提供很好的查询性能，但是单索引方案更加可取。
它不仅更省内存，并且比第二个索引的维护开销更小。
一张表的索引越少，插入、删除和更新的性能就更好。

要定义一个最优的索引，必须要理解索引如何工作，并且也要知道程序会如何查询数据。
这意味着需要了解出现在 `where` 子句中的列组合。

因此，对于外部顾问而言，要定义一个最优的索引是十分困难的，因为他们没有整个应用的访问路径。
顾问通常只考虑单一查询，他们未能充分利用索引可能为其他查询带来的额外优势。
数据管理员也面临类似情况，他们可能了解数据库架构，但对访问路径缺乏深入洞察。

数据库知识与业务领域知识交汇的唯一场所是开发部门。
开发者对数据具有敏锐的感知力，并熟悉访问路径。
他们能够通过恰当的索引配置，以最小的努力为整个程序带来最佳性能。

#### Slow Indexs, Part 2

前面已经解释了如何通过改变现有索引的顺序，来获取额外的好处，但该示例仅考虑了两个 SQL 语句。
然而，更改索引可能会影响所有针对该索引表的查询。
本节将解释数据库索引的方式，并展示在更改现有索引时可能产生的负作用。

> The Query Optimizer

查询优化器或叫查询计划器，会将 SQL 语句转换为一个执行计划。
该过程也称为编译 _compiling_ 或解析 _prasing_，存在两种不同的优化器类型。

_Cost-based optimizers_ (CBO) 会生成很多执行计划变种，并计算每个计划的成本。
成本计算基于当前使用的操作和预估的行数。
最终，成本值成为选择“最佳”执行计划的基准

_Rule-Based optimizers_ (RBO) 使用一个硬编码的规则集来生成执行计划。
基于规则的优化器不够灵活，现今更少使用。

---

更改索引也可能带来副作用。
在之前的例子中，自合并依赖，内部电话目录应用程序正在变得非常缓慢。
初步分析指明，以下查询是导致速度变慢的原因：

```sql
SELECT first_name, last_name, subsidiary_id, phone_number
FROM employees
WHERE last_name = 'WINAND'
AND subsidiary_id = 30;
```

执行计划如下

```ascii
------------------------------------------------------------------
| Id | Operation                   | Name          | Rows | Cost |
------------------------------------------------------------------
|  0 | SELECT STATEMENT            |               |    1 |   30 |
| *1 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES     |    1 |   30 |
| *2 | INDEX RANGE SCAN            | EMPLOYEES_PK  |   40 |    2 |
------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter("LAST_NAME"='WINAND')
   2 - access("SUBSIDIARY_ID"=30)
```

执行计划使用了一个索引，总成本为 30。
到目前为止，一切看起来不错。
然而，它可疑地使用了之前修改过的索引，这足以让我们怀疑，索引的变更导致了性能问题。
有其实考落到旧的索引定义时，它始于 `employee_id` 列，而该列根本不属于 `where` 子句的一部分。
在此之前，查询无法使用该索引。

为了进一步分析，如果能对比修改前后的执行计划则会很有帮助。
为了得到原来的执行计划，我们可以修改索引为原来的样子。
然而，大部分数据库提供了一个更加简单的方案来避免使用索引。
下面的例子使用 Oracle 优化器提示：

```sql
SELECT /*+ NO_INDEX(EMPLOYEES, EMPLOYEES_PK) */
       first_name, last_name, subsidiary_id, phone_number
FROM employees
WHERE last_name = 'WINAND'
AND subsidiary_id = 30;
```

在索引变更之前，可能使用的执行计划根本没有使用索引

```ascii
----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           |    1 |  477 |
| *1 | TABLE ACCESS FULL | EMPLOYEES |    1 |  477 |
----------------------------------------------------

Predicate Information (identified by operation id):
----------------------------------------------------
   1 - filter("LAST_NAME"='WINAND' AND "SUBSIDIARY_ID"=30)
```

即使 `TABLE ACCESS FULL` 必须读取和处理整个表，它看上去比使用索引更快。
这是非常不寻常的，因为查询只得到了一行。
使用索引查找一行应该比全表扫描更快才对，但该示例不是。
索引查询看上去要慢得多。

在这种情况下，最好是逐步分析执行计划的每一步。
第一步是在 `EMPLOYEES_PK` 索引上进行 `INDEX RANGE SCAN`。
该索引不包含 `LAST_NAME` 列，该索引范围搜索只能通过 `SUBSIDIARY_ID` 过滤。
Oracle 数据库在谓词信息 "_Predicate Information_" 区域，_execution plan_ 的条目 2 中显示此信息。
在那里，可以看到适用于每个操作的条件。

索引范围扫描 `INDEX RANGE SCAN` 的操作 ID 2 只应用了 `subsidiary_id=30` 过滤器。
这意味着它会遍历索引树，来找到第一个 `subsidiary_id` 等于 30 的入口。
接下来，它沿着叶节点查找该子公司的所有其他条目。
索引范围扫描的结果会得到满足下面条件的 ROWID 列表：取决于子公司的大小，可能会有几个或几百个结果。

下一步是 `TABLE ACCESS BY INDEX ROWID` 操作。
它会使用上一步得到的 ROWIDs 获取表中所有列的数据行。
只要 `last_name` 列可用，数据库就可以过滤 `where` 语句的剩余部分。
这意味着数据库必须先获取所有 `subsidiary_id=30` 的行，然后才能用 `last_name` 筛选。

该语句的响应时间不由结果集的大小决定，而是由某个子公司的雇员数量决定。
如果子公司只有几个成员，那么 `INDEX RANGE SCAN` 会有更好的性能。
然而，对于大型公司，全表扫描可能更快，因为它可以一次读取表中更多的数据。

该查询慢是因为索引查询返回许多 ROWIDs，每个原公司员工对应一个，数据库必须逐个获取他们。
这正是两种成分的完美结合，才使得索引变得缓慢：
数据库读取很大范围的索引，并逐个获取他们。

选择最佳执行计划同样依赖于表的数据分部，因此优化器会利用数据库内容的统计信息。
在上面例子中，使用了一个直方图来展示员工分部情况。
这使得优化器能够估算从索引查找返回的行数，该结果用于成本计算。

> Statistics

基于成本的优化器会使用关于表、列和索引的统计信息。
大部分数据都是在列的层面收集的：不同值的数量、最小最大值（数据范围）、NULL 出现的次数和列直方图（数据分部）。
一个表最重要的统计值是它的大小（行和块）。

索引最重要的统计信息是树深度，叶节点的数量，不同键的数量和聚类因子 _clustering factor_。

优化器利用这些值来估算 `where` 子句谓词的选择性。

---

如果没有可用的统计数据，例如他们已被删除，优化器将使用默认值。
Oracle 的默认值是一个选择性中等的小索引。
这导致估计 `INDEX RANGE SCAN` 将返回 40 行数据。
执行计划在行数列中展示了这一估算，显然这是一个严重的低估，因为该公司有 1000 名员工。

如果我们提供正确的统计信息，优化器则会表现更好。
下面执行计划展示了新的估计：1000 行的 `INDEX RANGE SCAN` 结果。
因此，它为后续的表格访问计算出了一个更高的成本值。

```ascii
------------------------------------------------------------------
| Id | Operation                   | Name          | Rows | Cost |
------------------------------------------------------------------
|  0 | SELECT STATEMENT            |               |    1 |  680 |
| *1 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES     |    1 |  680 |
| *2 | INDEX RANGE SCAN            | EMPLOYEES_PK  | 1000 |    4 |
------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter("LAST_NAME"='WINAND')
   2 - access("SUBSIDIARY_ID"=30)
```

该执行计划的成本为 680，甚至比全表扫描的 477 更高。
因此优化器会选择全表扫描 `FULL TABLE SCAN`。

这个慢索引的例子不应该掩盖一个事实：即适当的索引才是最佳解决方案。
当然，在 `last_name` 上进行搜索时，为该字段建立索引才是最佳方式。

```sql
CREATE INDEX emp_name ON employees (last_name);
```

有合适索引的执行计划如下：

```sql
--------------------------------------------------------------
| Id | Operation                   | Name      | Rows | Cost |
--------------------------------------------------------------
|  0 | SELECT STATEMENT            |           |    1 |    3 |
| *1 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES |    1 |    3 |
| *2 | INDEX RANGE SCAN            | EMP_NAME  |    1 |    1 |
--------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter("SUBSIDIARY_ID"=30)
   2 - access("LAST_NAME"='WINAND')
```

根据优化器的故及，索引访问仅返回一行数据。
因此，数据库只需从表中获取这一行：这显然比全表扫描更快。
一个正确定义的索引仍然优于原始的全表扫描。

上面两个执行计划几乎相同。
数据库执行相同的操作，优化器成本计算也相似。
然而，第二个计划的性能却好得多。
索引范围扫描的效率可能在很大范围内变化，尤其是在其后跟会表访问时。
使用索引不自动意味着语句以最佳方式执行。

### Functions
