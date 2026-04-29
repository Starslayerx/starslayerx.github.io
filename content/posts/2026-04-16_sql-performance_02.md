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

## The Equality Operator

等号是 SQL 中最简单也最常用的符号。
影响性能的索引错误仍然非常普遍，尤其是结合多个条件的 `where` 子句中，这种错误尤为常见。

本章展示如何验证索引的使用，并解释如何通过连接索引 _combined conditions_ 来优化组合条件。

### Primary Keys

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

### Concatenated Indexes

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

### Slow Indexs, Part 2

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

```ascii
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

根据优化器的估计，索引访问仅返回一行数据。
因此，数据库只需从表中获取这一行：这显然比全表扫描更快。
一个正确定义的索引仍然优于原始的全表扫描。

上面两个执行计划几乎相同。
数据库执行相同的操作，优化器成本计算也相似。
然而，第二个计划的性能却好得多。
索引范围扫描的效率可能在很大范围内变化，尤其是在其后跟会表访问时。
使用索引不自动意味着语句以最佳方式执行。

## Functions

使用 `last_name` 索引显著提升了性能，但是要求使用数据库中一样的写法（大写、小写）。
这节介绍如何解除这方面的限制，并且不降低性能。

### Case-Insensitive Search Using `UPPER` or `LOWER`

在 `where` 中忽略写法是很容易的。
例如，将两边的写法都转换为大写：

```sql
SELECT first_name, last_name, phone_number
FROM employees
WHERE UPPER(last_name) = UPPER('winand');
```

无论搜索词或 `last_name` 列使用何种大小写，`UPPER` 函数都能使他们按预期匹配。

> 另一种实现不区分大小写匹配的方法是使用不同的 “排序规则”。
> SQL Server 和 MySQL 默认使用的排序规则，默认情况下不区分大小写字母。

该查询逻辑看上去很合理，但是执行计划并非如此：

```ascii
----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           |   10 |  477 |
| *1 | TABLE ACCESS FULL | EMPLOYEES |   10 |  477 |
----------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter(UPPER("LAST_NAME")='WINAND')
```

计划器又选择进行全表扫描。
即使在 `last_name` 建立了索引，但这里并没有使用，因为搜索并非基于 `last_name`，而是 `UPPER(last_name)`。
从数据库的视角来看，这是完全不同的。

这是一个可能落入的陷阱。
我们能意识到 `last_name` 和 `UPPER(last_name)` 之间的关系，但从数据库来看。
优化器的视角是这样：

```sql
SELECT first_name, last_name, phone_number
FROM employees
WHERE BLACKBOX(...) = 'WINAND';
```

> Compile Time Evaluation

优化器可以在“编译时”评估右侧的表达式，因为它拥有所有输入参数。
因此 Oracle 执行计划仅显示搜索词的大写形式。
这种行为与在编译时评估常量表达式的编译期非常类似。

---

为了支持该查询，需要一个覆盖实际搜索条件的索引。
这意味着，不需要在 `last_name` 上建立索引，而是需要在 `UPPER(last_name)` 上建立索引。

```sql
CREATE INDEX emp_up_name
ON employees (UPPER(last_name));
```

包含函数或表达式的索引，称为函数索引 _function-based index, (FBI)_。
它不会将列数据直接拷贝到索引中，函数索引会使用函数，然后将结果放入索引中。
因此，所以全以大写存储名称。

如果 SQL 语句中出现了索引定义的确切表达式，数据库可以使用基于函数的索引：

```ascii
-----------------------------------------------------------------
| Id | Operation                   | Name         | Rows | Cost |
-----------------------------------------------------------------
|  0 | SELECT STATEMENT            |              |  100 |   41 |
|  1 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES    |  100 |   41 |
| *2 | INDEX RANGE SCAN            | EMP_UP_NAME  |   40 |    1 |
-----------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access(UPPER("LAST_NAME")='WINAND')
```

这是一种常规的索引范围扫描。
数据库遍历 B 树并沿着叶子节点链进行。
对于基于函数的索引，没有专门的操作或关键字。

> 有时候 ORM 工具会自动使用 `UPPER` 和 `LOWER`。
> 例如 Hibernate 会为大小写敏感的搜索注入 `LOWER`。

执行计划仍然与之前不带 `UPPER` 函数的情况不同，其行数估计过高。
奇怪的是，优化器预期从表中获取的行数比之前索引范围扫描交付的行数还多。

这种矛盾的估算通常表明统计信息存在问题。
在这个例子中，是因为 Oracle 数据库在创建新索引时，不会自动更新表统计信息。

重新更新统计数据后，优化器会得到更加准确的估计

```ascii
-----------------------------------------------------------------
| Id | Operation                   | Name         | Rows | Cost |
-----------------------------------------------------------------
|  0 | SELECT STATEMENT            |              |    1 |    3 |
|  1 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES    |    1 |    3 |
| *2 | INDEX RANGE SCAN            | EMP_UP_NAME  |    1 |    1 |
-----------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access(UPPER("LAST_NAME")='WINAND')
```

SQL Server 不支持函数索引，它提供了可以替代使用的计算列 _computed columns_。

```sql
ALTER TABLE employees
ADD last_name_up AS UPPER(last_name);

