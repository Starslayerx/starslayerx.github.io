+++
date = '2025-08-10T8:00:00+08:00'
draft = false
title = 'Python Function Parameters'
tags = ["Python"]
+++

今天是周日, 简单写点吧, 简单总结一下 Python 中函数参数

## Python Function Parameters

Python 函数参数机制非常灵活丰富, 理解各种参数类型及其用法对于写出优雅、易维护的代码非常重要. 本文将介绍 Python 中函数参数的种类与用法, 并详细讲解 Python 3.8 引入的参数分隔符 / 和 \*, 帮助你更好地设计函数接口.

#### 1. Postional Arguments 位置参数

函数定义中最常见的参数, 调用时按顺序传入值

```Python
def greet(name, age):
    print(f"Hello, {name}. You are {age} years old.")

greet("Alice", 30)  # Hello, Alice. You are 30 years old.
```

#### 2. Keyword Arguments 关键字参数

调用时以 `key=value` 形式传入, 顺序可变

```Python
greet(age=30, name="Alice")
```

#### 3. Default Arguments 默认参数

定义函数时给参数赋默认值, 调用时可省略

```Python
def greet(name, age=20):
    print(f"Hello, {name}. You are {age} years old.")

greet("Bob")        # 使用默认年龄20
greet("Bob", 25)    # 指定年龄
```

- 注意: 使用默认参数尽量不要使用可变类型(mutable), 例如列表, 因为默认参数是存储在函数中的, 而非函数实例中, 多次调用会改变默认值的内容.

```Python
def greet(names: list[str] = ["Alice", "Bob"]):
    ...
```

若希望使用默认值, 建议使用下面这种方法

```Python
def greet(names: list[str] | None = None):
    if not names:
        names = ["Alice", "Bob"]
    ...
```

同样的, 默认值参数如果为一个表达式, 则是在定义时求值, 而非运行改函数时才求值

#### 4. `*args` 可变位置参数

用于接收任意数量的位置参数, 形成元组

```Python
def sum_all(*args):
    return sum(args)

sum_all(1, 2, 3)  # 6
sum_all()         # 0
```

#### 5. `**kwargs` 可变关键字参数

用于接收任意数量的关键字参数, 形成字典

```Python
def print_info(**kwargs):
    for k, v in kwargs.items():
        print(f"{k} = {v}")

print_info(name="Alice", age=30)
```

### / 和 \* 的用法

Python 3.8 引入了两种新的函数参数分隔符: 斜杠 `/`(forward slash) 和 星号 `*`(asterisk) 符号, 用于更精细地控制参数的调用方式

#### Postional-only parameters (`/`)

斜杠前的参数必须通过位置传递, 不能用关键字传递

```Python
def func(a, b, /, c, d):
    print(a, b, c, d)
```

调用时

```Python
func(1, 2, c=3, d=4)   # 正确
func(1, 2, 3, 4)       # 也正确
func(a=1, b=2, c=3, d=4)  # 错误，a 和 b 不能用关键字传递
```

用途:

- 保护函数接口的参数顺序, 避免调用者用关键字修改参数值
- 兼容一些C语言扩展模块的调用约定
- 明确哪些参数是"位置专用"的

#### Keyword-only parameters (`*`)

星号后的参数必须用关键字传递, 不能用位置传递

```Python
def func(a, b, *, c, d):
    print(a, b, c, d)
```

调用时

```Python
func(1, 2, c=3, d=4)   # 正确
func(1, 2, 3, 4)       # 错误，c 和 d 只能用关键字传递
```

用途:

- 强制调用者明确指定关键字参数, 提高代码可读性
- 避免参数顺序引起的混淆

#### Use both `/` and `*`

`/` 和 `*` 也可以同时使用

```Python
def func(a, b, /, c, d, *, e, f):
    print(a, b, c, d, e, f)
```

调用时

- a 和 b 只能用位置参数传递
- c 和 d 都可以
- e 和 f 只能用关键字参数传递
