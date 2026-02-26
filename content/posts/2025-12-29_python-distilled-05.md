+++
date = '2025-12-19T8:00:00+08:00'
draft = false
title = 'Python Tricks Part 5: Functions'
categories = ['Note']
tags = ['Python']
+++

函数是 Python 的基础模块，本篇会介绍 function application 函数定义、function application 函数应用、scoping rules 作用域规则、closures 闭包、decorators 装饰器和其他函数式编程特性。
特别关注不同的编程习惯 idioms、求值模型以及与函数相关的模式。

## Default Arguments

你可以通过在函数定义处赋值的方式，给函数的参数添加默认值，例如：

```Python
def split(line, delimiter=","):
    statements
```

当一个函数定义了默认参数的时候，其右侧都必须是含有默认值的可选参数。

默认函数参数会在函数首次定义的时候调用一次，这有时会导致出乎意料的行为：

```Python
def func(x, items=[]):
    items.append(x)
    return items

func(1)  # returns [1]
func(2)  # returns [1, 2]
func(3)  # returns [1, 2, 3]
```

注意到每次掉都将函数默认值给修改了，要避免这种行为，使用 `None` 并进行检查：

```Python
def func(x, items=None):
    if not items:
        items = []
    items.append(x)
    return items
```

通常来说，建议只使用不可变对象作为默认参数值。

## Variadic Arguments

可变参数

如果在最后一个参数前使用 asterisk 星号作为前缀，函数就可以接受可变数量的参数。

```Python
def product(first, *args):
    result = first
    for x in args:
        result = result *x
    return result

product(10, 20)      # 200
product(2, 3, 4, 5)  # 120
```

在这个例子中，所有的额外参数都作为一个元组放在 `args` 变量中。
对于元组，你可以使用序列的标准操作处理，如迭代、切片，解包等。

## Keyword Arguments

函数参考可以通过显示命名每个参数并指定值来提供函数参数。
这被称为 keyword arguments 关键字参数，例如：

```Python
def func(w, x, y, z):
    statements

# Keyword argument invocation
func(x=3, y=22, w="hello", z=[1, 2])
```

使用关键字参数，参数的顺序就不重要了，只要每个参数获取到了对应的值即可。
如果有任何不符合的关键字参数，则会抛出一个 `TypeError` 异常。
关键字参数按照它们在函数调用中指定的顺序进行求值。

Positional arguments 位置参数和 keyword arguments 关键字参数可以在同一个函数调用中出现，
前提是所有的位置参数出现在前面，所有非可选参数都有值，且没有参数被赋予多个值。

下面是一个例子：

```Python
func("hello", 3, z=[1, 2], y=22)
func(3, 22, w="hello", z=[1, 2])  # TypeError. Multiple values for w
```

如果想，可以强制使用关键字参数，通过将参数放到星号后面来实现：

```Python
def read_data(filename, *, debug=False):
    ...

def product(first, *values, scale=1):
    result = first * scale
    for val in values:
        result = resul * val
    return result
```

在上面例子中，`read_data` 函数的 `debug` 参数只能通过关键字参数来确定。
这个限制通常能提高代码可读性：

```Python
data = read_data("Data.csv", True)  # No. TypeError
data = read_data("Data.csv", debug=True)  # Yes.
```

`product()` 函数接受任意数量的位置参数，和一个可选的关键字参数，例如：

```Python
result = product(2, 3, 4)            # 24
result = product(2, 3, 4, scale=10)  # 240
```

## Variadic Keyword Arguments

如果函数最后一个参数有前缀 `**`，那所有的额外关键字参数（没有匹配任何参数名的），都会被放在一个字典中传入函数。
此字典中条目的顺序保证与关键字参数的提供顺序一致。

接受任意关键字参数可能是定义函数的一种有效方式，这些函数需要处理大量的配置选项，若全列为参数会显得过于繁琐。

```Python
def make_table(data, **parms):
    fgcolor = parms.pop("fgcolor", "black")
    bgcolor = parms.pop("bgcolor", "white")
    width = parms.pop("width", None)
    # No more options
    if parms:
        raise TypeError(f"Unsupported configuration options {list(parms)}")

make_table(items, fgcolor="black", bgcolor="white", border=1,
                  borderstyle="grooved", cellpadding=10,
                  width=400)
```

字典的 `pop()` 方法去除字典中的一个元素并返回，如果没有定义返回一个可能的默认值。

## Functions Accepting All Inputs

如果一起使用 `*` 和 `**`，可以编写一个接受任意参数组合的函数。
位置参数通过元组传递，关键字参数通过字典传递。

```Python
# Accept variable number of positional or keyword arguments
def func(*args, **kwargs):
    # args is a tuple of positional args
    # kwargs is a dictionary of keyword args
    ...
```

`*args` 和 `**kwargs` 这样的组合通常用于编写包装器、装饰器、代理以及类似类型的函数。

例如，假设你有一个从可迭代对象处理行的函数：

```Python
def parse_lines(lines, separator=",", tyeps=(), debug=False):
    for line in lines:
        ...
        statements
        ...
```

现在，假设你想要创建一个特殊用途的函数，用于解析指定文件名对应的文件中的数据：

```Python
def prase_lines(filename, *args, **kwargs):
    with open(filename, 'rt') as file:
        return parse_lines(file, *args, **kwargs)
```

这样的好处之一是，函数 `prase_lines` 无序知道任何关于 `prase_lines` 函数的参数信息。
它接受调用者提供的任何额外参数并将其传递下去。
这也让函数 `prase_lines` 的维护更加简单了，如果要添加一个新字段，该函数无序修改也能工作。

## Positional Only Arguments

许多 Python 的内置函数值接受位置参数，可以使用 slash 斜杠 `/` 来指明。
例如 `func(x, y, /)`，这意味着在斜杠前面的参数只能通过位置参数指定。

这种定义并不常见，因为在 Python 3.8 才开始支持。
但这是一种有效的避免命名冲突的方法。
例如：

```Python
import time

def after(seconds, func, /, *args, **kwargs):
    time.sleep(seconds)
    return func(*args, **kwargs)


def duration(*, seconds, minutes, hours):
    return seconds 60 * minutes + 3600 * hours

after(5, duration, seconds=20, minutes=3, hours=2)
```

