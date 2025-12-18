+++
date = '2025-12-15T8:00:00+08:00'
draft = true
title = 'Python Tricks Part 3: Program Structure and Control Flow'
tags = ['Python']
+++

## Exceptions

异常 exceptions 具有一些标准属性，这些属性在需要针对错误执行进一步操作的代码中可能非常有用。

```Python
e.args
```

这是引发异常时提供的元组，在大多数情况下，这是一个包含描述错误字符串的单元元素元组。
对于 OSError 异常，其值是一个包含整数错误码、字符串错误消息，以及可选文件名的 2 元组或 3 元组。

```Python
e.__cause__
```

如果该异常是在处理另一个异常时有意引起的 `raise ... from ...`，Python 会将这两个异常链接起来，形成异常链。

```Python
e.__context__
```

如果异常是处理异常时无意间导致的，则会产生 `e.__context__`。

```Python
e.__traceback__
```

与异常相关联的堆栈回溯对象。

用于存储异常值的变量仅在相关的 except 块内部可以访问，一但控制论离开该块，该变量将变为未定义。

```Python
try:
    int('N/A')
except ValueError as e:
    print('Failed:', e)

print(e) # Fails -> NameError. 'e' not defined
```

多异常处理块通过多个异常子句指定：

```Python
try:
    # do something
except TypeError as e:
    # Handle Type error
except ValueError as e:
    # Handle Value error
```

当然也可以在单个子句中处理多个异常类型

```Python
try:
    # do something
except (TypeError, ValueError) as e:
    # Handle Type or Value error
```

可以使用 pass 忽略报错

```Python
try:
    # do something
except ValueError:
    pass
```

通常静默忽略报错是危险的，通常会引起许多奇怪的 bug，即使要忽略，也要通过某种方式报告异常的发生。

如果程序要捕捉除退出外的任何异常，可以这样：

```Python
try:
    # do something
except Exception as e:
    print(f'An error occured: {e!r}')
```

其中，这里的 `e!r` 是将信息转化为 `repr()` 方便输出。

| 写法    | 等价于     | 含义                 |
| :------ | :--------- | :------------------- |
| `{e}`   | `str(e)`   | 人类可读字符串       |
| `{e!s}` | `str(e)`   | 同上                 |
| `{e!r}` | `repr(e)`  | 面向开发者，精确表示 |
| `{e!a}` | `ascii(e)` | 仅 ASCII 表示        |

try 语句也支持 else 块，该块必须跟在 except 后面，如果没有引起对应异常就会执行 else 里面的内容：

```Python
try:
    file = open('foo.txt', 'rt')
except FileNotFoundError as e:
    print(f'Unable to open foo:' {e})
else:
    data = file.read()
    file.close()
```

`finally` 中定义无论如何都要执行的清理操作，例如：

```Python
file = open('foo.txt', 'rt')
try:
    # do some stuff
finally:
    file.close()
    # File closed regardless of what happened
```

`finally` 不是用来捕获异常的，而是执行无论是否出现异常都要执行的代码。
如果没有异常产生，则会立刻执行 `finally` 块中代码。

### The Exception Hierarchy

异常层次结构

异常处理的一大挑战就是管理大量潜在的可能产生的异常。
例如，仅内置异常就有 60 都多种，此外，还有标准库中的各种数百种异常。
通常没有任何办法去确定代码可能产生的异常类型。

异常并未作为函数调用签名的一部分被记录，也没用任何编译器来验证代码中的异常处理是否正确。
因此，异常处理有时会显得随意且缺乏条理。

在管理异常时，一个有用的工具是认识到他们通过继承被组织成一个层次结构。
在写代码时不使用具体的异常，而是聚焦于更加通用的异常类别。

例如，在容器中查找值时可能出现各种错误：

```Python
try:
    item = items[index]
except IndexError:  # Raised if items is a seqence
    ...
except KeyError:  # Raised if items is a mapping
    ...
```

不去判断两种具体的错误，而像下面这样：

