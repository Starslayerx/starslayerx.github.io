+++
date = '2026-01-09T8:00:00+08:00'
draft = false
title = 'Python Tricks Part 6: Generators'
categories = ['Note']
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

生成器的一个实际应用是作为代码重构工具，尤其适用于处理包含深层嵌套循环和条件判断的代码。
例如，下面这段脚本用于在 Python 文件目录中搜索所有包含 "spam" 一词的注释：

```Python
import pathlib
import re

# rglob() 递归目录下所有文件
for path in pathlib.Path(".").rglob("*.py"):
    # 防御性编程
    if path.exists():
        # 文本模式打开文件
        with path.open("rt", encoding="latin-1") as file:
            for line in file:
                # 匹配注释
                if m:=re.match(".*(#.*)$", line):
                    comment = m.group(1)
                    if "spam" in comment:
                        print(comment)
```

上面多层嵌套的结构看起来十分混乱，下面是使用生成器实现的代码：

```Python
import re

def get_paths(topdir, pattern):
    for path in pathlib.Path(topdir).rglob(pattern):
        if path.exists():
            yield path

def get_files(paths):
    for path in paths:
        with path.open("rt", encoding="latin-1") as file:
            yield file

def get_lines(files):
    for file in files:
        yield from file

def get_comments(lines):
    for line in lines:
        if m := re.match(".*(#.*)$". line):
            yield m.group(1)

def print_matching(lines, substring):
    for line in lines:
        if substring in lines:
            print(substring)

pypaths = get_paths(".", "*.py")
pyfiles = get_files(pypaths)
lines = get_lines(pyfiles)
comments = get_comments(lines)
print_matching(comments, "spam")
```

上面代码将问题分解为更小的自包含组件，每个组件只关心具体的任务。
每个组件都小巧且独立，这是一种有趣的抽象技术。

例如，考虑 `get_comments()` 生成器。
作为输入，它接受任何可迭代对象，该对象生成文本行。
这段文本可能来自任何地方（文件、列表、生成器等）。
因此，这一功能比以往嵌入在涉及文件中的深层嵌套 for 循环中时更为强大和灵活。
生成器通过将问题分解为小而明确的计算任务，如本例所示，鼓励了一种有益的代码重用方式。
小任务的代码也更容易推理、调试和测试。

生成器在改变函数应用的常规求值规则方面同样有用。
通常，当调整一个函数时，它会立即执行并产生结果。
生成器并不是这样，当调用生成器函数时，其执行会被延迟，直到其他代码片段通过显示调用 `next()` 或 `for` 循环来触发它。

例如下面代码，考虑之前介绍的用于展平嵌套列表的生成器函数：

```Python
def flatten(items):
    for i in items:
        if isinstance(i, list):
            yield from flatten(i)
        else:
            yield i
```

之前对这段代码的考虑是，由于 Python 的递归限制，如果嵌套太深将无法工作。
这可以通过使用栈以不同方式驱动迭代来修复，考虑以下版本：

```Python
def flatten(items):
    stack = [ iter(items) ]
    while stack:
        try:
            item = next(stack[-1])
            if isinstance(item, list):
                stack.append(iter(item))
            else:
                yield item
        except StopIteration:
            stack.pop()
```

此实现构建了一个迭代器的内部栈，不受 Python 内部递归限制的常规约束。
因为它将数据存储在内部列表中，而非在解释器上构建帧。
因此，如果需要展平某个极其深层数据结构中的百万层，该方法会运行良好。

## Enhanced Generators and yield Expressions

在生成器函数内部，`yield` 语句也可以用作表达式，出现在赋值运算符的右侧。

例如：

```Python
def receiver():
    print("Ready to receive")
    while True:
        n = yield
        print("Got", n)
```

这种方式使用 `yield` 的函数有时被称为 “增强型生成器” enhanced generator 或 “基于生成器的协程” generator-based coroutine。
但实际上，“协程” 在现代更常表示异步 async 函数。

一个使用 `yield` 作为表达式的函数仍然是生成器，但其用法有所不同。
它并非生成值，而是响应传递给它的值来执行，例如：

```Python
r = receiver()
r.send(None)  # 生成器刚创建时处于“未启动” 状态，只能 send(None) 或者 next()
# Ready to receive

r.send(1)
# Got 1

r.send(2)
# Got 2

r.send("Hello")
# Got Hello
```