CREATE INDEX emu_up_name
ON employees (last_name_up);
```

每当索引表达式出现在语句时，SQL Server 都能使用这个索引。

> Oracle Statistics for Function-Based Indexes

Oracle 数据库将不同列值的信息作为统计的一部分进行维护。
如果一个列属于所个索引，这些数据库会被重复使用。

基于函数的索引 FBI 的统计信息也以虚拟列 _virtual columns_ 的形式在表级别进行维护。
尽管 Oracle 数据库会自动为新索引搜集统计信息，但从 10g 版本起，它不会更新统计信息。
对于这个原因，Oracle 文档推荐在创建函数索引后更新表统计信息：

```
创建基于函数的索引后，使用 DBMS_STATS 包收集该索引及其基表的统计信息。
这些统计信息将帮助Oracle数据库正确判断何时使用该索引。
    -- Oracle Database SQL Language Reference
```

建议每次索引变更后，更新表及其所有索引的统计信息。
然而，这也可能导致不必要的副作用。
请与 DBA 协调此活动，并备份原始统计信息。

### User-Defined Functions

基于函数的索引是一个非常通用的方式。
除了 `UPPER` 这样的函数，还可以使用 `A + B` 这样的表达式，甚至用户定义的函数。

有一个重要的例外情况。
例如，在索引定义中，无论是直接还是间接地引用当前时间都是不可能的：

```sql
CREATE FUNCTION get_age(date_of_birth DATE)
RETURN NUMBER
AS
BEGIN
  RETURN
    TRUNC(MONTHS_BETWEEN(SYSDATE, date_of_birth) / 12);
END;
/
```

函数 `get_age` 使用当前日期 `SYSDATE` 和提供的出生日期，来计算年龄。
可以使用该函数作为 SQL 查询的一部分：

```sql
SELECT first_name, last_name, get_age(date_of_birth)
FROM employees
WHERE get_age(date_of_birth) < 42;
```

该查询计算所有小于 42 岁的员工。
使用基于函数的索引明显是一个很好的查询优化，但并不能在索引中使用 `get_age` 函数，因为它不是确定的。
这意味着函数调用的结果并不完全有其参数决定。
只有同样参数总是返回同样结果的，这种确定的函数，才能作为索引。

除了要求确定性，PostgreSQL 和 Oracle 数据库还要求函数在索引中使用时必须声明为确定性。
因此必须使用关键字 `DETERMINISTIC` (Oracle) 或 `IMMUTABLE` (PostgreSQL)。

其他不能被索引的函数例子有随机数生生成器，或者依赖环境变量的函数。

### Over-Indexing

如果对基于函数的索引概念感到陌生，可能会倾向于对所有内容都建立索引，但这实际上是最不应该做的事情。
原因是所有索引都需要持续的维护。
基于函数的索引尤其棘手，因为它们使得创建冗余索引变得非常容易。

上面的大小写敏感的搜索也可以使用 `LOWER` 函数实现：

```sql
SELECT first_name, last_name, phone_number
FROM employees
WHERE LOWER(last_name) = LOWER('winand');
```

单个索引无法同时支持大小写两种方法。
当然可以再为小写创建一个索引，但是这会导致数据库维护两份索引。
每次 insert, update 和 delete 都会去修改索引。
单个索引就足够了，并且应该在应用代码中始终使用同一个函数。

## Parameterized Queries

本节会讲解大部分 SQL 书籍都会跳过的部分：参数化查询 _parameterized queries_ 和绑定参数 _bind parameters_。

绑定参数 _bind parameters_ 也称作动态参数 _dynamic parameters_，是向数据库传递数据的另一种方式。
不同于直接将值放入 SQL 语句中，只需要使用像 `?, :name` 或 `@name` 这样的占位符，并通过单独的 API 调用来提供实际值。

直接将数字化写入临时语句并无不妥，然而，在程序中直接绑定参数有两个好处：

- 安全

  绑定变量是最好的避免 SQL 注入的方式

- 性能

  像 SQL Server 和 Oracle 数据库这样带有执行计划缓存的数据库，在执行相同语句多次时，重复使用执行计划。
  这会节约重建执行计划的消耗，但只有在 SQL 完全相同的使用才触发。
  如果在 SQL 语句中放入不同的值，则数据库会将其视为不同的 SQL 语句，导致重建执行计划。

  当使用绑定参数的时候，不需要编写实际的值，而是在 SQL 语句里面插入占位符。
  这样一来，即使使用不同的值执行这些语句，它们都不会发生变化。

自然的，会有例外情况。
例如，如果受影响的数据量取决于实际的值：

```ascii
99 rows selected.