在上面例子总，`after` 有个参数 `seconds`，同样 `duration` 也有一样的参数名称，通过强制要求位置参数，避免了变量命名的冲突。
这里的 `seconds=20` 被解析成关键字参数，通过 `**kwargs` 传递给 `duration` 函数。

## Names, Documentation Strings, and Type Hints

标准的函数命名惯例使用小写字母和 undersocre 下划线作为分隔符。
如果一个函数不想被直接调用，比如内部的方法，使用单个下划线作为前置，例如 `_helper()`。

函数名称可以通过 `__name__` 属性获取，有时候调试非常有用

```Python
def square(x):
    return x * x
square.__name__  # 'square'
```

在函数的第一次声明的时候，使用文档字符串描述使用方式很常见，例如：

```Python
def factorial(n):
    """
    Computes n factorial. For example:

    >>> factorial(6)
    120
    >>>
    """
    if n <= 1:
        return 1
    else:
        return n * factorial(n - 1)
```

文档字符串存储在函数属性 `__doc__` 里，IDE 经常会访问作为交互使用。

函数也可以添加类型提示：

```Python
def factorial(n: int) -> int:
    if n <= 1:
        return 1
    else:
        return n * factorial(n - 1)
```

类型提示对函数的计算结果不会有任何影响，即没有任何额外的性能提示或运行时错误检查。
提示会存储在函数的 `__annotations__` 属性中，该属性是一个参数名称映射提示的字典，许多 IDE 和第三方工具会使用提示内容。

有时候提示会写在一个本地变量旁边，例如：

```Python
def factorial(int: n) -> int:
    result: int = 1  # Type hinted local variable
    while n > 1:
        result *= n
        n -= 1
    return result
```

这种提示也会被解释器忽略，同样该提示是提供给第三方代码检查工具使用的。

## Function Application and Parameter Passing

当函数被调用时，函数参数会成为局部名称，绑定到传入的输入对象。
Python 将传递的对象原样传递给函数，不进行任何额外的拷贝。
要小心如果可变类型作为参数传递，如果对象进行了任何修改，会影响到原来的对象。

例如这个例子：

```Python
def square(items):
    for i, x in enumerate(items):
        items[i] = x * x  # modify items in-place

s = [1, 2, 3, 4, 5]
suqare(s)  # Change a to [1, 4, 9, 16, 25]
```

会修改输入值的函数会产生“副作用”，一般的规则是避免这种情况。
这种函数也难以和多线程和并发程序交互，因为副作用通常会被锁保护。

当然，还要重点区分修改对象和重新分配 reassign 对象的区别，例如：

```Python
def sum_squares(items):
    items = [x*x for x in items]  # Reassign "items" name
    return sum(items)

a = [1, 2, 3, 4, 5]
result = sum_squares(a)
print(a)  # [1, 2, 3, 4, 5] (Unchanged)
```

在这个例子中，`sum_squares` 函数看上去重写了 `items` 变量。
没错，标签 `items` 被重新分配了新的值。
然而，`items` 原始的值并没有被操作修改，而是将标签 `items` 绑定到了不同变量上面，即内部的列表推导式的值。
赋值给一个变量和修改一个对象是有区别的，当给一个变量赋值的时候，并不会重写已经存在的对象。
而是在给不同的对象重命名。

从风格上来讲，会产生“副作用”的函数通常会返回 `None` 作为结果。
例如列表的 `sort()` 函数：

```Python
items = [10, 3, 2, 9, 5]
items.sort()  # Observe: no return value
items  # [2, 3, 5, 9, 10]
```

该 `sort()` 方法会将列表原地排序，并不返回值。
缺少返回值是一个强有力的“副作用”指标，在这个例子中列表被重新排序了。

有时候，你已经有值存储在序列 sqeuence 或字典 mapping 中了，要将他们传递给一个函数，在函数调用中使用 `*` 和 `**`：

```Python
def func(x, y, z):
    ...

# Pass a sequence as arguments
s = (1, 2, 3)
result = func(*s)

# Pass a mapping as keyword arguments
d = {"x": 1, "y": 2, "z": 3}
result = func(**d)
```

如果从多个源获取数据，甚至显示提供部分参数，只要函数获得所有必须的参数、不含重复项，且调用前面中的所有内容都正确对齐，一切都会正常运行。
甚至可以在函数调用中多次使用 `*` 和 `**`，但如果确实值，或参数重复，将会遇到一个报错。

## Return Values

`return` 语句返回一个函数值，如果省略了返回或没有值，则返回 `None`。
如果要返回多个值，将他们放到一个元组中：

```Python
def parse_value(text):
    """
    Split text of the form name=val into (name, val)
    """
    parts = text.split("=", 1)
    return (parts[0].strip(), parts[1].strip())
```

返回的元组可以这样拆开成单个值：

```Python
name, value = parse_value("url=https//www.python.org")
```

有时候可以使用 named tuples 具名元组替代：

```Python
from typing import NamedTuple

class ParseResult(NamedTuple):
    name: str
    value: str

def parse_value(text):
    """
    Split text of the form name=val into (name, val)
    """
    parts = text.split("=", 1)
    return ParseResult(parts[0].strip(), parts[1].strip())
```

具名元组和普通的元组一样，你可以进行同样的操作例如解包，但还可以通过名称属性来访问元素：

```Python
r: ParseResult = parse_value("url=http://www.python.org")
print(r.name, r.vlaue)
```

## Error Handling

上面的 `parse_value()` 函数没有错误处理，如果字符串不是预期的格式应该怎么办？
一种方式是将结果视为可选的，即函数要么返回结果，要么返回 `None`，这个常用于表示缺失值。
例如，该函数可以这样修改：

```Python
def parse_value(text):
    parts = text.split("=", 1)
    if len(parts) == 2:
        return ParseResult(parts[0].strip(), parts[1].strip())
    else:
        return None
```

使用这种设计，结果检测的重担就传递给了调用者：

```Python
result = parse_value(text)
if result:
    name, value = result
```

或者使用更紧凑的 Python 3.8 新语法：

```Python
if result := parse_value(text)
    name, value = result
```

