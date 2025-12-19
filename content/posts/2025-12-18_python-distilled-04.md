+++
date = '2025-12-18T8:00:00+08:00'
draft = true
title = 'Python Tricks Part 4: Objects, Types and Protocols'
tags = ['Python']
+++

## Essential Concepts

每个存储在程序中的数据都是一个对象，每个对象都有一个 identity, type and value （身份、类型和值）。
例如 `a = 42` 会创建一个整数为 42 的类型，该对象的 identity 是内存中的一个数字，代表其在内存中的位置，`a` 是这个类的标签，指向这个特定的内存位置，标签本身并非对象的一部分。

object 对象的类型，也被称为 class 类，该类定义了对象的内部数据表示，和支持的方法。
当特定类型的对象创建后，该对象被称为该类的 instance “实例”。
当实例创建后，其 identity 就不会改变。
如果一个对象的值可以被修改，则该对象是可变的 mutable。
如果一个对象的值不可以被修改，则该对象是不可变的 unmutable。
一个持有对其他对象引用的对象，被称为容器。

对象通过其属性来表征，属性是于对象关联的值，通过点 `.` 运算符来访问。
属性可以是一个简单的值，例如一个数字，也可以是一个被调用以执行某些操作的函数。

这类函数被称为方法，例如下面这个例子

```Python
a = 34  # Create an integer
n = a.number   # Get the numberator (an attribute)

b = [1, 2, 3]  # Create a list
b.append(4)    # Add a new element using the append method
```

## Object Identity and Type

内置函数 `id()` 返回对象的 identity，该 identity 是一个整数，通常对应 object 的内存地址。
`is` 操作符会对比两个对象的 identity，`type()` 返回对象的类型。

下面是一个比较两个类的例子：

```Python
def compare(a, b):
    if a is b:
        print('same object')
    if a == b:
        print('same value')
    if type(a) == type(b):
        print('same type')
```

对象的类型本身也是一个对象，称为对象的类。
这个对象是唯一确定的，对于给定类型的所有实例来说始终相同。
类通常有名称，这些名称可以用于创建实例（如 list, int, dict），执行类型检查以及进行类型提示。

例如：

```Python
items = list()

def add_item(items: list, item):
    if isinstance(items, list):
        items.append(item)

def remove_item(items: list, item) -> list:
    return [i for i in items if i != item]
```

一个 subtype 子类通过 inheritance 定义，其携带有原始类的所有特性，增加或重定义方法。
`isinstance(instance, type)` 函数是检查值与类型是否匹配的首选方法，因为它能够识别子类型，该函数还可以同时检查多种可能的类型。

```Python
class mylist(list):
    pass

if isinstance(items, (list, tuple)):
    maxval = max(items)
```