```Python
try:
    item = items[index]
except LookupError:
    ...
```

`LookupError` 是一个表示异常高层级分组的类。
`IndexError` 和 `KeyError` 都继承自 `LookupError`，因此这里会捕获这里两个报错。

下表描述了常见的内置异常

| 异常类型        | 描述                                                                            |
| --------------- | ------------------------------------------------------------------------------- |
| BaseException   | 所有异常的根类型 Root class                                                     |
| Exception       | 所有程序相关的 (program-related) 基本异常类型 Base class                        |
| ArithmeticError | 所有数学相关的 (math-related) 基本异常类型 Base class                           |
| ImportError     | 所有导入相关的 (import-related) 基本异常类型                                    |
| LookupError     | 所有容器查找或范围相关的 (container lookup) 基本异常类型                        |
| OSError         | 所有系统相关的 (system-related) 基本异常类型 (alias: IOError, EnvironmentError) |
| ValueError      | 所有值错误相关的 (value-related) 基本异常类型，包含 Unicode                     |
| UnicodeError    | 有关 Unicode 字符串编码的基本异常类型                                           |

其中，`BaseException` 类型很少使用，因为其会匹配所有可能的异常。
包括影响程序控制留的异常，例如 `SystemExit`, `KeyboardInterrupt` 和 `StopIteration`，捕获这些异常并发本意。
反而，所有普通的程序错误都继承 `Exception`。

- `ArithmeticError` 是所有数学相关错误，例如 `ZeroDivisionError`, `FloatingPointError` 和 `OverflowError`
- `ImportError` 是所有导入相关错误
- `LookupError` 是所有容器访问相关错误
- `OSError` 是所有来自操作系统和环境的错误，该异常涵盖了很大范围内容，包括文件、网络连接、权限、管道、超时等
- `ValueError` 通常来自一个操作的错误输入
- `UnicodeError` 是 `ValueError` 的一个子类，代表所有和 Unicode 相关的编码解码异常

下表是一些直接继承自 `Exception` 的异常，但不属于一个大的异常组中。

| Exception Class     | Description                                          |
| ------------------- | ---------------------------------------------------- |
| NameError           | Name not found in the local or global namespace      |
| NotImplementedError | Unimplemented feature                                |
| RuntimeError        | A generic "something bad happened" error             |
| TypeError           | Operation applied to an object of the wrong type     |
| UnboundLocalError   | Usage of a local variable before a value is assigned |
| AssertionError      | Failed assert statement                              |
| AttributeError      | Bad attribute lookup on an object                    |
| EOFError            | End of File                                          |
| MemoryError         | Recoverable out of memory error                      |

### Exceptions and Control Flow

一般异常都是用于错误处理的，然而有几个异常用于修改程序控制流，下表中的几个都直接继承自 BaseException

| Exception Class   | Description                                    |
| :---------------- | :--------------------------------------------- |
| SystemExit        | Raised to indicate program exit                |
| KeyboardInterrupt | Raised a programe is interrupted vai Control-C |
| StopIteration     | Raised to signal the end of iteration          |

`SystemExit` 用于让程序按预期终止，作为参数可以提供一个整数退出码或字符串消息。
如果提供一个字符串，会向 `sys.stderr` 输出，并以退出码 1 退出。

```Python
import sys

if len(sys.argv != 2):
    raise SystemExit(f'Usage: {sys.argv[0]} filename')
filename = sys.argv[1]
```

当程序接收到 SIGINT 信号（通常按下 Ctrl-C 触发）时，会引发 `KeyboardInterrupt` 异常。
该异常的特殊之处在于它是异步的，这意味着它几乎可以在程序执行的任何时刻、任何语句处发生。
Python 的默认行为是在此处直接终止程序，若需要控制 SIGINT 信号传递，可以使用 signal 库。

`StopIteration` 异常是迭代协议的一部分，用于表示迭代结束。

### Defining New Exceptions

如果要创建自定义异常，继承 `Exception` 类：

```Python
class NetworkError(Exception):
    pass
```