除了返回 `None`，也可以遇到错误文本时抛出异常：

```Python
def parse_value(text):
    parts = tex.split("=", 1)
    if len(parts) == 2:
        return PraseResult(parts[0].strip(), parts[1].strip())
    else:
        raise ValueError("Bad value")
```

在这个情况下，调用者使用 `try-except` 语句来处理错误：

```Python
try:
    name, value = prase_value(text)
except ValueError:
    ...
```

是否要使用异常处理并不总是明确的。
一般而言，异常是更常见的异常处理方式。
但异常处理会更加昂贵，如果在编写一些性能要求高的代码，应该返回 `None`, `False`, `-1` 或者其他的特殊值来表示错误。

## Scoping Rules

每次执行一个函数，都会创建一个命名空间。
这个命名空间代表一个环境，包含了函数参数的名称和值，以及函数体内赋值的所有变量。
函数内部赋值的变量是局部变量，直接引用但未赋值的变量会到函数定义所在的模块的全局作用域中查找。

有两类和命名相关的错误，在全局环境查找一个未定义的自由变量会产生 `NameError` 异常。
查询一个未赋值的本地变量会产生一个 `UnboundLocalError` 异常，这种错误通常源于控制流缺陷。

比如这样：

```Python
def func(x):
    if x > 0:
        y = 42  # y not assigned if conditional is false
    return x + y
func(10)
func(-10)  # UnboundLocalError: y referenced before assigment
```

或者是原地计算产生的错误：

```Python
def func():
    n += 1  # Error: UnboundLocalError
```

需要强调的是，变量名的作用域永远不会改变，要么是全局变量，要么是局部变量，这取决于函数定义时的设定。

```Python
x = 42
def func():
    print(x)  # UnboundLocalError
    x = 12
```

上面例子看上去好像能正常运行，但实际上由于下面 `x = 12` 的赋值，x 实际上是局部变量，且还未定义，因此会报错。

如果要在函数内修改外部变量，使用 `global` 关键字：

```Python
x = 42
y = 37
def func():
    global x
    x = 13
    y = 0

func()  # x is 13, y is sitll 37
```

但使用 `global` 并不是特别符合 Python 风格。
如果要通过函数修改状态，考虑使用类定义，并通过修改实例变量或类变量来实现状态变化。

```Python
class Config:
    x = 42

def func():
    Config.x = 13
```

Python 允许嵌套函数定义，例如：

```Python
def countdown(start):
    n = start

    def display():  # Nested function definition
        print("T-minus", n)
    while n > 0:
        display()
        n -= 1
```

嵌套函数中的变量通过词法作用域绑定。
也就是说，名称首先在局部作用域解析，然后从最内层作用域到最外层作用域，依次在连续的封闭作用域中解析。
同样，这个过程不是动态的，名称的绑定是在函数定义时根据语法一次性确定的。

例如，下面这段代码不会工作：

```Python
def countdown(start):
    n = start
    def display():
        print("T-minus", n)
    def decrement():
        n -= 1  # UnboundLocalError: 局部变量在赋值前被引用
    while n > 0:
        display()
        decrement()
```

要修复这段代码，使用 `nonlocal` 关键字，声明该变量来自外层函数。

```Python
def countdown(start):
    n = start
    def display():
        print("T-minus", n)
    def decrement():
        nonlocal n
        n -= 1
    while n > 0:
        display()
        decrement()
```

`nonlocal` 不能用于引用全局变量，它必须指向外层作用域中的局部变量。
因此，如果一个变量是全局的，还是应该使用 `global` 关键字。

## Recursion

Python 支持递归函数，例如：

```Python
def sumn(n):
    if n == 0:
        return 0
    else:
        return n + sumn(n- 1)
```

但递归深度是有限制的，函数 `sys.getrecursionlimit()` 返回当前最大的递归深度，使用函数 `sys.setrecursionlimit()` 可以修改该值。
尽管可以改变递归深度，但栈的大小仍然受操作系统的限制。
当超出最大递归深度后，会抛出一个 `RuntimeError`。
如果限制递归深度太高，Python 可能会因 segmentation error 段错误或其他操作系统错误而崩溃。

在实践中，一般递归限制只会在使用树或图这类深度嵌套数据结构的时候，才会导致递归深度的问题。

## The lambda Expression

一个匿名函数可以使用 `lambda` 关键字定义：

```Python
lambda args: expression
```

args 是一个由逗号分隔的参数列表，而 expression 则是这些参数的表达式，例如：

```Python
a = lambda x, y: x + y
r = a(2, 3)  # r gets 5
```

使用 lambda 定义的代码必须是一个合理的表达式，多个语句以及其他非表达式语句，例如 try 和 while，不能出现在 lambda 表达式中。
lambda 表达式也遵循和函数一样的作用域。

一个使用 lambda 的例子是定义小的回调函数，例如你可能在 `sorted()` 函数中这样使用：

```Python
result = sorted(words, key=lambda word: len(set(word)))  # len(set(str)) 是不重复字符数量
```

如果 lambda 中含有自由变量则需要注意：

```Python
x = 2
f = lambda y: x * y
x = 3
g = lambda y: x * y

print(f(10))  # 30
print(g(10))  # 30
```

函数 `f(10)` 使用调用时 x 的值，而不是函数定义时 x 的值，这种行为被称为 late-binding 延迟绑定。

在定义时绑定值使用 default arguemnt 默认参数：

```Python
x = 2
f = lambda y, x=x: x * y
x = 3
g = lambda y, x=x: x * y

print(f(10))  # 20
print(g(10))  # 30
```

这是因为 default argument 默认参数值仅在函数定义时被求值，因此会捕获变量x的当前值。

## Higher Order Functions

Python 支持 "higher-order functions" 高阶函数的概念，这意味着函数可以作为参数传递给其他函数，存储数据结构中，并作为结果返回。
有时候会说函数是 "first-class" 一等公民，意味着函数和其他数据类型没有区别。

```Python
import time

def after(seconds, func):
    time.sleep(seconds)
    func()

def greeting():
    print("Hello World")

after(10, greeting)
```

在这个例子中，`after()` 函数的 `func` 参数就是所谓的 "callback function" 回调函数，这是指 after 会回调作为参数的函数。

