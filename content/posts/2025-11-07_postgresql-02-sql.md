+++
date = '2025-11-07T8:00:00+08:00'
draft = false
title = 'PostgreSQL 02: SQL Basis'
tags = ['PostgreSQL']
+++

## SQL 入门

SQL (Structured Query Language) 是关系数据库最重要的操作语言，且影响已经超出了数据库领域，这篇文章介绍基础部分。

### 语句分类

SQL 命令一般分为 DQL、DML、DDL 三类：

- DQL: Data Query Language 数据查询语句，基本就是 SELECT 查询命令，用于数据查询。

- DML: Data Manipulation Language 数据操纵语言，主要用于插入、更新、删除数据，即 INSERT、UPDATE、DELETE 三类。

- DDL: Data Definition Language 数据定义语言，用于创建、删除、修改表、索引等数据库对象的语言。

### 词法结构

每次执行的 SQL 语句可以由多条 SQL 命令组成，多条 SQL 语句命令之间由分号 (;) 分隔。

SQL 命令由一系列记号组成，这些记号可以由关键字、标识符、双引号包围的标识符、常量和单引号包围的常量组成。
SQL 命令中可以有注释，这些注释在 PostgreSQL 中等同于空白。

例如下面这些 SQL 命令

```SQL
SELECT * FROM OSDBA_TABLE01;
UPDATE OSDBA_TABLE SET COL1 = 64;
INSERT INTO OSDBA_TABLE VALUES (232, 'hello osdba');
```

#### DDL 语句

使用 `psql` 会默认连接到和用户名一样的数据库中，也可以使用命令 `psql postgres` 来连接到默认数据库，或者 `creatdb db_name` 创建新数据库。

##### 建表语句

表为关系数据库中基本对象

```SQL
CREATE TABLE tables_name (
    col01_name datatype,
    col02_name datatype,
    col03_name datatype,
    col04_name datatype,
);
```

- 变长的字符串在大多数数据库中都可以使用 `varchar` 类型，例如 PostgreSQL、MySQL 和 Oracle 数据库等。
- 整数数据在 PostgreSQL 和 MySQL 中都可以使用 `int` 类型。
- 日期类型的名称一般是 `date`

```SQL
CREATE TABLE socre (
    student_name varchar(40),
    chinese_score int,
    math_socre int,
    test_date date
);
```

创建表后使用 `\d` 显示数据库中有哪些表

```
          List of relations
 Schema | Name  | Type  |    Owner
--------+-------+-------+-------------
 public | score | table | starslayerx
(1 row)
```

使用 `\d score` 查看该表的定义情况

```
                          Table "public.score"
    Column     |         Type          | Collation | Nullable | Default
---------------+-----------------------+-----------+----------+---------
 student_name  | character varying(40) |           |          |
 chinese_score | integer               |           |          |
 math_score    | integer               |           |          |
 tset_date     | date                  |           |          |
```

其中 "character varying(40)" 等同于 "varchar(40)"，"integer" 也等同于 "int"。

建表时可以指定表的主键，在列的后面使用 `primary key` 来指定主键。

```SQL
CREATAE TABLE student (
    no int primary key,
    student_name varchar(40),
    age int
);
```

##### 删除表语句

删除语法比较简单

```SQL
DROP TABLE table_name;
```

#### DML 语句

DML 用于插入删除数据，主要包括 INSERT 语句、UPDATE 语句、DELETE 语句。

##### 插入语句

使用下面语句向前面建立的 student 表插入数据

```SQL
INSERT INTO student VALUES(1, '张三', 14);
```

也可以指定列名插入

```SQL
INSERT INTO student(no, age, student_name) VALUES(2, 13, '李四');
```

也可以不插入某些列数据，这些列会被置空

```SQL
INSERT INTO student(no, student_name) VALUES(3, '王二');
```

##### 更新语句

假设要将所有学生年龄更新到 15，则更新语句如下：

```SQL
UPDATE student SET age = 15;
```

更新数据库使用 WHERE 来指定更新哪条或哪些数据

```SQL
UPDATE student SET age = 15 WHERE no = 3;
```

##### 删除语句

例如删除学号 no 为 3 的学生记录

```SQL
DELETE FROM student WHERE no = 3;
```

若要删除表中所有数据

```SQL
DELETE FROM student;
```

##### 单表查询语句

查询 student 表中所有数据

```SQL
SELECT no, student_name, age FROM student;
```

列可以是表的列名，也可以是另一个表达式：

```SQL
SELECT age+5, FROM student;
```

还可以与列表无关

```SQL
SELECT no, 3+5 FROM student;
```

