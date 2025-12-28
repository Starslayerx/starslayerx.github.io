+++
date = '2025-12-18T8:00:00+08:00'
draft = false
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

程序通常需要展示对象，可能是查看其数据，或者是为了调试。

使用 `print()` 或 `str()` 会得到一个对人类可读的字符串

```Python
from datetime import date
d = data(2012, 12, 21)
print(d) # 2012-12-21
str(d)   # '2012-12-21'
```

但这种并不适合调试，比如你不知道输出的到底是 date 类型还是 str 类型，如果要想更加便于调试，应该使用 `repr()`。

```Python
d = date(2012, 12, 21)
repr(d)  # 'datetime.date(2012, 12, 21)'
print(repr(d))  # datetime.date(2012, 12, 21)
print(f'The date is: {d!r}')  # The date is datetime.date(2012, 12, 21)
```

## First Class Objects

在 Python 中所有对象都被称为 fisrt-class “一等公民”，这意味着所有对象都可以被视为数据。
作为数据，对象可以被当成变量存储、传参、作为函数返回值、和其他对象比较等。

例如，下面有两个值的字典：

```Python
items = {
    'number': 42,
    'text': 'Hello World'
}
```

通过向字典添加一些不寻常的值，可以体现 fisrt-class 一等公民的的特性。

例如：

```Python
items["func"] = abs  # Add the abs() function

import math
items["mod"] = math  # Add a module
items["error"] = ValueError  # Add an exception object
nums = [1, 2, 3, 4]
items["append"] = nums.append # Add a method of another object
```

在这个例子中，item 字典包含一个函数、一个模块、一个异常和一个其他对象的方法。
如果你愿意，可以使用字典查找来替代原始名称，代码依然可以正常运行。

```Python
items["func"](-45)   # 计算 abs(-45)
items["mod"].sqrt(4) # math.sqrt(4)

try:
    x = int("a lot")
except items["error"] as e:  # ValueError
    print("Couldn't convert")

items["append"](100)  # nums.append(100)
```

由于在 Python 中一切都是一等公民，因此可以写出非常灵活的代码：

```Python
line = "ACME,100,490.10"
column_types = [str, int, float]
parts = line.split(",")
row = [ty(val) for ty, val in zip(column_types, parts)]
# zip(*iterables, strict=True) 将多元素按位置组合，多的元素默认丢弃
```

将函数或类放在字典中是一直常见的技巧，用于取代复杂的 `if-else-elif` 语句，例如：

```Python
if format == "text":
    formatter = TextFormatter()
elif format == "csv":
    formatter = CSVFormatter()
elif format == "html":
    formatter = HTMLFormatter()
else:
    raise RuntimeError("Bad format")
```

上面代码可以这样重写

```Python
_formats = {
    "text": TextFormatter,
    "csv": CSVFormatter,
    "html": HTMLFormatter,
}

if format in _formats:
    formatter = _formats[format]()
else:
    raise RuntimeError("Bad format")
```

## Using None of Optional or Missing Data

有时候程序需要表示一个确实值或可选值，`None` 是为这一目的而保留的实例。
不返回值的函数会返回 None，可选的函数参数也使用 None。
None 没有任何属性，且转化为布尔值是 `False`，在内出使用单例模式 singleton 存储，在解释器内部只有一个值。

使用下面方法判断是不是 None:

```Python
if value is None:
    statements
    ...
```

使用 `==` 也能判断是否为 None，但不推荐这样，代码检查工具可能将其标注为 "style error"。

## Object Protocols and Data Abstraction

Python 语言的特性都由 protocols “协议”定义，例如下面函数：

```Python
def compute_cost(unit_price, num_units):
    return unit_price * num_units
```

这个函数看上去好像是接受两个整数或浮点数，但实际上他们可以接受更多的类型

```Python
from fractions import Fraction
compute_cost(Fraction(5, 4), 50)  # Fraction(125, 2)

from decimal import Decimal
compute_cost(Decimal("1.25"), Decimal("50"))  # Decimal("62.50")

import numpy as np
prices = np.array([1.25, 2.10, 3.05])
units = np.array([50, 20, 25])
compute_cost([prices, quantities])
# array([60.5, 42., 76.25])
```

