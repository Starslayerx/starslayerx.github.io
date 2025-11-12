+++
date = '2025-11-07T8:00:00+08:00'
draft = true
title = 'PostgreSQL 02: SQL Basis'
tags = ['PostgreSQL']
+++

## SQL 入门

SQL (Structured Query Language) 是关系数据库最重要的操作语言，且影响已经超出了数据库领域，这篇文章介绍基础部分。

### 语句分类

SQL 命令一般分为 DQL、DML、DDL 三类：

- DQL: 数据查询语句，基本就是 SELECT 查询命令，用于数据查询。

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