实际上，与表达式无关的时候，可以不 FROM 表格，这样可以当作计数器使用：

```SQL
SELECT 2+3;

 ?column?
----------
       5
(1 row)
```

使用 \* 代表所有的列

```SQL
SELECT * FROM student;
```

### 过滤条件的查询

SELECT 后面添加 WHERE 用于指定查询过哪些记录，例如查询学号为 3 个学生记录

```SQL
SELELCT * FROM student WHERE no=3;
```

### 排序

使用 ORDER BY 语句对查询结果进行排序，例如对年龄排序：

```SQL
SELECT * FROM student ORDER BY age;
```

ORDER BY 需要在 WHERE 语句之后，若顺序不对会报错。

```SQL
SELECT * FROM student WHERE age >= 15 ORDER BY age;
```

还可以对多个查询结果排序

```SQL
SELECT * FROM student ORDER BY age, student_name;
```

后面加上 DESC 开逆序

```SQL
SELECT * FROM student ORDER BY age DESC;
SELECT * FROM student ORDER BY age DESC, student_name;
```

### 分组查询

例如要统计不同年龄段的学生人数，可以使用分组查询

```SQL
# SELECT age, count(*) FROM student GROUP BY age;

age | count
-----+-------
15 | 2
14 | 1
```

注意，使用 GROUP BY 语句时需要使用聚合函数，常见的有 count、sum 等

### 多表关联查询

多表关联查询也称表 join，例如有一张 class 班级表，建表语句如下

```SQL
CREATE TABLE class(no int primary key, class_name varchar(40));
```

插入一些测试数据

```SQL
# INSERT INTO class VALUES(1, '初二（1）班');
# INSERT INTO class VALUES(2, '初二（2）班');
# INSERT INTO class VALUES(3, '初二（3）班');
# INSERT INTO class VALUES(4, '初二（4）班');

# SELECT * FROM class;
 no | class_name
----+-------------
  1 | 初二（1）班
  2 | 初二（2）班
  3 | 初二（3）班
  4 | 初二（4）班
(4 rows)
```

还有一张学生表 student，建表语句如下

```SQL
CREATE TABLE student(no int primary key, student_name varchar(40), age int, class_no int);
```

同样插入一些数据

```
# SELECT * FROM student;
 no | student_name | age | class_no
----+--------------+-----+----------
  1 | 张三         |  14 |        1
  2 | 吴二         |  15 |        1
  3 | 李四         |  13 |        2
  4 | 吴三         |  15 |        2
  5 | 王二         |  15 |        3
  6 | 李三         |  14 |        3
  7 | 吴二         |  14 |        3
  8 | 张四         |  14 |        4
```

假如想要查询每个学生名字与班级名称之间的关系，就需要关联查询两张表

```SQL
# SELECT student_name, class_name FROM student, class
  WHERE student.class_no = class.no;

 student_name | class_name
--------------+-------------
 张三         | 初二（1）班
 吴二         | 初二（1）班
 李四         | 初二（2）班
 吴三         | 初二（2）班
 王二         | 初二（3）班
 李三         | 初二（3）班
 吴二         | 初二（3）班
 张四         | 初二（4）班
(8 rows)
```

关联查询就是在 WHERE 子句中加上需要关联的条件 `WHERE student.class_no = class.no;`

由于两张表中有些列的名称相同，例如 student 中的 no 是学生编号，class 中的 no 是班级编号，所以关键条件中要明确使用 `表名.列名` 来明确唯一定位某一列

```SQL
SELECT student_name, class_name FROM student a, class b
WHERE a.class_no = b.no;
```

还可以在关联查询的 WHERE 子句中加上其他过滤条件

```SQL
SELECT student_name, class_name FROM student a, class b
WHERE a.class_no = b.no
AND age > 14;
```

### 子查询

当一个查询是另一个查询的条件时，称为子查询。

主要有 4 种语法的子查询：

- 带有谓词 IN 的子查询: `expression [NOT] in (sqlstatement)`
- 带有谓词 EXISTS 的子查询: `[NOT] EXISTS (sqlstatement)`
- 带有比较运算符的子查询: `comparsion(>,<,=,!=) (sqlstatement)`
- 带有 ANY (SOME) 或 ALL 谓词的子查询: `comparsion [ANY|ALL|SOME] (sqlstatement)`

下面仍然使用 class 和 student 两个表来做示例

- 使用带有谓词 IN 的子查询来查询 “初二（1）班” 的学生记录

  ```SQL
  SELECT * FROM student
  WHRER class_no IN (
      SELECT no FROM class
      WHERE class_name='初二（1）班'
  );
  ```

  查询结果如下

  ```SQL
  no | student_name | age | class_no
  ----+--------------+-----+----------
  1 | 张三 | 14 | 1
  2 | 吴二 | 15 | 1
  (2 rows)
  ```

