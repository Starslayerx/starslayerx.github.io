+++
date = '2025-11-20T8:00:00+01:00'
draft = true
title = 'Postgresql 04: Data Type'
tags = ['PostgreSQL']
+++

这篇文章介绍 PostgreSQL 数据类型

### 数据类型的输入与转换

对于简单的数据类型，正常输入就行了

```SQL
SELECT 1, 1.142, 'hello world';
```

对于复杂的数据类型，可以按照“类型名”加上单引号括起来的类型格式来输入

```SQL
SELECT bit '11110011';
```

实际上所有的数据类型都可以使用上面的输入方法

```SQL
SELECT int '1' + int '2';
```

PostgreSQL 支持标准 SQL 的数据类型转换函数 CAST 来进行数据类型转

```SQl
SELECT CAST('5' AS int), CAST('2014-07-17' AS date);
```

还可以使用更加简洁的双冒号写法进行类型转换

```SQl
SELECT '5'::int, '2014-07-17'::date;
```

## 布尔类型

boolean 的只要么是 true 要么是 false，如果是 unknown 状态使用 NULL 表示。
boolean 可以使用不带引号的 TRUE 或者 FALSE 表示，也可以使用其他表示“真”和“假”的带引号字符表示，
如 'true'、'false'、'yes'、'no' 等，并且大小写不敏感。

布尔类型的常见逻辑操作符有：AND、OR、NOT，并使用 IS 比较运算符，例如 `expression IS TRUE`

## 数值类型

### 整数类型

整数类型有三种：smallint (2 Byte)、int (4 Byte)、bigint (8 Byte)，注意 PostgreSQL 中没有 MySQL 的 tinyint (1 Byte), mediumint (3 Byte) 两种类型。

最常用的是 int 类型，只有在取值范围不够时才使用 bigint 类型，或磁盘空间紧张的时候才使用 smallint 类型。

### 精确的小数类型

精确的小数类型可以使用 numeric、numeric(m, n)、numeric(m) 表示。

其中 numeric 类型和 decimal 类型是等效的，这两种都是 SQL 标准，可以存储最多 1000 位精度的数字，并且可以准确地进行计算。
这种类型适合于货币金额和其他要求精确计算的场合，不过计算速度对比整数和浮点数类型要慢的多。

```SQL
NUMERIC(precision, scale)
```

- 精度 precision 必须为正数：表示总共允许的十进制数字个数，包括整数部分和小数部分
- 标度 scale 可以为 0 或者正数：表示小数点后允许的数字个数

`NUMERIC(precision)` 和 `NUMERIC(precision, 0)` 是相同的。
如果不带任何精度与标度地声明 NUMERIC，则表示创建一个可以存储任意精度和标度的数值，这种类型的字段不会把输入数值转化为任何特定的标度，而带有标度声明的 NUMERIC 字段会把输入值转化为指定标度。

在标准 SQL 和 MySQL 中，语法 DECIMAL 等价于 `DECIMAL(M, 0)`，M 在 MySQL 中默认为 10，PostgreSQL 中因为作用不大而把它改成了一个任意精度和标度的值。

如果字段声明了标度，超过小数点的标度会自动4舍5入有进行存储，对于没有精度也没有标度的 number 类型来说，会原样存储。
对于声明了精度，INSERT 语句超过该精度则会报错。

### 浮点类型数据

数据类型 real 和 double precision 是不精确的、变精度的数字类型。

除了普通字符值外，浮点类型还有 `Infinty`、`-Infinty` 和 `NaN` 这些特殊值。

### 序列类型

在序列类型中，serial 和 bigserial 与 MYSQL 的自增字段含义相同。
PostgreSQL 实际上是通过序列 (sequence) 实现的，和 Oracle 一样都有序列，MySQL 则没有。

示例如下

```SQL
CREATE TABLE t {
    id SERIAL
};
```

上面语句等价于声明下面几个语句：

1. 创建序列
2. 将序列类型值设置为整数，并给队列添加默认值，默认值为序列的 `nextval()`
3. 将序列绑定到该序列，保证声明周期与列关联

```SQL
CREATE SEQUENCE t_id_seq;
CREATE TABLE t {
    id integer NOT NULL DEFAULT nextval('t_id_seq')
};
ALTER SEQUENCE t_id_seq OWNED BY t.id;
```

### 货币类型

货币类型可以存储固定小数的货币数目，不同于浮点数，它是完全保证精度的。
其输出格式与参数 lc_monetary 设置有关，不同国家输出格式也不同。

```SQL
SELECT '12.4'::money;

show lc_monetary;
set lc_monetary = 'zh_CN.UTF-8';
set lc_monetary = 'en_US.UTF-8';
```

### 数学函数和操作符

PostgreSQL 支持丰富的数学操作，其中 `|/` 代表平方根，`||/` 代表立方根。

## 字符串类型

字符串有以下主要类型

- character varying(n), varchar(n): 变长，最大 1GB
- character(n), char(n): 定长，不足补空白，最大 1GB
- text: 变长，无长度限制，类似 MySQL 中的 longtext

varchar 和 char 分别是 character varying 和 character 的别名。
未声明长度的 character 等价于 character(1)，如果使用 character varying 时不带长度说明，则可接受任意长度的字符串。

## 二进制数据

二进制类型是可以存放任意数据的一种类型，PostgreSQL 中只有一种二进制类型：bytea 类型。
这种数据类型允许存储二进制字符串，对应 MySQL 和 Oracle 中的 blobs 类型。

二进制字符串是一个字节序列。和普通字符串的区别有两个：

- 二进制字符串可以存储字节零值，普通字符串不允许字节零值，而且也不允许那些不符合字符集编码的非法字节
- 对二进制字节串的处理实际上就是对字节的处理，而对字符的处理则取决于区域设置

由于二进制字符串的部分字符为不可打印字符，因此需要使用转义一个字节值，把它的数值转换成对应的三位 8 进制数。
有些 8 进制数可以加一个反斜杠直接转义，比如单引号和反斜杠本身。

## 位串类型

位串类型可以存放一系列的二进制位类型，相对于二进制类型来说，此类型的操作更方便、更直观。

为一系列 0 和 1 组成的字符串，下面是两种 SQL 位类型：

- bit(n)
- bit varying(n)

其中，bit(n) 类型的数据必须精确匹配长度 n，存储长一些或者短一些的数据都不正确。
bit varying(n) 类型的数据是最长为 n 的变长类型，更长的串会被拒绝。
写一个没有长度的 bit 等效于 bit(1)，没有长度的 bit varying 表示没有长度限制。

如果明确地把一个位串值转换成 bit(n)，那么其右边将被截断，或者在右边补齐 0 到刚好为 n 位，且不会抛出任何错误。
类似地，如果明确地把一个位串数值转换成 bit varying(n)，而其超过 n 位，则右边将会被截断。

### 位串的使用方法

下面介绍位串的使用方法
