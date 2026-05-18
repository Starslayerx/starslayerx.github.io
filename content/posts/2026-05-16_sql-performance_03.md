+++
date = '2026-05-16T8:00:00+01:00'
draft = true
title = 'SQL Performance Explained: Execution Plans'
categories = ['Note']
tags = ['SQL']
+++

## PostgreSQL

### Getting an Execution Plan

PostgreSQL 的执行计划是通过在 SQL 语句前面加一上 `EXPLAIN` 命令来获取的。
不过有个重要的限制：带有绑定参数的 SQL 语句 (例如 $1 $2) 不能直接解释，他们需要准备好才行。

```sql
PREPARE stmt(int) AS
SELECT $1;
```

注意 PostgreSQL 使用 `$n` 作为绑定参数。
数据库抽象层可能会把这个隐藏起来，这样就可以使用 SQL 标准里定义的问号 `?` 了。

执行准备好的语句是可以解释的：

```sql
EXPLAIN EXECUTE stmt(1);
```

在 PostgreSQL 9.1 及之前的版本中，执行计划在调用 `PREPARE` 时就已经创建好了。所以没法考虑执行时传入的具体参数值。
而从 PostgreSQL 9.2 开始，执行计划的创建被推迟到实际执行的时候，这样就可以根据绑定参数的具体值来生成计划了。

没有绑定参数的语句可以直接分析：

```sql
EXPLAIN SELECT 1;
```

在这种情况下，优化器总是会在查询计划中考虑实际值。
执行计划如下：

```text
               QUERY PLAN
------------------------------------------
 Result (cost=0.00..0.01 rows=1 width=0)
```

输出结果包含的信息与之前展示过的 Oracle 执行计划类似：操作名称 _Result_、相关成本 _cost_、行数估计 _row_ 以及预期行宽 _width_。
注意这里有两个成本信息，第一个是启动成本 _startup_，第二个是如果获取所有行时的执行总成本。
Oracle 数据库的执行计划只显示第二个值。

PostgreSQL `EXPLAIN` 命令有两个选项。
`VERBOSE` 选项会提供额外的信息，例如完整的表名 _fully qualitied table names_ `VERBOSE` 并不是特别有用。

第二个 `EXPLAIN` 选项是 `ANALYZE`。
虽然使用这个东西的人很多，但不建议直接使用，因为它会真的执行这条语句。
这对查询语句没什么影响，但是用于插入、更新或删除时，会修改数据。
为避免意外修改数据的风险，可以将其放入一个事务中，然后执行回滚操作。

```sql
BEGIN;
EXPLAIN ANALYZE EXECUTE stmt(1);

                    QUERY PLAN
--------------------------------------------------
 Result  (cost=0.00..0.01 rows=1 width=0) (actual time=0.002..0.002 rows=1 loops=1)
 Total runtime: 0.020 ms

ROLLBACK;
```

行数是唯一一个在估计值和实际值两部分都显示的树枝。
这样能快速找出错误的基数估计。

最后，准备好的语句需要再次关闭：

```sql
DEALLOCATE stmt;
```

### Operations

#### Index and Table Access

- **Seq Scan**

`Seq Scan` (Sequential Scan) 顺序扫描操作会扫描整个关系表，类似于全盘扫描 _FULL TABLE ACCESS_。

- **Index Scan**

`Index Scan` 索引扫描会执行 B 树遍历，遍历叶节点找到匹配的项目，并获取对应的表数据。
它类似 `INDEX RANGE SCAN` 后面跟着 `TABLE ACCESS BY INDEX ROWID` 操作。

- **Index Only Scan**

`Index Only Scan` 会执行 B 树遍历，并遍历所有叶子节点找到所有匹配项。
但不需要去访问表，因为索引包含了所有满足查询的列。

- **Bitmap Index Scan/ Bitmap Heap Scan/ Recheck cond**

Tom Lane 在 PostgreSQL 性能右键列表里发的帖子说的非常明白：

普通的索引扫描是一次从索引里取出一个元组指针，然后立刻去表里访问那个元组。
而位图扫描是一次性从索引里取出所有元组指针，内存里的位图 _bitmap_ 数据结构把它们排好序，
然后按照元组的物理存储位置顺序去访问表里的元组。

#### Join Operations

通常 `JOIN` 操作只会一次处理两个表。
当一个查询有多个 `JOIN` 的时候，它们会按顺序执行：
前两个表先合并，然后中间结果 _intermediate result_ 和下一个表再合并。
对于 `JOIN` 而言，表也意味着中间结果。

- **Nested Loops**

通过从一个表中获取数据，再针对第一个表的每一行去查询另一个表，从而将两个表连接起来。

- **Hash Join / Hash**