抛出自定义异常也使用 raise 语句：

```Python
raise NetworkError('Cannot find host')
```

当引发异常时，`raise` 语句提供的可选值将作为异常类构造函数的参数。
大多数情况下，这是一个表示某种错误的字符串。
然而，用户自定义的异常可以设计为接收一个或多个异常值：

```Python
class DeviceError(Exception):
    def __init__(self, errno, msg):
        self.args = (errno, msg)
        self.errno = errno
        self.errmsg = msg

# Raises an exception (multiple arguments)
raise DeviceError(1, 'Not Responding')
```

args 是异常类的特殊属性，当异常被捕获但没有指定具体变量是，args 会自动添加。
该属性用于打印异常回溯信息，如果未定义该属性，当错误发生时，用户将无法看到任何有关异常的有用信息。

通过继承，可以将异常组织成一个层次结构。

```Python
class HostnameError(NetworkError):
    pass

class TimeoutError(NetworkError):
    pass

def error1():
    raise HostnameError('Unkonw host')

def error2():
    raise TimeoutError('Timed otu')

try:
    error1()
except NetworkError as e:
    if type(e) is HostnameError:
        # Perform speical actions for this kind of error
```

### Chained Exceptions

有时候你可能会向抛出一个链式异常

```Python
class ApplicationError(Exception):
    pass

def do_something():
    x = int('N/A')  # raise ValueError

def spam():
    try:
        do_something()
    except Exception as e:
        raise ApplicationError('It failed') from e
```

如果抛出 `ApplicationError` 报错，则会有以下报错信息：

```text
---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
Cell In[3], line 3, in spam()
      2 try:
----> 3     do_something()
      4 except Exception as e:

Cell In[2], line 2, in do_something()
      1 def do_something():
----> 2     x = int('N/A')

ValueError: invalid literal for int() with base 10: 'N/A'

The above exception was the direct cause of the following exception:

ApplicationError                          Traceback (most recent call last)
Cell In[4], line 1
----> 1 spam()

Cell In[3], line 5, in spam()
      3     do_something()
      4 except Exception as e:
----> 5     raise ApplicationError('It failed') from e

ApplicationError: It failed
```

如果捕获 `ApplicationError`，则 `__cause__` 属性会包含其他异常，例如：

```Python
try:
    spam()
except ApplicationError as e:
    print('It failed. Reason:', e.__cause__)
```

输出如下：

```text
It failed. Reason: invalid literal for int() with base 10: 'N/A'
```

若想在不包含其他异常链的情况下引发新异常，可以 `raise ... from None`

```Python
def spam():
    try:
        do_something()
    except Exception as e:
        raise ApplicationError('It failed') from None
```

输出如下：

```text
---------------------------------------------------------------------------
ApplicationError                          Traceback (most recent call last)
Cell In[9], line 1
----> 1 spam()

Cell In[8], line 5, in spam()
      3     do_something()
      4 except Exception as e:
----> 5     raise ApplicationError('It failed') from None

ApplicationError: It failed
```

出现在 except 块中的编程错误同样会导致链式异常，但其运作方式略有不同。
假设有如下缺陷的代码：

```Python
def spam():
    try:
        do_something()
    except Exception as e:
        print('It failed:', err)  # str undefined (typo)
```

这样导致的报错有些许不同：

```text
---------------------------------------------------------------------------
ValueError                                Traceback (most recent call last)
Cell In[4], line 3, in spam()
      2 try:
----> 3     do_something()
      4 except Exception as e:

Cell In[3], line 2, in do_something()
      1 def do_something():
----> 2     x = int('N/A')

ValueError: invalid literal for int() with base 10: 'N/A'

During handling of the above exception, another exception occurred:

NameError                                 Traceback (most recent call last)
Cell In[5], line 1
----> 1 spam()

Cell In[4], line 5, in spam()
      3     do_something()
      4 except Exception as e:
----> 5     print('It failed:', err)

NameError: name 'err' is not defined
```