甚至以“非预期”的方式工作：

```Python
compute_cost("a lot", 10)
# a lot a lot a lot a lot a lot a lot a lot a lot a lot a lot
```

但不能将一些特定类型组合一起

```Python
compute_cost(Fraction(5, 4), Decimal("50"))
"""
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 2, in compute_cost
TypeError: unsupported operand type(s) for *: 'Fraction' and 'decimal.Decimal'
"
```

### Object Protocol

下面表的内容设计对象的整体管理，包括 creation 创建、initialization 初始化、destruction 消毁和 representation 表示。

| 方法                                  | 描述             |
| :------------------------------------ | :--------------- |
| `__new__(cls, [,*args [, **kwargs]])` | 创建新实例的方法 |
| `__init__(self [,*args [,**kwargs]])` | 创建后初始化实例 |
| `__del__(self)`                       | 消毁时调用       |
| `__repr__(self)`                      | 创建字符串表示   |

创建实例的过程会被转化成这样：

```Python
x = SomeClass.__new__(SomeClass, args)
if isinstance(x, SomeClass):
    x.__init__(args)
```

使用 `__new__()` 通常意味着涉及与实例创建相关的高级技巧。
例如，在希望绕过 `__init__()` 的类方法中，或在某些创建型设计模式中使用（如定义单例或实现缓存）。
`__new__()` 方法的实现并不一定需要返回当前类的实例，如果不返回，则在创建时后续对 `__init__()` 的调用会被跳过。

当一个对象被垃圾回收的时候，会调用 `__del__()` 方法。
此方法只有在实例不再被使用后才会调用，要知道的是 `del` 方法只会减少对象的引用计数。
`__del__()` 几乎永远不会定义，除非一个实例需要额外的资源管理或消毁。

`__repr__()` 方法由函数 `repr()` 调用，会创建一个表示该对象的字符串，这对调试很有帮助。
在交互式环境中的输出就是这个方法实现的，如果要重新创建该对象，使用 `eval()`。

```Python
a = [1, 2, 3, 4, 5]  # Create a list
s = repr(a)  # s = '[1, 2, 3, 4, 5]'
b = eval(s)  # Turns s back into a list
```

如果字符串表达式无法创建，则 `__repr__()` 会返回 `<...message...>`

```Python
f = open("foo.txt")
a = repr(f)
# a = "<_io.TextIOWrapper name='foo.txt' mode='r' encoding='UTF-8'">
```

### Number Protocol

在进行 `x + y` 这样计算表达式的时候，解释器会调用一个方法的组合，例如 `x.__add__(y)` 或 `y.__radd__(x)`。
最开始会先尝试 `x.__add__(y)`，但如果 y 是 x 的一个子类，则会先尝试调用 `y.__radd__(x)`。
如果最初尝试失败，且返回 `NotImplemented`，却会去尝试使用右操作数的方法，比如 `y.__radd__(x)`。
如果第二次调用失败，则整个计算会失败。

```Python
a = 42   # int
b = 3.7  # float
a.__add__(b)   # NotImplemented
b.__radd__(a)  # 45.7
```

上面代码模拟了 `a + b` 的计算，由于 `int.__add__()` 方法只能处理整数，因此会返回 `NotImplemented`，然后尝试使用右操作数方法 `float.__radd__()`， 该方法可以接受整数，计算后得到结果。

`__iadd__()`, `__isub__()` 这类方法用于原地运算，例如 `a += b` 和 `a -= b`。
这些运算符与标注运算符的区别在于，原地计算的实现可能能够提供某些订制功能，例如性能优化。
如果原地计算符没有定义，例如 `a += b` 会使用对应的普通计算 `a = a + b`。

没有方法能够定义逻辑运算符号，例如 `or`, `and` 或 `not`。
`or` 和 `and` 实现了短路求值，如果结果已确定，则会停止运算。
这种行为涉及未求值的子表达式，因此无法通过普通函数或方法的相同求值规则来表达。

### Comparison Protocol

对象可以通过多种方式进行比较，最基本的身份验证是 `is` 操作符。

```Python
a = [1, 2, 3]
b = a
a is b  # True

c = [1, 2, 3]
a is c # False
```