当函数作为数据传递的时候，它会显示地携带函数定时时，关于环境的信息。
例如下面这个例子，`greeting()` 函数是这样使用变量：

```Python
def main():
    name = "Guido"
    def greeting():
        print("Hello", name)
    after(10, greeting)  # Hello Guido

main()
```

在这个例子中，变量 `name` 被 `greeting()` 调用，但 `name` 是 `main` 函数的本地变量。
当 `greeting` 传递给 `after()`，该函数会记得其环境，并使用环境中的 name 变量。
该特性被称为 "closure" 闭包。

**Closure** **闭包**是一个函数及其执行函数体所需的变量所构成的环境。
闭包与嵌套函数在编写基于惰性或延迟求值的概念的代码时尤其有用。
如上面的 `after()` 函数，它不会接收一个立刻执行的函数，该函数会在之后的某个时间点才被调用，该编程模式在其他场景中也很常见。

例如，程序可能仅在响应事件时才执行的函数（如按键、鼠标移动、网络数据包到达等）。
在所有这些情况下，函数求值会延迟到发生某些有意义的事件时再进行。
当函数最后执行的时候，闭包确保函数能获取到它所需要的一切。

你也可以这样编写和创建函数，例如：

```Python
def make_greeting(name):
    def greeting():
        print("Hello", name)
    return greeting

f = make_greeting("Guido")
g = make_greeting("Ada")
f()  # Hello Guido
g()  # Hello Ada
```

在这个例子中，`make_greeting()` 函数并不会携带有任何有意义的计算。
相反，它创建并返回了一个真正执行任务的 `greeting()` 函数。
这种情况只会在函数被求值后发生。

在上面例子中，`f` 和 `g` 函数是两个版本的 `greeting()` 函数。
尽管创建这些函数的 `make_greeting()` 函数已不再执行，但 `greeting()` 函数仍能记住已定义的 `name` 变量，这是函数闭包的一部分。

关于闭包需要注意的一点是，对变量名的绑定并非“快照”，而是一个动态过程。
这意味着，闭包指向的是变量名及其最近被赋予的值。
下面示例说明了可能引发的情况：

```Python
def make_greetings(names):
    funcs = []
    for name in names:
        funcs.append(lambda: print("Hello", name))
    return funcs

a, b, c = make_greetings(["Guido", "Ada", "Margaret"])
a()  # Hello Margaret
b()  # Hello Margaret
c()  # Hello Margaret
```

在这个例子中，列表中不同的函数使用 lambda 构造，他们看起来似乎都使用了唯一的 name 值。
但实际上是，所有函数都会使用同一个 name 值，无论该值是在外部 `make_greetings()` 函数返回时如何设置的。
因为 `lambda` 函数中的 `name` 变量是一个自由变量，它会在函数执行时才查找值，而不是在定义时（for 不会创建新的作用域）。

正确的方式可以使用默认参数进行捕获：

```Python
def make_greetings(names):
    funcs = []
    for name in names:
        func.append(lambda name=name: print("Hello", name))
    return funcs

a, b, c = make_greetings(["Guido", "Ada", "Margaret"])
a()  # Hello Guido
b()  # Hello Ada
c()  # Hello Margaret
```

或者这样

```Python
def make_greetings(names):
    funcs = []
    for names in names:
        def greeting(name=name):
            print("Hello", name)
        funcs.append(greeting)
    return funcs
```

## Argument Passing in Callback Functions

使用 callback function 回调函数时，一个具有挑战性的编码问题是如何向提供的函数传递参数。
考虑之前的 `after()` 函数：

```Python
import time

def after(seconds, func):
    time.sleep(seconds)
    func()
```

在这个例子中，`func` 是硬编码且没有参数的。
如果你想要解析额外的参数，那就不好用了，例如：

```Python
def add(x, y):
    print(f"{x} + {y} -> {x + y}")
    return x + y
after(10, add(2, 3))  # Fails: add() called immediately
```

在这个例子中，在这个例子中，`add(2, 3)` 函数立即执行并返回结果 5。
随后，`after()` 函数在 10 秒后尝试执行 `5()` 时崩溃。

这显然不是这段代码的本意，这个问题暗示了一个更广泛的设计议题。
即关于函数及函数式编程的整体运用，特别是与函数组合相关的方面。
当函数通过各种方式混合在一起的时候，你通常需要思考函数的输入与输出如何相互连接。

这个例子的一种解决方案是，使用无参数 lambda 函数打包，例如：

```Python
after(10, lambda: add(2, 3))
```

这样的小型 zero-argument 零参数函数有时被称为 "thunk" (延迟计算或惰性求值)。
基本上，它代表一个表达式，该表达式将在最终被作为无参数函数调用时进行求值。
这可以作为一种通用方法，将任何表达式的求值延迟到稍后的时间点，将表达式放入 lambda 函数中，并在实际需要该值时调用该函数。

作为 lambda 的一种替代方案，可以使用 `functools.partial()` 来创建一个部分求值的函数：

```Python
from functools import partial

after(10, partial(add, 2, 3))
```

`partial()` 这里扮演一个函数工厂的角色，它不立刻计算结果，而是生成一个新的函数。
在未来需要被调用的时候，才去真实调用该函数。
这是一种有效的方法，可以使非标准函数在回调和其他应用中，匹配预期的调用签名。

下面是更多例子：

```Python
def func(a, b, c, d):
    print(a, b, c, d)

f = partial(func, 1, 2)
f(3, 4)    # func(1, 2, 3, 4)
f(10, 20)  # func(1, 2, 10, 20)

g = partial(func, 1, 2, d=4)
g(3)   # func(1, 2, 3, 4)
g(10)  # func(1, 2, 10, 4)
```

`partial()` 和 `lambda` 都有类似的目的，但两种方法在语义上有重大区别。
使用 `partial()` 时，参数会在偏函数首次定义时被求值并绑定；
而使用无参 `lambda` 时，参数的实际求值和绑定会延迟至该 `lambda` 函数后续真正执行时才发生（求值都被延迟了）。

```Python
def func(x, y):
    return x + y

a = 2
b = 3
f = lambda: func(a, b)
g = partial(func, a, b)

a = 10
b = 20
f()  # 使用当前的 a=10, b=20 的值
g()  # 使用初始的 a=2, b=3 的值
```