### Exception Trackbacks

异常栈回调信息在 `__traceback__` 属性中，为了报告 bug，可能需要自己生成回溯信息。
使用 `traceback` 模块来实现：

```Python
import traceback

try:
    spam()
except Exception as e:
    tblines = traceback.format_exception(type(e), e, e.__traceback__)
    tbmsg = ''.join(tblines)
    print('It failed:')
    print(tbmsg)
```

`format_exception()` 函数格式化异常信息，返回一个字符串，每个字符串是堆栈跟踪的一行

输出如下

```text
It failed
Traceback (most recent call last):
  File "<ipython-input-4-893664aa1d25>", line 3, in spam
    do_something()
  File "<ipython-input-3-0308b00b259c>", line 2, in do_something
    x = int('N/A')
        ^^^^^^^^^^
ValueError: invalid literal for int() with base 10: 'N/A'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<ipython-input-7-747b1ec03c7f>", line 2, in <module>
    spam()
  File "<ipython-input-4-893664aa1d25>", line 5, in spam
    print('It failed:', err)
                        ^^^
NameError: name 'err' is not defined
```

### Exception Handling Advice

异常处理是大型应用程序里很难的一部分，下面是一些实用的规则：

第一条规则是不要捕获哪些在代码特定位置无法直接处理的异常，例如

```Python
def read_data(filename):
    with open(filename, 'rt') as file:
        rows = []
        for line in file:
            row = line.split()
            row.append((row[0], int(row[1]), float(row[2])))
    return rows
```

假如 `open()` 函数传入了一个错误的文件名，但其内部不应该判断这个异常。
`read_data()` 应该抛出异常，并在外面处理这个可能的问题，例如给函数提供文件名的部分应该处理这个问题。

另一方面，函数或许能从错误中恢复：

```Python
def read_data(filename):
    with open(filename, 'rt') as file:
        rows = []
        for line lin file:
            row = line.split
            try:
                row.append((row[0], int(row[1]), float(row[2])))
            except ValueError as e:
                print('Bad row:', row)
                pirnt('Reason:', e)
        return rows
```

在捕获异常时，尽量使用 except 子句的范围合理精确。
上述代码本可以通过 `except Exception` 来捕获所有错误，但这样会导致代码捕获本身不应该被忽略的合法编程错误，这会使调试困难。

最后，如果要显示引发异常，考虑自定义异常

```Python
class ApplicationError(Exception):
    pass

class UnauthorizedUserError(ApplicationError):
    pass

def spam():
    ...
    raise UnauthorizedUserError('Go away')
    ...
```

这看似细微，但在大型代码库中，更棘手的问题之一是如何确定程序故障的责任归属。
如果要自定义异常，最好能够区分合法编程异常和故意抛出的异常。
例如，如果代码抛出上面的 `ApplicationError` 那么你立刻就知道为什么会抛出这个异常。
另一方面，如果抛出了内置异常，那通常意味着更加严重的问题。

## Contenxt Managers and the with Statement

管理系统资源，例如文件、锁和连接相关的异常通常是一个棘手的问题。

例如，一个被抛出的异常可能导致控制流跳过负责释放关键资源（如锁）的语句。

`with` 语句允许一系列语句在运行时上下文中执行，该上下文充当上下文管理器的对象控制。

```Python
with open('debuglog', 'wt') as file:
    file.write('Debugging\n')
    statements
    file.write('Done\n')

import threading
lock = threading.Lock()
with Lock:
    # Critical section
    statements
    # End critical section
```

在第一个示例中，当控制流离开后续语句时，`with` 语句会自动关闭已打开的文件。
在第二个示例中，当控制进入和离开后续语句时，`with` 语句会自动获取并释放锁。

`with obj` 语句允许对象 `obj` 管理控制流进入和退出其关联代码块时的行为。
当 `with obj` 语句执行时，其会调用 `obj.__enter__` 表示创建了一个新的上下文。
当离开该上下文时，会调用 `obj.__exit__(type, value, traceback)` 方法。
如果没有引发任何异常，这三个参数都设置为 None。
否则，他们包含与导致控制流离开上下文的异常相关类型、值和回溯信息。