SELECT first_name, last_name
FROM employees
WHERE subsidiary_id = 20;

----------------------------------------------------------------
| Id | Operation                   | Name        | Rows | Cost |
----------------------------------------------------------------
|  0 | SELECT STATEMENT            |             |   99 |   70 |
|  1 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES   |   99 |   70 |
| *2 | INDEX RANGE SCAN            | EMPLOYEE_PK |   99 |    2 |
----------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("SUBSIDIARY_ID"=20)
```

对于小型子公司，所以查找能提供最佳性能，但对于大型公司，全表扫描可能比索引更高效。

```ascii
1000 rows selected.

SELECT first_name, last_name
FROM employees
WHERE subsidiary_id = 30;

----------------------------------------------------
| Id | Operation         | Name      | Rows | Cost |
----------------------------------------------------
|  0 | SELECT STATEMENT  |           | 1000 |  478 |
| *1 | TABLE ACCESS FULL | EMPLOYEES | 1000 |  478 |
----------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter("SUBSIDIARY_ID"=30)
```

在这种情况下，`subsidiary_id` 的直方图 _histogram_ 发挥了它的作用。
优化器利用它来确定 SQL 查询中提到的 `subsidiary_id` 的出现频率。
因此，它为这两个查询得到了不同的行数估算。

`TABLE ACCESS BY INDEX ROWID` 操作的成本对行数估计高度敏感。
选择十倍数量的行将使成本值相应提升，使用索引的总体成本甚至比全表扫描还高。
因此，优化器为较大的子公司选择另一种执行计划。

当使用绑定参数的时候，优化器无法获取具体数值来确定其出现频率。
随后，它便假设了一个均匀分布，并始终得出相同的行数估计和成本值。
最终，它总是会选择相同的执行计划。

> TIP

Column histograms 列直方图在数据分布不均匀的时候最有用。
对于均匀分布的列，通常只需要将不同值的数量除以表中的行数。
这种方法在使用绑定参数时同样有效。

---

如果将优化器和编译器对比，绑定参数 _bind variables_ 就像程序变量一样。
如果直接填值，就类似常量。
数据库可以使用 SQL 语句的值进行优化，就像编译器可以在编译器计算常量一样。
绑定参数对于优化器来说是不可见的，就像运行时变量对于编译器。

从这个角度来看，如果不使用绑定参数，就能让优化器选择最佳计划。
那么绑定参数反而能提升性能，这显得有点自相矛盾。
但代价是，生成和估计执行计划要耗费大量资源，如果每次生成的结果都一样就会不划算。

> TIP: 不使用绑定参数就像是每次都要重新编译程序

为数据库选择专门的还是通用的执行计划是一个两难的选择。

为了确保每次都能获得最优的执行计划，数据库必须每次执行时都评估所有可能的计划变体。
要么为了节省优化开销，在可能的情况下尽量使用缓存的执行计划。
但这需要承担使用次优 _suboptimal_ 计划的风险。

这种困境在于：如果不实际进行完整的优化流程，数据库就无法欲知该流程是否会产生不同的执行计划。
数据库厂商试图通过启发式方法 _heuristic methods_ 来解决这一难题，但取得的成效非常有限。

作为开发者，可以刻意使用绑定参数来解决这一问题。
即，除非变量会影响执行计划，否则应该总是使用绑定参数。

不均匀分布的状态码，例如 "todo" 和 "done" 状态码就是一个很好的例子。
通常，"done" 条目的数量比 "todo" 多出一个数量级。
这种情况下，使用索引仅在搜索 "todo" 条目时才有意义。

分区 _partitioning_ 是另一个例子，如果将表和索引拆分到多个存储区域。
在这种情况下，实际的数值会影响到哪些分区 _partitions_ 需要被扫描。
此外，LIKE 查询的性能也会受到绑定参数的影响。

> TIP: 在现实情况中，只有在少数情况下，变量会影响执行计划。
> 因此，应该毫不犹豫地使用绑定参数，即使只是为了防止 SQL 注入。

不使用绑定参数：

```python
import psycopg