Hash Join 会将一侧的侯选记录加载到哈希表中，然后针对连接另一侧的每条记录来探测该哈希表。

- **Merge Join**

Merge (sort) Join 就像拉链一样，把两个已经排列好的列表合并在一起。
连接的两边都必须预先排好序。

#### Sorting and Grouping

- **Sort / Sort Key**

根据排序键中指定的数据集进行排序，排序操作要大量内存来生成中间结果。

- **Group Aggregate**

根据分组子句对预排序的数据进行聚合，该操作不会缓冲大量数据。

- **Hash Aggregate**

使用一个临时的哈希表对记录进行分组。
哈希聚合操作不需要预先排序的数据集，而是会占用大量内存来生成中间结果。

#### Top-N Queries

- **LIMIT**

当抓取到需要的行数时，就停止底层操作。

top-N 查询的效率取决于底层操作的执行模式，当中止流水线操作时，效率会非常低。

- **Window Agg**

表示使用了窗口函数

### Distinguishing Access and Filter Predicates

PostgreSQL 数据库使用三种不同的方法来应用 `where` 子句：

- **Aceess Predicate** ("Index Cond")

访问谓词 _Accesss Predicate_ 说明了遍历叶子节点的开始和停止条件

- **Index Filter Predicate** ("Index Cond")

索引过滤谓词 _Index Filter Predicate_ 仅在叶节点遍历其间应用。
他们不参与起始和中止条件的设定，也不会缩小扫描范围。

- **Table Level Filter Predicate** ("Filter")

对于不在索引里的列上的查询条件，是在表级别进行判断的。
要做到这一点，数据库必须先从那堆表里把那一行数据读出来。

---

PostgreSQL 的执行计划不会单独展示索引访问 _index access_ 和过滤谓词 _filter predicates_，这两个都通过 "Index Cond" 展示。
这意味着，必须把执行计划 _execution plan_ 和索引定义 _index definition_ 做对比，才能区分开访问谓词 _access predicate_ 和索引过滤谓词 _index filter predicates_。

> Note: PostgreSQL 的执行计划提供的信息不够，没法找出索引过滤条件。

显示为筛选 _Filter_ 的谓词始终是表级别的筛选谓词，即使在显示用于索引扫描操作时也是如此。

考虑下面这个示例：

```sql
CREATE TABLE scale_data (
    section NUMBERIC NOT NULL,
    id1     NUMBERIC NOT NULL,
    id2     NUMBERIC NOT NULL,
);

CREATE INDEX scale_data_key ON scale_data(section, id1);
```

下面的筛选条件用在了 ID2 这一列上，但这一列不在索引里。

```sql
PREPARE stmt(int) AS
SELECT count(*)
  FROM scale_acta
 WHERE section = 1
   AND id2 = $1;


                     QUERY PLAN
-----------------------------------------------------
Aggregate (cost=529346.31..529346.32 rows=1 width=0)
  Output: count(*)
  -> Index Scan using scale_data_key on scale_data
     (cost=0.00..529338.83 rows=2989 width=0)
     Index Cond: (scale_data.section = 1::numeric)
     Filter: (scale_data.id2 = ($1)::numeric)
```

ID2 谓词会在索引扫描下方显示为 "Filter"。
这样因为 PostgreSQL 在进行索引扫描时，也顺便访问了表。
换句话说，Oracle 数据库里的 "TABLE ACCESS BY INDEX ROWID" 一操作，被隐藏在了索引扫描操作中。
因此，索引扫描有可能对不在索引中的列也进行过滤。

> Imoprtant: `Filter` 谓词是表级别的过滤条件，即便在索引扫描 *Index Scan* 中显示时也是如此。

只要查询条件里包含的列，全部都被覆盖到了这个新的复合索引中。
那么在 `EXPLAIN` 的输出中，原先的 `Filter` 这一行就会彻底消失，所有的条件都会被合并到 `Index Cond` 里面显示：

```sql
CREATE INDEX scale_slow
          ON scale_data (section, id1, id2);

                     QUERY PLAN
------------------------------------------------------
Aggregate (cost=14215.98..14215.99 rows=1 width=0)
  Output: count(*)
  -> Index Scan using scale_slow on scale_data
     (cost=0.00..14208.51 rows=2989 width=0)
     Index Cond: (section = 1::numeric AND id2 = ($1)::numeric)
```

ID2 上的条件没法缩小叶子节点的遍历范围，因为索引里 ID1 这一列排在 ID2 前面。
也就是说，索引扫描会针对条件 `SECTION=1::numeric` 扫遍整个范围，然后对每一条满足 `SECTION` 条件的记录，再套上过滤器 `ID2=($1)::numeric`。