`is` 运算符的 Python 的内置部分，无法被重新定义。
其他比较操作符都是同下面表中的方法实现的。

| 方法                  | 描述                                          |
| :-------------------- | :-------------------------------------------- |
| `__bool__(self)`      | Returns False or True for truth-value testing |
| `__eq__(self, other)` | `self == other`                               |
| `__ne__(self, other)` | `self != other`                               |
| `__lt__(self, other)` | `self < other`                                |
| `__le__(self, other)` | `self <= other`                               |
| `__gt__(self, other)` | `self > other`                                |
| `__ge__(self, other)` | `self >= other`                               |
| `__hash__(self)`      | Computes an integer hash index.               |

`__bool__()` 方法用于确定 truthiness 真实性，如果 `__bool__()` 没有定义，则会回退到 `__len()__` 方法，如果都没有定义，则会被视为 `True`。

`__eq__()` 方法用于确定 equality，默认的 `__eq__()` 使用 `is` 操作符实现（当然各种类都会重写 == 逻辑）。
`__ne__()` 会话对应 `!=`，但只要 `__eq__()` 定义了就不需要了。

排序由关系运算符决定 (`< > <= >=`)，要计算 `a < b` 会先尝试执行 `a.__it__(b)` 除非 b 是 a 的子类，这种情况会执行 `b.__gt__(a)`。
如果初始方法没有定义，或者返回一个 `NotImplemented` 则会调用 `b.__gt__(a)`，其他类似的符号也一样。

数值计算包可能利用这点对两个矩阵进行逐元素比较，返回一个结果矩阵。
如果比较不可行，则方法应该返回内置类型 `NotImplemented`，这和异常 `NotImplementedError` 是不同的。

```Python
a = 42
b = 52.3
a.__it__(b)  # NotImplemented
b.__gt__(a)  # True
```

有序对象并不需要实现表中的所有比较操作。
若希望对象能够排序或使用诸如 `min()` 或 `max()` 等函数，则至少必须定义 `__lt()__` 方法。

若需为自定义类添加比较运算符，functools模块中的 `@total_ordering` 类装饰器会很有帮助。
只要至少实现 `__eq__()` 方法及任意一种其他比较方法，它就能自动生成所有必需的方法。

`__hash__()` 方法定义在那些需要放入集合或用作映射（字典）键的实例上。
其返回值是一个整数，对于比较相等的两个实例，该值应当相同。
此外，`__eq__()` 方法应始终与 `__hash__()` 一同定义，因为这两个方法需协同工作。
`__hash__()` 返回的值通常用作各种数据结构的内部实现细节。
然而，两个不同的对象可能具有相同的哈希值。
因此，需要 `__eq__()` 来解决潜在的冲突问题。

### Conversion Protocol

有时候你需要将对象转换成不同的内置类型，包括字符串和数字，下面表格定义了这些方法：

| 方法                            | 描述                                |
| ------------------------------- | ----------------------------------- |
| `__str__(self)`                 | Conversion to a string.             |
| `__bytes__(self)`               | Conversion to bytes.                |
| `__format__(self, format_spec)` | Creates a formatted representation. |
| `__bool__(self)`                | `bool(self)`                        |
| `__int__(self)`                 | `int(self)`                         |
| `__float__(self)`               | `float(self)`                       |
| `__complex__(self)`             | `complex(self)`                     |
| `__index__(self)`               | Conversion to an integer index      |

`__str__()` 方法被 `str()` 函数和与 printing 有关的函数调用。
`__format__()` 方法被 `format()` 函数或字符串方法使用，`format_spec` 参数是一个包含了格式化标准的字符串，该字符串与 `format()` 函数的 `format_spec` 参数相同，例如：

```Python
f"{x:spec}"  # Calls x.__format__
format(x, "spec")  # Calls x.__format__("spec")
"x is {0:spec}".format(x)  # Calls x.__format__("spec")
```

格式规范的语法是任意的，并且可以基于每个对象进行自定义，但对于内置类型有一套标准的约定用法。

`__bytes__()` 方法用于创建一个对象的字节表达式，但不是所有类型都支持字节转换。
数值转换方法 `__bool__()`、`__int__()`、`__float__()` 和 `__complex__()` 应返回对应内置类型的值。