subsidiary_id = 101

# 错误做法：字符串格式化
sql = "SELECT first_name, last_name FROM employees WHERE subsidiary_id = " + str(subsidiary_id)

with psycopy.connect('dbname=test user=postgres') as conn:
    with conn.cursor() as cur:
        cur.execute(sql)
```

使用绑定参数：

```python
import psycopg

subsidiary_id = 101

# 正确做法：使用 %s 占位符
sql = "SELECT first_name, last_name FROM employees WHERE subsidiary_id = %s"

with psycopy.connect("dbname=test user=postgres") as conn:
    with conn.cursor() as cur:
        # 参数作为元组传入
        cur.execute(sql, (subsidiary_id,))

        for record in cur:
            print(record)
```

如果数据库驱动是 sqlite，则占位符使用 `?`，例如：

```python
sql = "SELECT first_name, last_name FROM employees WHERE subsidiary_id = ?"
```

问号 `?` 是 SQL 标准定义的唯一占位符字符。
属于位置参数，这意味着问号是按照从左到右的顺序进行编号的。
若要将某个值绑定到特定的问号上，必须指定它的编号。

然而，这种方式在实际操作中可能非常不便，因为一旦增加或删除占位符，原有的编号顺序就会发生改变。
为了解决这个问题，许多数据库提供了针对命名参数 _named parameters_ 的私有扩展。
例如，使用 at 符号 `@name` 或冒号 `:name`。

绑定参数不能修改 SQL 语句的结构。
这意味着，不能使用其作为表或者列的名称。
下面这样写无法执行：

```python
sql = "SELECT * FROM ? WHERE ? = ?"
params = ('employees', 'id', 1)
```

> Cursor Sharing and Auto Parameterization

SQL Server 和 Oracle 数据库都具备自动将 SQL 字符串中的字面量值 _Literal Values_ 替换为绑定参数的功能。
这些特性在 Oracle 中被称为 `CURSOR_SHARING`，在 SQL Server 中被称为强制参数化 `Forced Parameterization`。

## Sharing For Ranges

不等号，例如 `>, <` 和 `between` 可以使用索引，会像上面解释的等于运算符一样使用索引。
甚至 `LIKE` 过滤器在某些情况下，也会像范围查询一样使用索引。

这些操作会限制复合索引 _multi-column indexes_ 中列顺序的选择。
这种限制甚至可能排除所有最优的索引选项，在某些查询中，根本无法定义一个正确的列顺序。

### Greater, Less and Between

索引范围扫描 `INDEX RANGE SCAN` 最大的性能风险是叶子节点检索。
因此索引的黄金标准是要让索引范围尽可能的小。

例如这个查询的范围就很明确：

```sql
SELECT first_name, last_name, date_of_birth
  FROM employees
 WHERE date_of_birth >= TO_DATE(?, 'YYYY-MM-DD')
   AND date_of_birth <= TO_DATE(?, 'YYYY-MM-DD')
```

对出生日期 `date_of_birth` 上的索引只能根据特定的范围搜索。
扫描从第一个日期开始，到第二个日期结束，无法进一步缩小扫描的范围。

如果涉及到第二列，起始和停止条件就不那么明显了：

```sql
SELECT first_name, last_name, date_of_birth
  FROM employees
 WHERE date_of_birth >= TO_DATE(?, 'YYYY-MM-DD')
   AND date_of_birth <= TO_DATE(?, 'YYYY-MM-DD')
   AND subsidiary_id = ?