在这个例子中，首次调用 `r.send(None)` 是必要的，以便生成器执行到达第一个 `yield` 表达式语句。
此时，生成器暂停执行，等待通过关联的生成器对象 `r` 的 `send()` 方法向其发送一个值。
传递给 `send()` 的值由生成器中的 `yield` 表达式返回。
一但接收到值，生成器会继续执行语句，直到遇到下一个 `yield`。
该函数会无限运行，可以通过 `close()` 方法来关闭生成器。

```Python
r.close()
r.send(4)
# Traceback (most recent call last):
#   File "<stdin>", line 1, in <module>
# StopIteration
```

`close()` 操作会在当前的 `yield` 位置引发一个 `GeneratorExit` 异常。
通常这会导致生成器静默终止。
一但关闭，如果继续向生成器发送值，将会引发 `StopIteration` 异常。

在协程内部，可以使用 `throw(ty [val, [,tb]])` 方法抛出异常，其中 `ty` 是异常类型，`val` 是异常参数，`tb` 是可选的回溯信息。
例如：

```Python
r = receiver()
# Ready to recieve
r.throw(RuntimeError, "Dead")
# Traceback (most recent call last):
#   File "<stdin>", line 1, in <module>
#   File "receiver.py", line 14, in receiver
#     n = yield
# RuntimeError: Dead
```

无论以何种方式抛出异常，都会从生成器中当前正在执行的 `yield` 语句开始传播。
生成器可以选择捕获该异常并酌情处理，如果生成器未处理异常，异常将传播至生成器外部，由更高层代码处理。

## Applications of Enhanced Generators

增强生成器是一种奇怪的编程结构，不同于简单的 for 循环，增强型生成器没有核心语言特性来驱动。
那么，为什么会需要一个接收传入值的函数呢？

历史上，增强生成器在并发库中有大量使用，尤其是基于异步 I/O 的。
在这种情况下，他们通常被称为 “协程” 或 “基于生成器的协程“。
然而，大多数功能都已经被融入 Python 的 async 和 await 特性中了。
因此，对于该特定用例， yield 几乎没有实际意义。
尽管如此，它仍有一些实际应用场景。

类似生成器，一个增强生成器可以用于实现不同种类的计算和控制流。
其中一个例子就是 `contextlib` 库中的 `@contextmanager` 装饰器：

```Python
@contextmanager
def manager():
    print("Entering")
    try:
        yield "somevalue"
    except Exception as e:
        print("An error occurred", e)
    finally:
        print("Leaving")
```

这里，生成器被用来连接上下文管理器的两个部分。
上下文管理器可以是通过实现一下协议的对象来定义的：

```Python
class Manager:
    def __enter__(self):
        return somevalue
    def __exit__(self, ty, val, tb):
        if ty:
            # An exception occurred
            ...
            # Return True/ if handled. False otherwise
```

使用 `@contextmanager` 生成器时，管理器进入时会执行 `yield` 语句前所有代码（`__enter__()`）方法。
当管理器退出时，会执行 `yield` 后面的所有代码（通过 `__exit__()` 方法）。
如果发送了一个错误，则被报告为 `yield` 语句上的异常。
例如：

```Python
with manager() as val:
    print(val)
# Entering
# somevalue
# Leaving

with manager() as val:
    print(int(val))
# Entering
# An error occurred invalid literal for int() with base 10: 'somevalue'
# Leaving
```

为了实现这一功能，使用一个包装类。
以下是一个简化的实现：

```Python
class Manager:
    """这是一个包装器，允许将生成器函数用作 with 语句的上下文管理器。"""
    def __init__(self, gen):
        self.gen = gen  # 接收一个生成器

    def __enter__(self):
        # 执行到 yield 处
        return self.gen.send(None)

    def __exit__(self, ty, val, tb):
        """
        - ty: 异常类型
        - val: 异常值
        - tb: 追踪信息
        """
        # 传播异常
        try:
            # 如果有异常
            if ty:
                try:
                    self.gen.throw(ty, val, tb)  # 将异常抛回生成器
                except ty:
                    return False  # 异常继续传播
            # 没有异常
            else:
                self.gen.send(None)  # 继续执行生成器
        except StopIteration:
            return True  # 生成器正常结束
```