Python 不会通过这些方法执行隐式类型转换，因此，即使对象实现了 `__init__()` 方法，表达式 `3 + x` 仍会引发 TypeError，执行 `__init__()` 的唯一方式是显示调用 `init()` 函数。

`__index__()` 方法在对象参与需要整数值的运算时，会将其转换为整数，这包括序列操作中的索引使用。
例如，如果 `items` 是一个列表，那么运算 `items[x]` 会尝试执行 `item[x.__index__()]`。
即使 `x` 不是整数，`__index__()` 也是需要的，例如一些基本的转换 `oct(x)` 和 `hex(x)`。

### Container Protocol

下面表是容器对象使用的方法

| Method                          | Description                |
| ------------------------------- | -------------------------- |
| `__len__(self)`                 | Returns the length of self |
| `__getitem__(self, key)`        | Returns `self[key]`        |
| `__setitem__(self, key, value)` | Sets `self[key] = value`   |
| `__delitem__(self, key)`        | Deletes `self[key]`        |
| `__contains__(self, obj)`       | obj in self                |

下面是一些例子：

```Python
a = [1, 2, 3, 4, 5, 6]
len(a)  # a.__len__()
x = a[2]  # x = a.__getitem__(2)
a[1] = 7  # a.__setitem__(1, 7)
del a[2]  # a.__delitem__(2)
5 in a  # a.__contains__(5)
```

`__len__()` 方法由 `len()` 函数调用，返回一个非负的长度。
该函数同样用于确定真值，除非同时定义了 `__bool__()` 方法。

要访问单个元素，`__getitem__()` 方法可以通过键值返回一个元素。
key 可以是任意的 Python 对象，但期望是一个有序序列的整数，例如列表和数组。
当使用 `del` 删除单个元素的时候，会调用 `__delitem__()` 方法，`__contains__()` 方法被用来实现 `in` 操作符。

切片操作，例如 `x = s[i:j]` 通过 `__getitem__()`, `__setitem__()` 和 `__delitem__()` 方法实现。
对于切片而言，一个特殊的切片实例作为 key 传递过来，该实例具有描述所请求切片范围的属性。

```Python
a = [1, 2, 3, 4, 5, 6]
x = a[1:5]  # x = a.__getitem__(slice(1, 5, None))
a[1:3] = [10, 11, 12]  # x.__setitem__(slice(1, 3, None), [10, 11, 12])
del a[1:4]  # a.__delitem__(slice(1, 4, None))
```

Python 的切片特性要比许多人以为的要强大的多，例如下面扩展的切片用于处理多维矩阵或数组十分有用。

```Python
a = m[0:100:10]    # Strided slice (step=10)
b = m[1:10, 3:20]  # Multidimensional slice
c = m[0:100:10, 50:75:5]  # Multiple dimensions with strides
m[0:5, 5:10] = n   # extended slice assigment
del m[:10, 15:]    # extended slice deletion
```

一般的切片格式是 `i:j[:step]`，其中步幅是可选的。

此外，省略号 Ellipsis (...) 可用于表示扩展切片中任意数量的前导或尾随纬度。

```Python
a = m[..., 10:20]  # extended slice access with Ellipsis
m[10:20, ...] = n
```

当使用切片的时候，`__getitem__()`, `__setitem__()` 和 `__delitem__()` 分别实现了 access, modification 和 deletion。
但传递给这些方法的并不是一个整数，而是一个包含 Ellipsis 的元组，例如

```Python
a = m[0:10, 0:100:5, ...]
# a = m.__getitem__((slice(0, 10, None), slice(0, 100, 5), Ellipsis))
```

目前，Python 的字符串、元组和列表对扩展切片提供了一定支持。
Python 本身及其标准库中并未使用多维切片或省略号 Ellipsis 功能，这些特性专门为第三方库和框架保留。
最常见的场景在 numpy 这样的库中。

### Iteration Protocol

