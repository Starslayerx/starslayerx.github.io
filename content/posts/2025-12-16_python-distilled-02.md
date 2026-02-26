+++
date = '2025-12-16T8:00:00+08:00'
draft = false
title = 'Python Tricks Part 2: Operators, Expressions and Data Manipulation'
categories = ['Note']
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

任何可迭代对象都可以展开，如 list, tuple and set，都通过星号(\*)。

```Python
items = [1, 2, 3]
a = [10, *items, 1]  # [10, 1, 2, 3, 1]
b = (*items, 10, *items)  # [1, 2, 3, 10, 1, 2, 3]
c = {10, 11, *items}  # {10, 11, 1, 2, 3}
```

在上面例子中，item 简单的被粘贴到 list, tuple, set 中，就和手动输入进去一样。
有时候这种展开 expansion 被称为“展开操作符” splatting。

在定义字面量的时候，可以展开任意多的可迭代对象。
但要注意的是，许多可迭代对象只能迭代一次。
如果你使用星号 \* 将其展开，其内容会被消耗掉，且该可迭代对象在后续迭代中将不再产生任何值。

## Operations on Sequences

序列 sequence 是有大小的可迭代对象，并且允许通过从 0 开始的索引访问。
例如 strings, lists and tuples.

加号运算符 `+` 将两个同类型的 sequence 拼接起来。
乘法运算符 `*` 将序列复制 n 份，但这里实际上是浅拷贝的引用，而不是复制内存元素的值。

如果不希望使用引用，可以使用 `list()`

```Python
a = [3, 4, 5]
c = [list(a) for _ in range(4)]  # list() makes a copy of a list
```

## Operations on Mutable Sequences

切片赋值是将 `[i:j]` 中 `i ≤ k < j` 的元素替换为新列表，而不是一一对应才能替换，例如：

```Python
a = [1, 2, 3, 4, 5]
a[1] = 6              # [1, 6, 3, 4, 5]
a[2:4] = [10, 11]     # [1, 6, 10, 11, 5]
a[3:4] = [-1, -2, -3] # [1, 6, 10, -1, -2, -3, 5]
a[2:] = [0]           # [1, 6, 0]
```

切片还可能有步幅 stride 参数，这时候才需要在切片赋值列表时一一对应

```Python
a = [1, 2, 3, 4, 5]
a[1::2] = [10, 11]     # [1, 10, 3, 11, 5]
a[1::2] = [30, 40, 50] # ValueError
```

## Operations on Mappings

映射 mappings 是一种相关联的键值对关系，内置的 dict 类型就是一个例子。
当使用元组 tupel 作为键是，可以省略圆括号，并使用逗号分开值

```Python
d = { }
d[1, 2, 3] = "foo"
d[1, 0, 3] = "bar"
```

上面代码等同于

```Python
d[(1, 2, 3)] = "foo"
d[(1, 0, 3)] = "bar"
```

使用元组作为键是映射中创建符合键的常用技巧。

## Generator Expressions

生成器表达式是一种对象，有着和列表推导式一样的计算方法，但迭代产生值。
语法和列表推导式相同，编写时使用圆括号 parentheses 而不是方括号 square brackets。

生成器只能迭代一次，如果尝试迭代第二次，什么都不会得到。
但是生成器可以被转换为列表

```Python
clist = list(comments)
```

当我们将生成器表达式作为单个函数参数传递的时候，可以去掉外层的括号

```Python
sum((x*x for x in values))
sum(x*x for x in values)  # Extra parens removed
```

## The Attribute (.) Operator

操作符点 `.` 用于访问一个对象的属性

```Python
foo.x = 3
print(foo.y)
a = foo.bar(3, 4, 5)
```

一个表达式中可以使用多个点操作符，但从风格上来讲，一般不会创建很长的这种写法。

```Python
foo.bar(3, 4, 5).spam
```

## The Function Call() Operator

`f(args)` 这样写是对函数 `f` 进行调用，函数的每个参数都是一个表达式。
在调用函数前，所有表达式都会从左到右调用求值。
这有时被称为应用序求值 application order evaluation。