由于 partials 偏函数会计算出函数结果，由 `partial()` 创建的可调用对象是可以序列化为字节、保存到文件中，甚至通过网络传输的对象。
但使用 `lambda` 函数就不可能实现，因此在需要传递函数，尤其是可能传递给不同进程或不同机器上的 Python 解释器的应用场景中，`partial()` 根据适应性。

部分函数应用和一种 "currying" 柯里化的应用密切相关。
柯里化是一种函数式编程的技术，将多参数函数表示为单参数函数的嵌套函数。

```Python
# 3 参数函数
def f(x, y, z):
    return x + y + z

# Curried version
def fc(x):
    return lambda y: (lambda z: x + y + z)

a = f(2, 3, 4)
b = fc(2)(3)(4)
```

回到之前的参数传递问题，另一种传递参数的方法是将其视作外部调用函数的独立参数来接收。

```Python
def after(seconds, func, *args):
    time.sleep(seconds)
    func(*args)

after(10, add, 2, 3)  # 10 秒后调用 add(2, 3)
```

你会发现，在 func() 中传递关键字参数是不被允许的。
这就是这样设计的，因为如果允许使用关键字参数，则可能会和 `after` 的参数 seconds 产生冲突。
关键字参数可能被保留用于指定 `after()` 函数本身的选项，例如：

```Python
def after(seconds, func, *args, debug=False):
    time.sleep(seconds)
    if debug:
        print("About to call", func, args)
    func(*args)
```

然而并非全无希望，如果需要为 `func()` 指定关键字参数，仍然可以通过 partial() 实现，例如：

```Python
after(10, partial(add, y=3), 2)
```

如果想要让 `after()` 接受关键字参数，一个安全的方式是只允许使用位置参数传递值，例如：

```Python
def after(seconds, func, debug=False, /, *args, **kwargs):
    time.sleep(seconds)
    if debug:
        print("About to call", func, args, kwargs)
    func(*args, **kwargs)

after(10, add, 2, y=3)
```

另一个让人不安的洞见是，`after()` 的参数实际上是两个不同函数的参数合并。
或许传递参数的问题可以分解为两个函数，例如这样：

```Python
def after(seconds, func, debug=False):
    def call(*args, **kwargs):
        time.sleep(seconds)
        if debug:
            print("About to call", func, args, kwargs)
        func(*args, **kwargs)
    return call

after(10, add)(2, y=3)
```

现在，`after()` 函数的参数与 `func` 函数的参数之间没有任何冲突。
然而，这样做可能会和同事之间发生冲突 XD ...

## Returning Results from Callbacks

另一个没有说的问题是关于返回计算结果，例如 `aftre()` 函数：

```Python
def after(seconds, func, *args):
    time.sleep(seconds)
    return func(*args)
```

在异常处理的时候有两种情况：

```Python
after("1", add, 2, 3)  # Fails: TypeError (integer is excepted)
after(1, add, "2", 3)  # Fails: TypeError (can't concatenate int to str)
```

上例中，两个都抛出 `TypeError`，但原因却不同。
第一个是 `after()` 函数导致的错误，第二个是 `func()` 函数导致的错误。
一个解决思路是将 call function 回调函数中的错误以不同方式封装，使其能够与其他类型的错误分开处理。

例如：

```Python
class CallbackError(Exception):
    pass

def after(seconds, func, *args):
    time.sleep(seconds)
    try:
        return func(*args)
    except Exception as err:
        raise CallbackError("Callback function failed") from err
```

这段修改后的代码将来自所提供回调函数的错误隔离到其自身的异常类型中：

```Python
try:
    r = after(delay, add, x, y)
except CallbackError as err:
    print("It failed, Reason", err.__cause__)
```

如果 `after()` 本身的执行出现问题，该异常将未经捕获地向外传播。
另一方面，与所提供回调函数实际执行相关的问题会被捕获并报告为 CallbackError。
这一切都相当微妙，在实践中，异常处理很难。
这种方式让责任归属更加明确，同时 `after()` 的行为也更容易记录。
如果回调函数出现问题，则总是报告为 `CallbackError`。

另一种方法是将结果包装到实例中去，包含了结果和错误信息：

```Python
class Result:
    def __init__(self, value=None, exc=None):
        self._value = value
        self._exc = exc
    def result(self):
        if self._exc:
            raise self._exc
        else:
            return self._value
```

然后使用这个类作为 `after()` 函数的返回值

```Python
def after(seconds, func, *args):
    time.sleep(seconds)
    try:
        return Result(value=func(*args))
    except Exception as err:
        return Result(exc=err)

r = after(1, add, 2, 3)
print(r.result())  # 5

s = after("1", add, 2, 3)  # TypeError

t = after(1, add, "2", 3)  # Return a "Result"
print(t.result())
```

第二种方法的工作原理是将回调函数的结果推迟到一个独立的步骤中。
如果 `after()` 出现问题，会立刻报告；
而如果回调函数 `func()` 出现问题，则会在用户尝试通过调用 `result()` 方法获取结果时报告。

这种将结果封装在特定实例中，以便后续解包的编程风格在现代编程语言中正变得越来越普遍。
一个这样做的动机是因为能够促进类型提示，例如：

```Python
def after(seconds, func, *args) -> Result:
    ...
```

尽管在大多数 Python 代码中，这类模式并不常见，但在处理线程和进程等并发原语时，会频繁出现。
例如，在使用线程池时，所谓的 Fruture 实例就表现出这种行为：

```Python
from concurrent.futures import ThreadPoolExecutor

pool = ThreadPoolExecutor(16)
r = pool.submit(add, 2, 3)  # Returns a Future
print(r.result())           # Unwarp the Future result
```

## Decorators

装饰器是一个函数，它能够为另一个函数创建包装器。
包装的主要目的是为了修改或增强被包装对象的行为。
语法上，装饰器使用 `@` 符号来表示：

```Python
@decorate
def func(x):
    ...
```

前面代码是下面的简写形式：

```Python
def func(x):
    ...

func = decorate(func)
```