扩展生成器的另一个应用是利用函数封装一种“工作器”任务。
函数调用的核心特性之一是，它能够创建局部变量环境。
访问局部变量的效率极高，远快于访问类和示例的属性。
由于生成器会一直存在，直到显示关闭或消毁，因此可以利用生成器来设置长期运行的任务。

以下是一个生成器示例，它接收字节片段并将其组装成行：

```Python
@consumer
def line_receiver():
    data = bytearray()  # 缓冲区
    line = None         # 解析出的一整行
    linecount = 0       # 当前缓冲区换行数量

    while True:
        part = yield line
        linecount += part.count(b'\n')
        data.extend(part)
        if linecount > 0:
            index = data.index(b'\n')  # 查找 \n 位置
            line = bytes(data[:index+1])
            data = data[index+1]
            linecount -= 1
        else:
            line = None
```

在这个例子中，生成器被设计为接受字节段 fragments，这些字节片段被收集到一个字节数组中。
如果数组包含换行符，则提取并返回一行。否则，返回 None。

以下示例展示了其工作原理：

```Python
r = line_receiver()
r.send(b"hello")
r.send(b"world\nit ")
# b'hello world\n'
r.send(b"works!")
r.send(b"\n")
# b'it works!\n'
```

类似的代码可以写成一个类：

```Python
class LineReceiver:
    def __init__(self):
        self.data = bytearray()
        self.linecount = 0

    def send(self, part):
        self.linecount += part.count(b"\n")
        self.data.extend(part)

        if self.linecount > 0:
            index = self.data.index(b"\n")
            line = bytes(self.data[:index+1])
            self.data = self.data[index+1:]
            self.linecount -= 1
            return line
        else:
            return None
```

编写一个类可能更加熟悉，但代码却在某种程度上更加复杂了。
使用生成器向接收器输入大量数据块的速度比使用类代码块的速度快大约 40%~50%。
大部分性能提升源于笑出来实例属性查找（局部变量访问速度更快）。

尽管存在其他许多潜在应用，但最重要的是要记住：如果看到 `yield` 出现在与迭代无关的上下文中，它很可能正在属于诸如 `send()` 或 `throw()` 等增强功能。

## Generators and the Bridage to Awaiting

生成器函数的一个经典应用场景是在与异步 I/O 相关的库中，例如标准库的 asyncio 么模块。
然而，自 Python 3.5 起，这些功能大多已转移至与异步函数，及 await 语句相关的语言特性中。

await 语句实际上是与一个伪装成生成器的对象进行交互。
下面是 await 所使用底层协议的示例：

```Python
class Awaitable:
    def __await__(self):
        print("About to wait")
        yield  # Must be a  generator
        print("Resuming")

# Function compatible with "await". Returns an "awaiable"
def function():
    return Awaitable

async def main():
    await function()  # 主要是 Awaitable 就行，不一定要 async def
```

下面可以这样运行 `asyncio`

```Python
import asyncio
asyncio.run(main())
# About to await
# Resuming
```

## Final Wrods: A Brief History of Generatros and Looking Forward

生成器是Python中较为引人注目的成功案例之一。
然而，它们只是关于迭代的更大故事的一部分。
迭代是所有编程任务中最常见的之一。
在Python的早期版本中，迭代是通过序列索引和`__getitem__()`方法实现的。
后来，这演变成了基于`__iter__()`和`__next__()`方法的当前迭代协议。
生成器随后出现，作为一种更便捷的实现迭代器的方式。
在现代Python中，几乎没有理由使用生成器以外的任何方式来实现迭代器。
即使是在你可能自己定义的可迭代对象上，`__iter__()`方法本身也通常以这种方式方便地实现。

在Python的后续版本中，生成器承担了新的角色，因为它们演变成了与协程相关的 “增强” 功能（例如`send()`和`throw()`方法）。
这些功能不再纯粹与迭代相关，而是为在其他上下文中使用生成器开辟了可能性。
最值得注意的是，这构成了许多用于网络编程和并发的所谓“异步”框架的基础。
然而，随着异步编程的发展，其中大部分功能已经转变为与`async/await`语法相关的后续特性。
因此，现在很少看到生成器[…]
