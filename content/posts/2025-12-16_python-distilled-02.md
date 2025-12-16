+++
date = '2025-12-15T8:00:00+08:00'
draft = true
title = 'Python Tricks Part 2: Operators, Expressions and Data Manipulation'
tags = ['Python']
+++

## Literals

整数

```Python
04
0b101010 # Binary 二进制
0o52     # Octal  八进制
0x2a     # Hexadecimal 十六进制
```

浮点数，内部使用 IEEE 754 双精度存储

```Python
4.2
42.
4.2e+2 # 科学记数法
4.2E2
-4.2e-2
```

数字类型字面量还可以使用 `_` 来方便阅读

```Python
123_456_789
```

## Truth Values

- `true`: 非0数字、任何非空的字符串，列表，元组或字典
- `false`: 0、None、空列表，元组或字典

## Operations Involving Iterables

TODO: 消耗性迭代器