如果 `__exit__()` 返回 `True`，则说明异常已经被正确处理，不应该被传播。
返回 `None` 或 `False` 会导致异常传播。

`with obj` 语句接受一个可选的 `as var` 指定符 (specifier)。
如果指定，`obj.__enter__()` 返回的值将被赋予给 `var`。
这个值通常与 `obj` 相同，因为这允许在同一个步骤中构造对象并将其用作上下文管理器。

考虑下面类：

```Python
class Manager:
    def __init__(self, x):
        self.x = x

    def yow(self):
        pass

    def __enter__(self):
        return self

    def __exit__(self, ty, val, tb):
        pass
```

你可以在上下文管理器中创建并使用一个实例：

```Python
with Manager(42) as m:
    m.yow()
```

下面是一个关于 list transactions 的例子：

```Python
class ListTransaction:
    def __init__(self, thelist):
        self.thelist = thelist

    def __enter__(self):
        self.workingcopy = list(self.thelist)
        return self.workingcopy

    def __exit__(self, type, value, tb):
        if type is None:
            self.thelist[:] = self.workingcopy
            return False
```

该类允许对现有列表进行一系列修改，但只有在未发生任何异常的情况下，修改才会生效。
否则，原始列表将保持不变。

```Python
items = [1, 2, 3]

with ListTransaction(items) as working:
    working.append(4)
    working.append(5)
    print(items)  # Produces [1, 2, 3, 4, 5]

try:
    with ListTransaction(item) as working:
        working.apned(6)
        working.apned(7)
        raise RuntimeError("We're hosed!")
except RuntimeError:
    pass

print(items)  # [1, 2, 3, 4, 5]
```

`contextlib` 库包含更多关于上下文管理器的高级用法。
如果经常使用上下文管理器，该库值得一看。

## Assertions and \_\_debug\_\_

`assert` 语句可以在程序中引入调试代码，一般形式是：

```Python
assert test [, msg]
```

其中 `test` 是一个返回 `True` 或 `False` 的表达式。
如果为 `False` 则 `assert` 会抛出一个 `AssertionError`，并包含一条 msg 信息

```Python
def write_data(file, data):
    assert file, 'write_data: file not defiend!'
```

断言语句不应用于必须执行，以确保程序正确性的代码。
因为 Python 以优化模式允许时（通过解释器的 -O 选项指定），这些断言将不会执行。

因此使用断言来检查用户输入或某些重要的操作结果是错误的，`assert` 用于永远应该为 `True` 的不变量。
如果这一项被违反，则应该报一个 bug，而不是给用户一个 error。

例如，如果之前展示的 `write_data()` 函数旨在提供最终用户使用，那么 `assert` 语句应该替换为常规的 `if` 语句，并配合适当的错误处理机制。
`assert` 的使用常见于测试中，例如，你可能使用其包含一个最小测试：

```Python
def factorial(n):
    result = 1
    while n > 1:
        return *= n
        n -= 1
    return result

assert factorial(5) == 120
```

这种测试不是为了详尽，而是为了类似“冒烟测试”的功能。
如果函数中存在明显的错误，代码在导入时会因断言失败而立刻崩溃。

断言在指定预期输入和输出的预期方面也很有用。

```Python
def factorial(n):
    assert n > 0, "must supply a postive value"
    result = 1
    while n > 1:
        result *= n
        n -= 1
    return result
```

同样，这不是为了检查用户输入。
这更多是用于检查系统内部的一致性，如果其他代码传入负数，那么就会报错，这样会方便调试。

## Final Words

尽管 Python 支持多种涉及函数和对象的编程风格，但其程序执行的基本模型仍属于命令式编程。
异常处理需要非常谨慎对待的部分，尤其是设计库、框架和 API 时，异常还可能严重影响妥善管理，这些问题通常需要使用上下文管理器来解决。