在这个例子中，定义了函数 `func()`，但马上将该函数作为参数传递给 `docorate()`，并返回一个新的对象替代掉原始的 `func` 函数。

下面看一个具体的例子，使用 `@trace` 装饰器为函数添加调试信息：

```Python
def trace(func):
    def call(*args, **kwargs):
        print("Calling", func.__name__)
        return func(*args, **kwargs)
    return call

# Example
@trace
def square(x):
    return x * x
```

这段代码中，`trace` 创建了一个装饰器，输出函数的调试信息，然后再调用函数。
看上去十分简单，但在实际中，函数还会含有一些元信息，例如函数名称、doc string 和 type hints 类型提示等。
如果直接使用上面的函数来包装，会将这些信息隐藏。
因此，编写装饰器时常用 `@wraps()` 装饰器，例如：

```Python
from functools import wraps

def trace(func):
    @wraps(func)
    def call(*args, **kwargs):
        print("Calling", func.__name__)
        return func(*args, **kwargs)
    return call
```

`@wraps()` 装饰器会将函数的元数据复制到要替换的函数上，在这个例子中 `func()` 函数的元数据被复制到了 `call()` 包装器函数上。

使用装饰器时，其必须放在函数的上面一行，一个函数可以应用更多的装饰器：

```Python
@docorator1
@docorator2
def func(x):
    pass
```

在这个例子中，装饰器等同于这样：

```Python
def func(x):
    pass

func = docorator1(decorator2(func))
```

装饰器的顺序可能会很重要。
例如在类定义中，`@classmethod` 和 `@staticmethod` 总是需要放到最外层。

例如：

```Python
class SomeClass(object):  # Yes
    @calssmethod
    @trace
    def a(cls):
        pass

    @trace
    @classmethod
    def a(cls):           # No. Fails
        pass
```

这种放置限制的原因与 `@classmethod` 返回的值有关。
有时装饰器会返回一个不同于普通函数的对象。
如果最外层装饰器未预料到这种情况，则会崩溃。
在这种情况下，`@classmethod` 会创建一个类方法描述符对象。
除非 `@trace` 装饰器在设计时已考虑到这一点，否则装饰器顺序不当会使装饰器失效。

装饰器也能够接受参数，例如，修改 `@trace` 装饰器来支持自定义信息：

```Python
@trace("You called {func.__name}")  # f-string 是立刻求值的，不能在这里使用
def func():
    pass
```

提供参数的装饰器语义如下：

```Python
def func():
    pass

# Create the decorator function
temp = trace("You called {func.__name__}")

# Apply it to func
func = temp(func)
```

在这种情况下，接受参数的最外层函数负责创建装饰器函数。
随后，该函数会与待装饰的函数一同调用，以获取最终结果。

以下是装饰器可能实现的样子：

```Python
from functools import wraps

def trace(message):      # message: 装饰参数
    def decorate(func):  # func: 装饰函数
        @wraps(func)
        def wrapper(*args, **kwargs):
            print(message.format(func=func))  # str.format() 延迟求值
            return func(*args, **kwargs)
        return wrapper
    return decorate
```

这种实现的一个有趣特性是，外层函数实际上是一种 "decorator factory" 装饰器工厂。

假如这样写代码：

```Python
@trace("You called {func.__name__}")
def func1():
    pass

@trace("You called {func.__name__}")
def func2():
    pass
```

但这样会显得很繁琐，你可以通过调用外部装饰器函数一次，并重复使用其结果来简化：

```Python
logged = trace("You called {func.__name__}")

@logged
def func1():
    pass

@logged
def func2():
    pass
```

装饰器不一定要替换原始函数。
有时，装饰器仅执行诸如注册之类的操作。
例如，如果你正在构建一个事件处理器的注册列表，你可能希望定义一个这样工作的装饰器：

```Python
@eventhandler("BUTTON")
def handle_button(msg):
    ...

@eventhandler("RESET")
def handle_reset(msg):
    ...
```

下面是管理装饰器定义:

```Python
# Event handler decorator
_event_handlers = { }
def eventhandler(event):
    def register_function(func):
        _event_handlers[event] = func
        return func
    return register_function
```

## Map, Filter, and Reduce

熟悉函数式编程的程序员可能想知道常见的列表操作，如 map 映射, filter 过滤和 reduce 归约。
多部分这些功能都可以对列表推导式和生成器表达式有效：

```Python
def square(x):
    return x * x

nums = [1, 2, 3, 4, 5]
squares = [square(x) for x in nums] # [1, 4, 9, 16, 25]
```

你甚至不需要这个单行函数：

```Python
squares = [x * x for x in nums]
```

其中也可以进行过滤 filtering

```Python
a = [x for x in nums if x > 2]  # [3, 4 ,5]
```

内置函数 `map()`，其功能与使用生成器表达式映射函数相同。

```Python
squares = map(lambda x: x*x, nums)
for n in squares:
    print(n)
```

该内置函数会创建一个过滤值的生成器：

```Python
for n in filter(lambda x: x > 2, nums):
    print(n)
```

如果要累加或减去值，使用 `functools.reduce()`，例如：

```Python
from functools import reduce

nums = [1, 2, 3, 4, 5]
total = reduce(lambda x, y: x + y, nums)  # 15
product = reduce(lambda x, y: x * y, nums, 1)  # 120

pairs = reduce(lambda x, y: (x, y), nums, None)  # (((((None, 1), 2), 3), 4), 5)
```

`reduce()` 接受一个两个参数的函数，一个可迭代对象和一个初始值。
从左到右对提供的可迭代对象jinx值累积，这有时被称为 "left-fold" 左折叠操作。

下面是伪代码：

```Python
def reduce(func, items, initial):
    result = initial
    for item in items:
        result = func(result, item)
    return result
```

经验表明，使用 `reduce()` 有时会令人赶到困惑。
此外，常见的归约操作，如 `sum()`、`min()` 和 `max()` 已内置在语言中。
使用这些内置操作比 `reduce()` 更加容易理解。

## Function Introspection, Attributes, and Signatures

如前所述，函数也是对象，这意味着他们可以被复制给变量、存储在数据结构中，并可以像程序中其他类型数据一样使用。
他们也可以通过多种方式进行查看。
下表展示了函数的一些常见属性，其中许多属性在调试、日志记录以及其他涉及函数的操作中非常有用。