```

一个理想的索引应该同时包含这两列，但问题是以哪种顺序呢？
正确的方式是 `subsidiary_id` 在前，如果日期在前，则叶子节点还需要访问并判断子公司 ID。
如果子公司 ID 放在前面，在叶子节点就不需要额外判断了。

> TIP: 经验法则 - 先为相等性建立索引，再为范围建立索引。

实际性能取决于数据和搜索条件。
如果 `date_of_birth` 上的过滤器本身具有很高的选择性，那么这种差异可以忽略不计。
日期范围越大，性能差异也会越显著。

通过该例子，可以证伪 _falsify_ 一个谬论：选择性最强的列应该位于最左侧的索引位置。
如果查看数据，并仅考虑第一列的选择性，会发现两个条件都匹配了 13 条记录。
无论仅按 `date_of_birth` 筛选，还是仅按 `subsidiary_id` 筛选，情况都是如此。
选择性在这里没有用处，但一种列顺序仍然优于另一种。

为了优化性能，了解索引扫描范围很重要。
在大多数数据库中，甚至可以在执行计划中看到这一点，只需要知道该寻找什么即可。
一下来自 Oracle 数据库执行计划明确表明 `EMP_TEST` 索引以 `date_of_birth` 列开头。

```ascii
--------------------------------------------------------------
| Id | Operation                   | Name      | Rows | Cost |
--------------------------------------------------------------
|  0 | SELECT STATEMENT            |           |    1 |    4 |
|* 1 | FILTER                      |           |      |      |
|  2 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES |    1 |    4 |
|* 3 | INDEX RANGE SCAN            | EMP_TEST  |    2 |    2 |
--------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter(:END_DT >= :START_DT)
   3 - access(DATE_OF_BIRTH >= :START_DT
          AND DATE_OF_BIRTH <= :END_DT)
       filter(SUBSIDIARY_ID = :SUBS_ID)
```

索引范围扫描 INDEX RANGE SCAN 的谓词 _predicate_ 信息提供了关键线索。
它标识了 WHERE 子句中的条件是作为访问谓词还是过滤谓词。
这正是数据库告诉我们它如何使用每个条件的方式。

`date_of_birth` 列的条件是唯一列出的访问谓词。
他们限制了扫描的索引范围。
因此 `date_of_birth` 是 `EMP_TEST` 索引中的第一列，`subsidiary_id` 列仅作用过滤器。

> 访问谓词 _access predicates_ 是索引检索的开始和结束条件，他们定义了扫描范围。
> 索引过滤谓词仅在叶节点遍历其间应用，他们不会缩小扫描的索引范围。

如果我们将索引定义反过来，数据库可以使用所有的条件和访问谓词信息：

```ascii
--------------------------------------------------------------
| Id | Operation                   | Name      | Rows | Cost |
--------------------------------------------------------------
|  0 | SELECT STATEMENT            |           |    1 |    3 |
| *1 | FILTER                      |           |      |      |
|  2 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES |    1 |    3 |
| *3 | INDEX RANGE SCAN            | EMP_TEST2 |    1 |    2 |
--------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter(:END_DT >= :START_DT)
   3 - access(SUBSIDIARY_ID = :SUBS_ID
          AND DATE_OF_BIRTH >= :START_DT
          AND DATE_OF_BIRTH <= :END_T)