- 同样可以使用带 EXISTS 谓词的子查询实现

  ```SQL
  SELECT * FROM student s
  WHERE EXISTS (
    SELECT 1 FROM class c
    WHERE s.class_no=c.no
    AND c.class_name = '初二（1）班'
  );
  ```

EXISTS 是一个存在性测试，它检查子查询是否返回至少一行数据，执行逻辑如下：

1. 对于每个学生记录，数据库会执行一次子查询
2. 子查询检查：class 表中查找匹配 WHERE 规则的记录
3. 返回判断：如果查询找到匹配，EXISTS 为 True，则结果包含该学生，否则为 False 不包含

上面查询使用 SELECT 1 是因为只关系是否有返回值，而不关系返回值的类型

- 还可以使用带有比较符的子查询实现

  ```SQL
  SELECT * FROM student
  WHERE class_no = (
      SELECT no FROM calss c
      WHERE class_name = '初二（1）班'
  );
  ```

- 还可以使用带 ANY (SOME) 或 ANY 谓词的子查询来实现

  ```SQL
  SELECT * FROM student
  WHERE class_no = ANY(
    SELECT no FROM class c
    WHERE class_name = '初二（1）班'
  );
  ```

- 如果要查询*两个*班级的学生记录，则不能使用带有等于 “=” 比较符的子查询

  ```SQL
  SELECT * FROM student
  WHERE no = (
      SELECT no FROM class c
      WHERE class_name in ('初二（1）班', '初二（2）班')
  );

  ERROR:  more than one row returned by a subquery used as an expression
  ```

  上面报错说子查询不能返回多行，这种操作就是在说“找出学号等于（初二1班或初二2班的班级号）的学生“，但是等于号不能等于两个东西，只能等于一个东西，逻辑就错误了。

  这种不能返回多行的子查询也叫*标量子查询*，标量子查询不仅能嵌套在 WHERE 子句中，也可以嵌套在 SELECT 的列表中。

  如果要查询每个班级学生的最大年龄，可以使用下面 SQL 语句：

  ```SQL
  SELECT no, class_name, (
      SELECT max(age) AS max_age
      FROM student s WHERE s.no = c.no
  ) AS max_age FROM class c;
  ```

  查询两个班级学生信息的时候，用带有 ANY (SOME) 谓词的子查询就没问题了，AYN 就是说只要等于里面任何*一个*就行。

  ```SQL
  SELECT * FROM student
  WHERE class_no = ANY(
      SELECT no FROM class c
      WHERE calss_name in ('初一（1）班', '初二（2）班')
  );
  ```

## 其他 SQL 语句

### INSERT ... SELECT 语句

使用 INSERT ... SELECT 语句可以将一张表中的数据插入另一张表中，属于 DML 语句。

假设创建一张学生表的备用表 student_bak，建表语句如下：

```SQL
CREATE TABLE student_bak(
    no int primary key,
    student_name varchar(40),
    age int,
    class_no int
);
```

可以使用下面语句将 student 备份到备份表中

```SQl
INSERT INTO student_bak SELECT * FROM student;
```

### UNION 语句

使用 UNION 语句可以把从两张表中查询出来的数据合在一个结果集下

```SQL
SELECT * FROM student WHERE no = 1
UNION
SELECT * FROM student_bak WHERE no = 2;
```

要注意的是，UNION 语句会将结果相同的两条记录合并成一条。
如果不想合并相同记录，可以使用 UNION ALL 语句。

```SQl
SELECT * FROM student WHERE no = 1
UNION ALL
SELECT * FROM student_bak WHERE no = 1;
```

### TRUNCATE TABLE 语句

TRUNCATE TABLE 语句作用是清空表内容，不含 WHERE 条件的 DELETE 语句（`DELETE FROM table_name`）也可以情况表内容，但两者实现原理不同。

- TRUNCATE TABLE 是 DDL 数据定义语句，相当于重新定义一个新的表的方法，把原来表内容直接丢弃了，执行速度快
- DELETE FROM 是 DML 数据操作语句，可以认为是一行一行地删除，执行速度较慢

## Summary 总结

从上面内容可以看出来，SQL是一种声明式编程语言，与命令式编程语言有较大的差异。
声明式编程语言主要是描述用户需要做什么，需要得到什么结果，不像命令式编辑语言需要描述怎么做，过程是什么。
SQL语句能智能地实现用户的需要，而不需要用户关心具体的运行过程。
