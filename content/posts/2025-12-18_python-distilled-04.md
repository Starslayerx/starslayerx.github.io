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

一个 subtype 子类通过 inheritance 定义，其携带有原始类的所有特性，可以增加或重定义方法。
`isinstance(instance, type)` 函数是检查值与类型是否匹配的首选方法，因为它能够识别子类型。
该函数还可以同时检查多种可能的类型。

```Python
class mylist(list):
    pass

if isinstance(items, (list, tuple)):
    maxval = max(items)
```

尽管类型检查可以加入程序中，但类型检查往往没有想象中的那么好用。
首先，过度的类型检查很影响性能。
其次，程序并不总是定义成完美契合于一个良好的类型层次结构，比如 `isinstance(items, list)` 无法检测一个非继承自 list 但类 list 的类型。

## Reference Counting and Garbage Collection

Python 通过自动垃圾回收机制管理对象，所有对象都通过引用计数 reference counting。
每当一个对象被分配一个新的名称，或放入一个新容器中后，就会增加引用计数，如下：

```python
a = 37  # 创建对象 37
b = a   # 增加引用计数
c = []
c.append(b)  # 增加引用计数
```

对象的引用计数会因 `del` 语句或引用超出作用域而减少。
可以使用 `sys.getrefcount()` 获取一个对象的引用计数：

```Python
import sys

a = 37
sys.getrefocunt(a)
```

引用计数可能会比预计的大，这是因为解释器会在不同地方共享这种变量，以节约内存。
由于这些变量不可变，因此往往不会被注意到。

> 注意

从 Python 3.12 [PEP 683](https://peps.python.org/pep-0683/) 开始引入了 Immoral Objects 永生对象，对于这类对象，其内部运行时状态将不再改变，引用计数永远不会减到 0，除了在解释器终止的时候。

使用新版对小数使用这个方法会得到一个非常大的值（2 的 32 次方减 1）。

```bash
% python -c "import sys; print(sys.getrefcount(37))"
4294967295 #  2^(32) - 1
```

当对象的引用计数到 0 后会被垃圾回收。
然而，在某些情况下，一组不再使用的对象之间可能存在循环依赖。

```Python
a = { }
b = { }
a['b'] = b
b['a'] = a
del a
del b
```

在这个例子中，del 语句将 a 和 b 的引用计数减少，并摧毁引用的名称。
但由于互相引用，引用计数无法变成 0，也就无法释放。
最终，解释器不会发生内存泄露，但对象的消毁会延迟，直到循环检测器执行后找到并删除那些无法访问的对象。

随着解释器在执行过程中分配越来越多的内存，循环检测算法会定期运行。
具体的行为可以使用 `gc` 标准库来微调和控制，`gc.collect()` 函数可以立即触发循环垃圾回收器。

在使用巨大的数据结构的时候，手动删除对象是合理的。

```Python
def some_calculation():
    data = create_giant_data_structure()
    # Use data for some part of a calculation
    ...

    # Release the data
    del data

    # Calculation continues
    ...
```

## References and Copies

等于号会创建一份引用，而不是复制一份对象

```Python
a = [1, 2, 3]
b = a
b is a  # True
b[1] = -2
a  # [1, -2, 3]
```

上面例子中，a 和 b 都指向同一个对象，因此修改 b 会导致 a 也被修改。

如果不希望拷贝引用，而是真实的赋值，有两种方法，浅拷贝和深拷贝。

```Python
a = [1, 2, [3, 4]]
b = list(a)  # 创建一份浅拷贝
b is a   # False
b.append(100)
b # [1, 2, [3, 4], 100]
a # [1, 2, [3, 4]]

b[2][0] = -100
b # [1, 2, [-100, 4]]
a # [1, 2, [-100, 4]]
```

上面例子中，b 修改外层变量，a 没有改变，但修改内层变量 a 被改变了。

这是因为浅拷贝只会复制最外层的变量，对于嵌套的变量不会拷贝内层变量，使用深拷贝才会循环拷贝嵌套变量。
并没有可以进行深拷贝的操作符，但是开使用 `copy.deepcopy()` 来实现。

```Python
import copy

a = [1, 2, [3, 4]]
b = copy.deepcopy(a)

b[2][0] = -100
b # [1, 2, [-100, 4]]
a # [1, 2, [3, 4]]
```

不要在非必要的地方使用 deepcopy，因为速度很慢，同时要注意深拷贝无法拷贝系统或运行时状态（例如打开文件、网络连接、线程、生成器等）。

## Object Representation and Printing