```

最后，又一个 `between` 操作符。
它可以在单个条件中指定上下边界：

```sql
DATE_OF_BIRTH BETWEEN '01-JAN-71' AND '10-JAN-71'
```

注意 `between` 总是包含特定的值，就像使用小于或等于和大于等于符号一样：

```sql
DATE_OF_BIRTH >= '01-JAN-71' AND
DATE_OF_BIRTH <= '10-JAN-71'
```

### Indexing LIKE Filters

SQL 中的 `LIKE` 操作符经常会导致意料之外的性能影响，因为某些搜索词会阻止索引的有效使用。
这意味着，有些项可以高效地索引，但是其他的不行。
关键在于通配符的位置。

下面的例子在搜索词中间使用了 % 通配符：

```sql
SELECT first_name, last_name, date_of_birth
FROM employees
WHERE UPPER(last_name) LIKE 'WIN%D'
```

执行计划如下

```ascii
-----------------------------------------------------------------
| Id | Operation                   | Name         | Rows | Cost |
-----------------------------------------------------------------
|  0 | SELECT STATEMENT            |              |    1 |    4 |
|  1 | TABLE ACCESS BY INDEX ROWID | EMPLOYEES    |    1 |    4 |
| *2 | INDEX RANGE SCAN            | EMP_UP_NAME  |    1 |    2 |
-----------------------------------------------------------------
```

`LIKE` 过滤器在树遍历过程中只能使用第一个通配符之前的字符。
剩余的字符知识过滤谓词，不会缩小扫描的索引范围。
因此，一个单一的 `LIKE` 表达式可以包含两种谓词类型。

1. 第一个通配符之前的部分作为访问谓词 _access predicate_
2. 其他字符作为过滤谓词 _filter predicate_

> Caution

对于 Postgres 数据库，可能需要指定 operator class 以使用 LIKE 表达式作为 access predicates。
详见 Operator Classes and Operator Families 的 Postgres 文档。

---

首个通配符前的前缀选择性越高，索引搜索范围就越小，从而让索引查询更快。

| LIKE 'WI%ND' | LIKE 'WIN%D' | LIKE 'WINA%' |
| ------------ | ------------ | ------------ |
| `WI`AW       | WIAW         | WIAW         |
| `WI`BLQQNPUA | WIBLQQNPUA   | WIBLQQNPUA   |
| `WI`BYHSNZ   | WIBYHSNZ     | WIBYHSNZ     |
| `WI`FMDWUQMB | WIFMDWUQMB   | WIFMDWUQMB   |
| `WI`GLZX     | WIGLZX       | WIGLZX       |
| `WI`H        | WIH          | WIH          |
| `WI`HTFVZNLC | WIHTFVZNLC   | WIHTFVZNLC   |
| `WI`JYAXPP   | WIJYAXPP     | WIJYAXPP     |
| **`WI`NAND** | **`WIN`AND** | **`WINA`ND** |
| `WI`NBKYDSKW | `WIN`BKYDSKW | WINBKYDSKW   |
| `WI`POJ      | WIPOJ        | WIPOJ        |
| `WI`SRGPK    | WISRGPK      | WISRGPK      |
| `WI`TJIVQJ   | WITJIVQJ     | WITJIVQJ     |
| `WI`W        | WIW          | WIW          |
| `WI`WGPJMQGG | WIWGPJMQGG   | WIWGPJMQGG   |
| `WI`WKHLBJ   | WIWKHLBJ     | WIWKHLBJ     |
| `WI`YETHN    | WIYETHN      | WIYETHN      |
| `WI`YJ       | WIYJ         | WIYJ         |

第一个表达式在通配符之前有两个字符匹配。
它将索引范围限制到了 18 行，只有一个匹配到了 `LIKE` 表达式，剩下的 17 个都丢弃了。
第二个表达式前缀长一点，将索引范围缩小到了 2 行，这样数据库只多读了一行数据。
最后一个表达式没有过滤谓词，数据库会读取所有匹配项。

> Important: 只有第一个通配符之前的会作为访问谓词 _access predicate_
> 剩下的字符不会缩小索引扫描范围，不匹配的条目只会被排除在结果之外

相反的情况也是可能的：一个以通配符开头的 `LIKE` 表达式。
这样的表达式不能作为谓词使用，如果没有其他访问谓词的条件，数据库必须全表扫描。

> Tip: 避免使用 `%TERM` 这种通配符在前的表达式

通配符字符 _wildcard characters_ 的位置会影响索引的使用，至少在理论上是如此。
实际上，搜索词通过绑定参数提供时，优化器会生成一个通用的执行计划。
这种情况下，优化器必须猜测大多数执行是否会有前导通配符。

大多数数据库在优化带有绑定参数的 `LIKE` 条件时，都假设没有前导通配符。
但如果 `LIKE` 表达式用于全文搜索 _full-text search_，这个假设就是错误的。
遗憾的是，没有直接方法可以将 `LIKE` 条件标记为全文搜索。

不使用绑定参数而直接指定搜索词是一种解决方案，但问题是这回增加优化开销。
并可能引发 SQL 注入漏洞，一种解决方案是故意模糊化 _obfuscate_。

> Labeling Full-Text `LIKE` Expressions

当使用 `LIKE` 操作符进行全文搜索的时候，我们可以将通配符和搜索条件分开：

```sql
WHERE text_column LIKE '%' || ? || '%'
```

这样写有两个好处：

1. 可以复用执行计划
2. `'%' || ? || '%'` 会自动拼接起来，并且防止 SQL 注入

之前介绍过，如果使用绑定参数会影响查询命中率，则会导致优化器不一定使用最优的方案。
而这种情况刚好避开了这个问题，因为全文搜索无论怎么写都会去全表扫描。

---

PostgreSQL 在 `LIKE` 参数未知时会采用保守策略，不假设存在固定前缀，因此通常不会使用索引。
如果要让其使用索引，就需要将要搜索的实际词汇放到 SQL 语句里面，不过这就需要通过其他方式来防御 SQL 注入。

即使数据库针对前缀通配符优化了执行计划，性能可能仍然不如预期。
可以考虑使用各数据库厂商提供的专用于全文搜索解决方案：

- MySQL

  MySQL 提供了 `MATCH` 和 `AGAINST` 关键字用于全文搜索。
  从 MySQL 5.6 版本开始，也可以为 InnoDB 引擎的表创建全文索引（此前仅支持 MyISAM 表）。
  详情请参阅 MySQL 文档中的 Full-Text Search Functions。

- Oracle Databse

  Oracle 数据库提供了 CONTAINS 关键字。
  具体请参考 Oracle Text Application Developer’s Guide。

- PostgreSQL

  PostgreSQL 通过 `@@` 符号实现全文搜索，详见 Full Text Search。
  另一种选择是使用 WildSpeed2 扩展来直接优化 LIKE 表达式。
  该扩展会将文本以所有可能的旋转 _Rotations_ 方式存储，确保每个字符都有一次机会出现在字符串的开头。
  这意味着索引后的文本不仅存储一次，而是根据字符串长度重复存储多次——因此它会占用大量的存储空间。

- SQL Server

  SQL Server 同样提供了 CONTAINS 关键字。
  详见 SQL 的 Full-Text Search 文档。

### Index Merge

关于索引最常见的问题之一：是为每个列分别创建索引，还是为 `where` 子句中的所有列创建一个复合索引好？
大多数情况下答案很简单：包含多个列的单一索引更好。

然而，有些查询无论如何定义索引，单个索引都无法胜任。
例如，包含两个或更多独立范围条件的查询，如下例所示：

```sql
SELECT first_name, last_name, date_of_birth
FROM employees
WHERR UPPER(last_name) < ?
AND date_of_birth < ?
```

要定义一个支持此查询且不产生过滤谓词 _Filter Predicates_ 的 B-tree 索引是不可能的。
要理解这一点，只需要记住 B 树是一个链表。

- 如果这样定义一个索引 `UPPER(last_name)`, `date_of_birth`，顺序从 A 开始，到 Z 结束。
  只有在两名员工同名时，才会考虑出生日期。

- 如果反过来定义索引，则会从最年长的员工开始，到最年轻的员工结束。
  那这样名称只会对排列顺序有很小的影响。

无论如何修改索引定义，条目始终沿着一条链排列。
在一端，是 “小” 条目，另一端是 “大” 型条目。
因此，索引只能支持一个范围条件作为访问谓词。
要支持两个独立的范围条件，就需要第二个纬度。

当然，也可以接受过滤谓词并坚持使用多列索引。
在许多情况下，这仍然是最佳解决方案。
此时，索引定义应将选择性 _Selectivity_ 更高的列放在前面，以便将其用作访问谓词。
这可能就是 “高选择性列优先” 这一迷思的起源，但这条规则仅在无法避免过滤谓词时才成立。

另一个选择是使用两个独立的索引，每列一个。
这样数据库必须扫描两个索引，然后结合两个结果。
仅重复索引查找本身就已经涉及更多工作量，因为数据库必须遍历两个索引树。
此外，数据库还需要大量的 CPU 和内存来合并这两个结果。

> Note: 单个索引比两个更快

数据库使用两种方式合并索引：

1. 首先是索引连接 _join_
2. 第二种方法利用了数据仓库 _data warehouse_ 领域的功能

数据库仓库是一切临时查询的鼻祖。
只需要几次点击，就能将任意条件组合成想要的查询。
无法预测 `WHERE` 子句中可能出现的组合，这使得迄今为止所描述的索引方法几乎无法实现。

数据库仓库使用一种特殊的索引类型来解决这个问题：即所谓的位图索引 _bitmap index_。
位图索引的优势在于他们可以相对容易的组合使用。
这意味着，当索引所列的使用，也可以得到不错的性能。
相反地，如果能提前知道查询语句，可以创建一个定制的多列 B 树索引，这样性能仍然会比多列的位图索引快。

目前，位图最大的弱点在于插入、更新和删除的可扩展性极差。
并发写入几乎不可行，这在数据库仓库中不是问题，因为加载过程是按顺序依次调度的。
而在线应用中，位图索引大多没有用处。

> Important: 位图索引对于在线业务处理几乎不可用

许多数据库提供了一个基于 B-tree 和 位图的混合解决方案。
在没有更好的访问路径的情况下，他们将多次 B 树扫描的结果转换为内存中的位图结构。
这样可以高效地合并。
位图接口不会持久存储，而是在语句结束都丢弃，从而绕过了写入扩展性差的问题。
缺点是需要大量内存和 CPU 时间，毕竟这是优化器走头无路时选择的方法。

## Partial Indexes

目前为止，只讨论了要在索引中增加哪些列。
通过部分索引 _partial index_ (PostgreSQL) 或筛选索引 _filtered index_ (SQL Server) 可以指定具体的行。

部分索引对于使用常量值的常用 `where` 条件非常有用，如下例中的状态码所示：

```sql
SELECT message
FROM messages
WHERE processed = 'N'
AND reciever = ?
```

在队列系统中这样的查询非常常见，查询查找所有未处理的信息给一个特定的接收者。
已经处理的信息很少会需要。
即使需要，他们通常也是通过主键这样的具体标准访问。

我们可以使用双列索引来优化该查询。
仅考虑此查询时，列是顺序并不重要，因为不存在范围条件。

```sql
CREATE INDEX messages_todo
          ON messages (receiver, processed)
