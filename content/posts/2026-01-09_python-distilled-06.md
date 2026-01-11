+++
date = '2026-01-09T8:00:00+08:00'
draft = true
title = 'Python Tricks Part 6: Generators'
tags = ['Python']
+++

生成器是 Python 中一种强大的特性，其通常被介绍为一种定义新型迭代模式的便捷方式。
但生成器从根本上改变了整个函数执行的模式，本篇文章重点关注：生成器、生成器委托、基于生成器的协程，以及生成器的其他内部机制。

## Generators and yield

如果一个函数使用 `yield` 关键字，这定义了一个生成器。
生成器的主要用户是生成用于迭代的值。

例如：

```Python
def countdown(n):
    print("Counting down from", n)
    while n > 0:
        yield n
        n -= 1
# Example use
for x in countdown(n):
    print("T-minus", x)
```

如果调用该函数则不会开始执行：

```Python
c = countdown(10)
# <generator object countdown at 0x106faa260>
```

相反，会创建一个生成器对象。
该生成器对象只有在你迭代它的时候才会开始执行，使用的一种方式是调用 `next()`。

例如：

```Python
next(c)
# Counting down from 10
# 10

next(c)
# 9
```

当调用 `next()` 时，生成器函数会执行语句直到遇到 `yield` 语句。
`yield` 语句会返回一个结果，此时函数的执行被挂起，直到再次调用 `next()`。

当其暂停的时候，函数会保留所有的本地变量和执行环境。
恢复执行时，程序会从 `yield` 之后的语句继续运行。

`next()` 是调用生成器上 `__next__()` 方法的简写形式。
例如，你可以这样：

```Python
c.__next__()
# 8

c.__next__()
# 7
```

通常不会在生成器直接使用 `next()`，而是使用 `for` 或其他一些语句：

```Python
for n in countdown(10):
    statements

a = sum(countdown(10))
```

一个生成器函数会持续生成项目，直到返回为止。
这回导致引发一个 `StopIteration` 异常，并会终止 `for` 循环。
如果生成器函数返回一个 `None` 类型，则该值会被附加到 `StopIteration` 异常上。

例如，加入你有一个同时使用 `yield` 和 `return` 的函数：

```Python
def func():
    yield 10
    return 20
```

代码会这样执行

```Python
def func():
    yield 10
    return 20

f = func()
f
# <generator object func at 0x107095900>

next(f)
# 10

next(f)
# Traceback (most recent call last):
  # File "<stdin>", line 1, in <module>
# StopIteration: 20
```

观察到 return 值被附加到了 `StopIteration` 上面。
如果要获取该值，需要显示捕获改异常：

```Python
try:
    next(f)
except StopIteration as err:
    value = err.value
```

生成器的一个微秒问题是涉及生成器函数仅被部分消耗的情况。
例如这段提前挑出循环的代码：

```Python
for n in countdown(10):
    if n == 2:
        break
    statements
```

在这个例子中，for 循环通过调用 break 语句提前终止，导致关联的生成器位能完全执行。
若生成器函数需要执行某些清理操作，请务必使用 `try-finally` 语句或上下文管理器。

例如：

```Python
def countdown(n):
    print("Counting down from", n)
    try:
        while n > 0:
            yield n
            n -= 1
    finally:
        pinrt("Only made it to", n)
```

即使生成器未被耗尽，也保证会执行 finally 块中的代码。
以及使用上下文管理器的方式：

```Python
def func(filename):
    with open(filename) as file:
        ...
        yield data
        ...
```

资源的正确清理是一个棘手的问题。
只要使用 `try-finally` 或上下文管理器这样的结构，即使生成器提前终止，也能确保其正确执行。

## Restartable Generators

通常生成器只会执行一次，如果要支持重复迭代，定义一个类并将 `__iter__()` 定义为一个生成器：

```Python
class countdown:
    def __init__(self, start):
        self.start = start

    def __iter__(self):
        n = self.start
        while n > 0:
            yield n
            n -= 1
```

## Generator Delegation

生成器一个重要特性是，包含 `yield` 的函数永远不会自己执行，它总是需要外部代码驱动执行。
这使得编写涉及 `yield` 的库函数有些困难，因为仅仅调用生成器函数并不足以使其执行。
为了解决这个问题，可以使用 `yield from` 语句。

```Python
def countup(stop):
    n = 1
    while n <= stop:
        yield n
        n += 1

def countdown(start):
    n = start
    while n > 0:
        yield n
        n -= 1

def up_and_down(n):
    yield from countup(n)
    yield from countdown(n)
```

在编写 必须递归遍历 嵌套可迭代对象 的代码时，`yield from` 十分有用。
例如，下面代码用于展平嵌套列表：

```Python
def flatten(items):
    for i in items:
        if isinstance(i, list):
            yield from flatten(i)
        else:
            yield i

a = [1, 2, [3, [4, 5], 6, 7], 8]
for x in flatten(a):
    print(x, end=' ')
```

该实现的一个局限在于，它仍受限于 Python 递归深度限制，无法处理深度递归的结构。

## Using Generators in Pratice