| Attribute           | Description                             |
| ------------------- | --------------------------------------- |
| `f.__name__`        | Function name                           |
| `f.__qualname__`    | Fully qualified name                    |
| `f.__module__`      | Name of module in which defined         |
| `f.__doc__`         | Documentation string                    |
| `f.__annotations__` | Type hints                              |
| `f.__globals__`     | Dictionary that is the global namespace |
| `f.__closure__`     | Closure variables                       |
| `f.__code__`        | Underlying code object                  |

- `f.__qualname__` 是包含嵌套格式的完整路径名 `外层类名.内层类名.方法名` 或 `外层函数.内层函数名`
- `f.__module__` 是一个包含模块名称的字符串
- `f.__globals__` 是一个字典，包含函数定义时的全局命名空间，类似这样
  ```Python
  {
    '__name__': '__main__',
    '__doc__': None,
    '__package__': None,
    '__loader__': <...>,
    '__spec__': None,
    '__annotations__': {},
    '__builtins__': <module 'builtins'>,
    '__file__': 'test.py',
    'x': 10,
    'f': <function f at 0x...>
    # ... 还有很多内置属性
  }
  ```
- `f.__closure__` 保存了嵌套函数中闭包变量的引用值

  这可能不太显眼，例如：

  ```Python
  def add(x, y):
      def do_add():
          return x + y
      return do_add()

  a = add(1, 2)

  a.__closure__
  # (
  #   <cell at 0x1072ae200: int object at 0x105a6dc48>,
  #   <cell at 0x1072af550: int object at 0x105a6dc68>
  # )

  a.__closure__[0].cell_contents # 2
  ```

- `f.__code__` 对象代表编译过的函数体字节码

  ```Python
  a.__code__
  # <code object do_add at 0x107062b40, file "<ipython-input-3-ae9516cf2be1>", line 2>
  ```

函数还可以添加任意属性，例如

```Python
def func():
    statements
func.secure = 1
func.private = 1
```

属性在函数体内不可见，他们不是局部变量，也不会在执行环境中以名称形式出现。
使用函数属性的主要方式，是存储元数据。
有时框架或各种元编程技术会利用函数标记。
一个例子是抽象基类中方法所使用的 `@asbtractmethod` 装饰器，该装饰器只为函数添加属性：

```Python
def abstractmethod(func):
    func.__isabstractmethod__ = True
    return func
```

其他一些代码会寻找这个属性，并利用它来为示例创建过程添加额外的检查。

若想深入了解某个函数的参数信息，可以通过调用 `inspect.signature()` 函数获取其签名。

```Python
import inspect

def func(x: int, y: float, debug=False) -> float:
    pass
sig = inspect.signature(func)
```

签名对象提供了许多便捷功能，用于打印和获取参数的详细信息，例如：

```Python
# Print out the signature in a nice form
print(sig)  # (x: int, y: float, debug=False) -> float

# Get a list of
print(list(sig.parameters))  # [ 'x', 'y', 'debug']

# Iterate over the parameters and print various metadata
for p in sig.parameters.values():
    print("name", p.names)
    print("annotation", p.annotation)
    print("kind", p.kind)
    print("default", p.default)
```

签名的描述函数属性的元数据，作为一条数据，签名有多种用途。

对签名的一个有用操作是比较，例如可以通过一下方式检查两个函数是否具有相同的签名：

```Python
def func1(x, y):
    pass

def func2(x, y):
    pass

assert inspect.signature(func1) == inspect.signature(func2)
```

这种比较在一些框架里面可能会很有用。
例如，一个框架可以通过签名对比检查你编写的函数或方法是否符合预期的原型。

如果存储在函数的 `__signature__` 属性中，签名将在帮助信息中显示，并在后续使用 `inspect.signature` 时返回。

例如：

```Python
def func(x, y, z=None):
    ...

func.__signature__ = inspect.signature(lambda x, y: None)
```

在这个例子中，可选参数 `z` 在检查 `func` 是被隐藏。
相反，附加的签名将由 `inspect.signature()` 返回。

## Environment Inspection

可以使用内置方法 `globals()` 和 `locals()` 来查看函数的执行环境。
`globals()` 返回当前作为全局命名空间使用的字典，和属性 `func.__globals__` 属性相同。
这通常就是保存外层模块内容的那个字典。
`locals()` 返回一个包含本地变量和闭包变量的字典，该字典并非真实保存这些变量的数据结构。
修改 `locals()` 字典中的某个条目不会影响底层变量，例如：

```Python
def func:
    y = 20
    locs = locals()
    locs["y"] = 30
    print(locs["y"])  # 30
    print(y)          # 20
```

如果想要改变变量，需要将其拷贝回来

```Python
def func:
    y = 20
    locs = locals()
    locs["y"] = 30
    y = locs["y"]
```

一个函数可以使用 `inspect.currentframe()` 获取其栈帧。
一个函数可以通过沿着栈帧的 `f.f_back` 属性追踪栈跟踪，从而获取其调用者的栈帧。

```Python
import inspect

def spam(x, y):
    z = x + y
    grok(z)

def grok(a):
    b = a * 10
    print(inspect.currentframe().f_locals)  # {'a': 5, 'b': 50}
    print(inspect.currentframe().f_back.f_locals)  # {'x': 2, 'y': 3, 'z': 5}

spam(2, 3)
```

后时候，你会看到使用 `sys._getframe()` 来获取栈帧

```Python
import sys
def grok(a):
    b = a * 10
    print(sys._getframe(0).f_locals)  # myself
    print(sys._getframe(1).f_locals)  # my caller
```

下表一些常用的帧属性

| Attribute    | Description                                                              |
| ------------ | ------------------------------------------------------------------------ |
| f.f_back     | Previous stack frame (toward the caller)                                 |
| f.f_code     | Code object being executed                                               |
| f.f_locals   | Dictionary of local variables (locals())                                 |
| f.f_globals  | Dictionary used for global variables (globals())                         |
| f.f_bulltins | Dictionary used for built-in names                                       |
| f.f_lineno   | Line number                                                              |
| f.f_lasti    | Current instruction. This is an index into the bytecode string of f_code |
| f.f_trace    | Function called at start of each source code line                        |