如果一个实例 obj 支持迭代，会提供一个 `obj.__iter__()` 方法，然后返回一个迭代器。
一个迭代器 iter 返回一个方法，`iter.__next__()`，该方法返回下一个对象或抛出一个 `StopIteration` 表示迭代结束。
这两个方法都被 for 语句使用，也应用于其他执行迭代的操作中。

例如，`for x in s` 语句的执行相当于下面操作：

```Python
_iter = s.__iter__()
while True:
    try:
        x = _iter.__next__()
    except StopIteration:
        break
    # Do statements in body of for loop
    ...
```

如果一堆对象实现了 `__reversed__()` 方法，它可以选择性的提供一个反向迭代器。
该方法应该返回一个迭代器对象，其接口于普通迭代器相同。
内置方法 `reversed()` 会调用该接口，例如：

```Python
for x in reversed([1, 2, 3]):
    print(x)
# 3
# 2
# 1
```

一个常见的实现迭代的方法是使用 `yield` 关键字的生成器函数

```Python
class FRange:
    def __init__(self, start, stop, step):
        self.start = start
        self.stop = stop
        self.step = step
    def __iter__(self)
        x = self.start
        while x < self.stop:
            x += self.step

# Example use:
nums = FRange(0.0, 1.0, 0.1)
for x in nums:
    print(x) # 0.0, 0.1, 0.2, 0.3, ...
```

这是因为生成器本身符合迭代协议，通过这种方式实现迭代器会简单一些，因为只需关注 `__iter__()` 方法。

### Attribute Protocol

下表中的方法分别使用点运算符 (`.`) 和 `del` 运算符来 read, write 和 delete 对象的属性。

| Method                           | Description                                                                       |
| :------------------------------- | :-------------------------------------------------------------------------------- |
| `__getattribute__(self, name)`   | Returns the attribute `self.name`.                                                |
| `__getattr__(self, name)`        | Returns the attribute `self.name` if it's not found through `__getattribute__()`. |
| `__setattr__(self, name, value)` | Sets the attribute `self.name = value`.                                           |
| `__delattr__(self, name)`        | Deletes the attribute `del self.name`.                                            |

每当访问一个属性时，都会调用 `__getattrbute__()` 方法。
如果找到该属性，则返回其值。
否则，会调用 `__getattr__()` 方法，该方法默认行为是抛出一个 `AttributeError` 异常。
设置属性的时候会调用 `__setattr__` 方法，删除属性的时候会调用 `__delattr__` 方法。

这些方法相当直接，因为他们允许一个类完全重新定义所有这些属性的访问方式。
用户自定义的类可以定义属性和描述符，从而实现对属性访问的更精细控制。

### Function Protocol

一个对象可以通过实现 `__call__()` 方法来模拟一个函数。
如果对象 x 实现了该方法，这可以这样调用 `x(arg1, arg2, ...)` 这会触发 `x.__call__(arg1, arg2, ...)`。

许多内置类型都支持函数调用。
例如，类型通过实现 `__call__()` 方法来创新新实例，bound methods 绑定方法通过实现 `__call__()` 将 `self` 参数传递给实例方法。
此外，像 `functools.partial()` 这样的库函数也能创建模拟函数的对象。

### Context Manager Protocol

`with` 语句允许一系列语句在被称为 context manager 上下文管理器的实例控制下执行。
语法大概下面这样：

```Python
with context [as var]:
    statements
```

一个上下文管理器需要实现下表中的方法

| Method                            | Description                                                              |
| :-------------------------------- | :----------------------------------------------------------------------- |
| `__enter__(self)`                 | 进入一个新的上下文时调用。返回值放在 with 语句的 as 指定符后面的变量中   |
| `__exit__(self, type, value, tb)` | 离开上下文时调用。如果发送异常，tb等参数会包含异常类型、异常值和回调信息 |

如果没有发送异常，`__exit__()` 的所有三个值都会设置为 `None`，`__exit__()` 应该返回 `True` 或 `False` 来表明异常是否被处理了。
如果返回 `True`，任何挂起的异常将被清除，程序将正常继续执行，从 `with` 块后的第一条语句开始。

上下文管理接口的主要用途是简化涉及系统状态对象（如打开的文件、网络连接和锁）的资源控制。
通过实现此接口，当执行离开使用对象的上下文时，对象可以安全地清理资源。