```

索引实现了该目的，但其中包含了许多从未被搜索的行，即所有已处理过的消息。
由于索引的对数扩展性，即使浪费了一些磁盘空间也非常快。

通过部分索引 _partial indexing_ 可以限制并只包含未处理的消息。
语法非常简单：

```sql
CREATE_INDEX messages_todo
          ON messages (reciever)
       WHERE processed = 'N'
```

该索引只包含符合 `where` 语句的行。
在这个特定情况下，甚至可以移除 `processed` 列，因为该列的值始终为 `'N'`。
这意味着，索引在两个方面缩小了规模：
垂直方向，因为行数更少；水平方向，因为删除了那一列。

因此，索引非常小。
对于队列来说，甚至意味着尽管表格在不断增长，但索引的大小却保持不变。
这个索引并不包含所有消息，只包含尚未处理的消息。

部分索引的 `where` 子句可以变得非常复杂。
唯一的限制是关于函数的：
在索引定义里，只能用确定的函数，这和其他地方一样。
SQL Server 有着更加严格的规则，既不能在索引中使用函数也不能使用 `OR` 运算符。

只要查询里出现了 `where` 子句，数据库就能使用部分索引。

## NULL in the Oracle Database

SQL 里的 `NULL` 经常让人搞不清楚。
虽然 `NULL` 的基本意思很简单，就是表示数据缺失，但实际用起来会有些奇怪。
例如，不能直接使用 = 判断，而得用 `IS NULL`。
另外，Oracle 数据库在 `NULL` 的处理上还有一些奇怪的怪癖：
一方面是因为它没有完全按照标准来处理 `NULL`，另一方面是它在索引里对 `NULL` 的处理方式非常 “特别”。

SQL 标准并不将 `NULL` 视作一个值，而是将其视为一个缺失或未知值的占位符 _placeholder_。
因此，没有值可以是空值。
但是，Oracle 将空字符视作 `NULL`:

`NULL` 这个概念在很多编程语言中都会用到。
不管哪门语言，空字符串都不等于 `NULL`，但 Oracle 数据库是个例外。
实际上，根本无法在 VARCHAR32 字段中存储一个空字符串，Oracle 会把它变成 NULL。

该特点不仅很奇怪，也很危险。
此外，Oracle 数据库中 NULL 的古怪特点不止于此，它还会延伸到索引中。

### Indexing NULL