查看帧段是一直调试和代码检查的有效方法，例如下面例子，可以查看调用者函数选中的变量值：

```Python
import inspect
from collections import ChainMap

def debug(*varnames):
    f = inspect.currentframe().f_back
    vars = ChainMap(f.f_locals, f.f_globals)
    print(f"{f.f_code.co_filename}:{f.f_lineno}")
    for name in varnames:
        print(f"    {name} = {vars[name]!r}")

# Example use
def func(x, y):
    z = x + y
    debug("x", "y")
    return z
```

## Dynamic Code Execution and Creation

`exec(str, [, globals [, locals]])` 函数执行一条含有任意代码的字符串，提供给 `exec()` 的代码会被执行，就好像代码实际上出现在 exec 操作的位置一样。

例如：

```Python
a = [3, 5, 10, 13]
exec("for i in a: print(i)")
```

传递给 `exec()` 的代码会在调用者的本地和全局命名空间中执行。
然而，要注意对本地变量的修改不会有效果。

```Python
def func():
    x = 10
    exec("x = 20")
    print(x)  # 10
```

这是因为，`locals` 只是一个收集了局部变量的快照字典 `{'x': 20}`，而不是实际的局部变量本身。
`globals` 返回的是模块的全局字典，模块级变量确实就是存在字典里。

`exec()` 还可以接受 1 或 2 个字典对象，分别作为要执行代码的全局和本地命名空间。例如：

```Python
globals = {
    "x": 7,
    "y": 10,
    "birds": ["Parrot", "Swallow", "Albatros"],
}

locs = { }

# 使用上面字典中定义的全局、本地命名空间
exec('z = 3 * x + 4 * y', globs, locs)
exec('for b in birds: print(b)', globs, locs)
```

省略参数情况见下表

| 情况           | globals          | locals     | 读              | 写         |
| -------------- | ---------------- | ---------- | --------------- | ---------- |
| 都提供         | globs            | locs       | 先 locs → globs | 写入 locs  |
| 只提供 globals | globs            | globs      | globs           | 写入 globs |
| 只提供 locals  | 当前模块 globals | locs       | locs → globals  | 写入 locs  |
| 都省略         | 当前作用域       | 当前作用域 | 当前作用域      | 当前作用域 |

一种常见的动态代码执行是函数和方法的创建。
例如，这里有个函数创建了 `__init__()` 方法为类添加一系列名称：

```Python
def make_init(*name):
    parms = ','.join(names)
    code = f"def __init__(self, {parms}):\n"
    for name in names:
        code += f" self.{name} = {name}\n"
    d = { }
    exec(code, d)
    return d["__init__"]

# Example use
class Vector:
    __init__ = make_init('x', 'y', 'z')

# 等价于
class Vector:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
```

该技术在标准库的很多地方都使用了，例如 `namedtuple()`, `@dataclass` 和依赖动态代码创建的类似 feature。

## Asynchronous Functions and Await

Python 提供了一些执行异步代码的特性，包括所谓的 "async" 函数 (coroutines) 或 awaitables。
他们大多和并发以及 `asyncio` 库相关，但有一些库构建在这之上。

一个异步函数或叫协程函数，定义方式是在函数前面添加 `async` 关键字：

```Python
async def greeting(name):
    print(f"Hello {name}")
```

调用该函数并不会执行，而会得到一个 coroutine 协程对象，例如：

```Python
greeting("Guido")
# <coroutine object greeting at 0x101476dc8>
```

要运行函数必须在其他代码的监督下执行

```Python
import asyncio
asyncio.run(greeting("Guido"))  # Hello Guido
```

这个例子展示了异步函数最重要的特性，即他们永远不会自己执行。
他们的运行总是需要某种管理器或代码库的介入。
虽然不一定像示例中的那样使用 asyncio，但异步函数的执行始终离不开某种外部机制的调度。

除了被管理之外，异步函数的执行方式和其他 Python 函数相同。
语句按顺序执行，所有常见的控制流特性均使用。
如果要返回结果，使用常见的 `return` 就行：

```Python
async def make_greeting(name):
    return f"Hello {name}"
```

外部用于执行异步函数的 `run()` 函数会给返回所给定的返回值。
例如：

```Python
import asyncio
a = asyncio.run(make_greeting("Paula"))  # Hello Paula
```

异步函数可以使用 `await` 调用其他异步函数：

```Python
async def make_greeting(name):
    return f"Hello {name}"

async def main():
    for name in ["Paula", "Thomas", "Lewis"]:
        a = await make_greeting(name)
        print(a)

asyncio.run(main())
```

`await` 的使用仅在封闭的异步函数定义中有效，它也是确保异步函数执行的必要部分。
如果省略 `await` 代码会出错，该关键字也引出了一个函数染色问题。
即无法直接从非异步函数中，调用异步函数来编写代码。

在同一应用程序中将异步与非异步功能结合使用时，可能会涉及相当大的复杂性。
尤其是在考虑涉及高阶函数、回调和装饰器等编程技巧时。
大多数情况下，对异步函数的支持必须作为特殊情况来构建。

Python 在迭代器和上下文管理器协议就是这样做的。
例如，一个异步上下文管理器可以使用 `__aenter__()` 和 `__aexit__()` 方法在类上定义：

```Python
class AsyncManager(object):
    def __init__(self, x):
        self.x = x

    async def yow(self):
        pass

    async def __aenter__(self):
        return self

    async def __aexit__(self, ty, val, tb):
        return self
```

注意这些方法是 async 异步函数，并可以使用 `await` 执行其他 async 函数。
如果要使用该管理器，必须在异步函数内使用 `async with` 语法：

```Python
async def main():
    async with AsyncManager(42) as m:
        await m.yow()
```

一个类可以使用 `__aiter__()` 和 `__anext__()` 来定义异步生成器，这些被 `async for` 语句和异步函数使用。

## Final Words: Thoughts on Functions and Compoistion

任何语言都是由组件组合构建而成的。
在 Python 中，这种组合包括多种库和对象。
然而，一切的基础都是函数。
函数是系统构建的粘合剂，也是数据流动的基本机制。
