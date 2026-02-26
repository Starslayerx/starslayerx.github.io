+++
date = '2026-01-13T8:00:00+08:00'
draft = false
title = 'Python Tricks Part 8: Modules and Packages'
categories = ['Note']
tags = ['Python']
+++

Python 程序由 modules 和 packages 组成，使用 `import` 语句导入。

## Modules and the import Statement

任何 Python 源文件都可以作为一个模块导入，例如下面 `module.py` 代码：

```Python
a = 37

def func():
    print(f'func says that a is {a}')

class SomeClass:
    def method(self):
        print('method says hi')

print('loaded module')
```

改文件包含一些常见的编程元素，包括一个全局变量、一个函数、一个类定义 和 最后的语句。
通过下面方法导入：

```Python
import module

module.a
module.func()
s = module.SomeClass()
s.method()
```

执行 import 会发送下面这几件事：

1. 加载模块源码，如果找不到抛出 `ImportError`
2. 创建新模块对象。该对象作为模块内所有全局定义 global defintions 的容器，被称为 “命名空间” namespace
3. 该模块源码在新创建的模块命名空间内执行
4. 如果没有错误发生，调用者会创建一个名称，指向新的模块对象。该名称与模块名称一致，但不包含任何文件后缀。

这些步骤中，第一步是最复杂的。新手容易犯的错误就是使用错误的名称或将代码放到了未知的位置。
且模块文件必须放在 `sys.path` 所包含的文件路径中，且文件名称要遵循和 python 变量一样的规则。

剩下的步骤都隔离在一个模块中，因此不用担心不同模块间命名冲突的问题。
Python `import` 会执行所有导入的源码，因此导入上面模块会输出 `loaded module`。

如果要在单个 `import` 中导入多个模块，使用逗号将其分隔，例如：

```Python
import socket, os, re
```

有时候一个模块会使用 `as` 改变名称：

```Python
import module as mo
mo.func()
```

模块是一等公民，这意味着他们可以被复制给变量、填充数据结构和作为程序数据传递。

## Module Caching

模块的源码只会导入一次，无论使用多少次 `import` 语句。
一个新手常犯的错误就是，在交互式环境中，先导入了某个模块。
然后该模块更新了，然后重新 import 该模块，但该模块并不会更新。

可以在 `sys.modules` 里面找到缓存的模块，为一个字典，键为模块名称，值为模块对象。
如果将对应模块删除，则下一次 import 可以重新导入该模块。

有时导入语句会在一个模块内，例如：

```Python
def f(x):
    import math
    return math.sin(x) + math.cos(x)
```

看上去好像这样会导致速度很慢，即每次函数调用都会重新导入一次。
但实际上成本是很低的，只承担一次查询字典的开销。
这样写法主要是风格上的问题，一般倾向于将所有导入放到开头。
另一方面，如果你有一个特殊的很少使用的函数，将导入写在函数内部，可以加快程序导入速度。

## Importing Selected Names from a Module

`from module import name` 该语法将模块特定定义导入当前命名空间。
这和直接 import 不同的地方在于，不是创建一个新的命名空间，而是将该对象引用放入当前命名空间。

```Python
from module import func  # Imports module and puts func in current namespace

func()  # Calls module.func()

module.func()  # Fails. NameError: module
```

该 `from` 语句接受逗号分隔的名称列表，来导入多个定义。

```Python
from module import func, SomeClass
```

从语义上来说，`from module import name` 语句将名称从模块复制到本地命名空间。
Python 会在背后首先执行 `import module`，然后使用缓存进行赋值 `name = sys.modules['module'].name`。

一个常见的误解是人为使用 `from` 形式的导入更加高效，即只加载模块的一部分。
实际上并非如此，每当一个模块加载时，整个模块都会被加载并存储在缓存中。

使用 `from` 形式的 import 不会改变作用域。
当函数寻找变量的时候，它仍然会在定义中的文件中寻找，而不是导入或调用的函数中寻找。

```Python
from module import func

a = 42
func()  # func says that a is 37
func.__module__  # 'module'
func.__globals__['a']  # 37
```

使用星号 asterisk (`*`) 可以导入所有非下划线开始的定义：

```Python
# Load all definitions into the current namespace
from module import *
```

这种语法不能写在函数内部，否则会报错 `SyntaxError`。
因为这样会破坏 Python 编译器对局部变量 Local Variables 的优化机制。

模块可以通过定义 `__all__` 列表来精确控制通过 `from module import *` 导入的集和。

```Python
# module.py

__all__ = [ 'func', 'SomeClass' ]

a = 37  # Not exported

def func():  # Exported
    ...

class SomeClass:  # Exported
    ...
```

在实践中，使用 `from module import *` 是不被推荐的。
过度使用会导致混淆，并污染局部命名空间。

```Python
from math import *
from random import *
from statistics import *

a = gauss(1.0, 0.25)  # From which module ???
```

显示导入要好得多

```Python
from math import sin, cos, sqrt
from random import gauss
from statistics import mean

a = gauss(1.0, 0.25)
```

## Circular Imports

如果两个模块相互导入对方会产生特别的问题。
例如假设你有两个文件：

```Python
# moda.py
import modb

def func_a():
    modb.func_b()

class Base:
    pass
```

```Python
# modb.py
import moda

def func_b():
    print('B')

class Child(moda.Base):
    pass
```

这段对面存在一种奇怪的导入顺序问题。
具体是，使用 `import modb` 能正常工作，但如果使用 `import moda` 就会崩溃，并说 `moda.Base` 没有定义。

根据控制流，错误原因如下：

- 执行 `modb.py` 文件，Python 首先创建一个 `__name__ = __main__` 的模块
- 第一行为 `import moda`，会加载模块 `moda.py`
- 由于 `moda.py` 第一行为 `import modb`，这会导致又去加载 `modb`
- 由于模块缓存 `sys.module` 中没有 `modb` （而是 `__main__`），因此又会回到 `modb.py` 作为导入模块执行
- 这时候，`modb.py` 的第一行的导入存在模块缓存内，因此不会循环导入
- 当执行到 `Child` 的时候，由于缓存中创建的 `moda` 并没有加载完全，`moda.Base` 还未定义，从而导致报错

要修复这个问题，可以将 `import modb` 语句放到其他地方。
例如将其移动到 `func_a()` 函数内部：

```Python
# moda.py
def func_a():
    import modb
    modb.func_b()

class Base:
    pass
```

或者也可以将导入写在文件末尾

```Python
# moda.py
def func_a():
    modb.func_b()

class Base:
    pass

import modb  # Must after Base is defined
```

但这两者写法在代码审查中都可能被质疑。
大多数情况下，不会看到导入语句出现在文件末尾。
更好的处理方式一般是将 `Child` 类定义搬到单独的 `base.py` 文件中去。

```Python
# modb.py
import base

def func_b():
    print('B')

class Child(base.Base):
    pass
```

## Module Roloading and Unloading

目前没有可靠的机制来支持对先前导入的 modules（模块）进行 reloading（重新加载）或 unloading（卸载）。
即使可以通过删除 `sys.modules` 导入的模块，但这些内容仍然会保存在内存中。
这样因为其他导入的模块会缓存模块变量。
此外，如果模块中定义了类的实例，这些实例会包含指向其类对象的引用，而类对象又持有定义模块的引用。

模块引用存在于多个位置，这使得修改其实现后重新加载模块通常不切实际。
例如，如果从 `sys.modules` 里面删除模块，并通过 `import` 重新导入它，
但这并不会追溯性地改变程序中之前对该模块的所有引用。
相反，将会拥有最近 `import` 语句导入的一个新模块，和之前旧 `import` 代码保留的旧模块。

在 `importlib` 库中有一个 `reload()` 函数来重新导入模块。
将之前的模块作为参数导入该模块：

```Python
import module
import importlib
importlib.reload(module)  # loaded module <module 'module' from 'module.py'>
```

`reload()` 通过导入新的模块源码，并在已存在的模块命名空间顶部执行实现。

如果其他模块之前通过 `import module` 这种标准导入语句，则该函数可以当代码自动更新。

但这里存在很多危险：

1. 首先 reloading 不是递归的，它只会导入传递给 `reload()` 的单个模块
2. 如果导入模块使用 `from module import name` 的方式导入，则无法重新导入
3. 最后，如果创建了类的实例，则这些实例仍使用的是旧的类定义

此外，C/C++ 扩展无法通过任何方式取消或重新导入。

## Module Compliation

当模块首次导入时，他们会被编译为字节码。
该代码写在目录 `__pycache__` 的 `.pyc` 文件中，该目录通常和运行的 `.py` 代码目录下。
当程序的不同地方再次导入时，会直接加载编译后的代码，加快 import 速度。

如果在操作系统中 Python 没有权限创建文件，则也能正常工作，但 import 速度会慢很多。

另一个需要知道模块缓存的原因是要警惕可能影响它的编程技巧。
高级源编程涉及动态代码生成，和 `exec()` 函数会削弱字节码缓存的优势。

一个显著的例子是使用 dataclasses:

```Python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float
```

Dataclass 通过生成方法函数作为文本块并使用 `exec()` 执行。
所有这些生成的代码都没有通过 import 系统进行缓存。
对于单个 dataclass 可能注意不到，但如果有 100 个 dataclasses，就会发现他们比正常的类慢 20 倍。

## The Module Search Path

当导入模块的时候，解释器搜索 `sys.path` 里的路径。
第一个值通常是空字符串 `''`，指代当前工作目录。
如果运行一个 py 脚本，则第一个值是当前脚本所在的目录。
其他的值通常包括一些列目录名，和 `.zip` 文件。
`sys.path` 的顺序即模块搜索顺序。
如果要添加新的搜索路径，将其添加到该列表中。
可以通过设置 `PYTHONPATH` 环境变量来实现。

```bash
env PYTHONPATH=/some/path
```

ZIP 文件提供了一种将多个模块放到一个文件中的方法。
例如，假如创建了两个模块 `foo.py` 和 `bar.py`，并将他们放到 `mymodules.zip` 中。
则该文件可以添加到 Python 路径中：

```Python
import sys
sys.path.append('mymodules.zip')
import foo, bar
```

压缩文件内部的路径也可以加入 path:

```Python
sys.path.append('/tmp/modules.zip/lib/python')
```

并且压缩文件不一定要有 `.zip` 后缀，历史上曾经使用 `.egg`，该后缀源自于一个早期 python 包管理工具 setuptools。
但 `.egg` 文件只不过是在 `.zip` 之上添加了一些元数据罢了（版本号、依赖等）。

## Execution as the Main Program

Python 文件经常作为脚本运行，例如：

```bash
python3 module.py
```

每个模块定义一个变量 `__name__`，其中包括了模块名。
代码可以检查该变量来确定执行的模块。
在命令行中指定的程序会在 `__main__` 模块中运行。
有时候程序会根据是否是 `__main__` 模块来改变其行为。
例如，一个模块可能包含一些测试代码，如果模块在 `__main__` 模块中就运行，在其他模块中则不运行。

```Python
# Check if running as a program
if __name__ == '__main__':
    # Yes. Running as the main script
    statements
else:
    # No, be imported as a module
    statements
```

库源码文件通常使用这种技巧，来测试或运行示例代码。
假如你在开发一个模块，可以将调试代码放到上面展示的 `if` 语句中，并在主程序运行模块。

假如你在创建一个 Python 代码目录，可以使用特殊的 `__main__.py` 文件。
例如：

```text
myapp/
    foo.py
    bar.py
    __main__.py
```

然后可以通过命令 `python3 myapp` 来运行该模块。
执行会从 `__main__.py` 开始，将 `myapp/` 做成压缩文件也可以。

## Packages

Python 代码可以被组织成一个 “包”。包是一组模块的集和，这些模块被归在一个共同的高级名称下。
这种分组有助于解决不同应用程序中使用的模块名称之间的冲突，并将你的代码与他人的分开。
包通过创建一个独特的目录名称和一个空的 `__init__.py` 文件定义。
然后将额外的 Python 文件和子包放到目录中，例如：

```text
graphics/
    __init__.py
    primitive/
        __init__.py
        lines.py
        fill.py
        text.py
        ...
    graph2d/
        __init__.py
        plot2d.py
        ...
    graph3d/
        __init__.py
        plot3d.py
        ...
    formats/
        __init__.py
        gif.py
        png.py
        tiff.py
        jpeg.py
```

`import` 语句用于从包中加载模块，其使用方式与加载简单模块相同，只是现在需要使用更长的名称。

```Python
# Full path
import graphics.primititve.fill
...
graphics.primitive.fill.floodfill(img, x, y, color)

# Load s specific submodule
from graphics.primitive import fill
...
fill.floodfill(img, x, y, color)

# Load a specific function from a submodule
from graphics.primitive.fill import floodfill
...
floodfill(img, x, y, color)
```

当包导入的时候，`__init__.py` 中的代码会先被执行。
该文件可以是空的，也可以包含一些初始化代码。
如果导入深度嵌入的子模块，在遍历目录结构时遇到的所有 `__init__.py` 文件都会执行。
`import graphics.primitive.fill` 首先会执行 `graphics/` 下的 `__init__.py` 文件，然后是 `primitive/` 下的 `__init__.py` 文件。

一个 `import` 语句的重要特性是，所有模块的导入都要求绝对的或明确的包路径。
这包括包内部使用的导入语句。
例如，假设模块 `graphics.primitives.fill` 模块想要导入 `graphics.primitives.lines` 模块。
则简单的 `import lines` 无法正常工作，会得到一个 `ImportError` 错误。
相反，需要完全确定导入路径，像这样：

```Python
# graphics/primitives/fill.py

# Fully qualified submodules import
from graphics.primitives import lines
```

遗憾的是，像这样写出完整包名即麻烦又容易出错。
例如，重命名一个包会导致硬编码的导入失效。
因此，更好的选择是相对包导入：

```Python

# graphics/primitives/fill.py

# Paskage-relative import
from . import lines
```

相对导入只能使用 `from module import symbol` 这样的语法。
因此 `import ..primitives.lines` 或 `import .lines` 这样写是语法错误。
或 `from .. import primitives.lines` 这样也是不合法的。
总之，相对导入只能在一个 package 包内使用，可以是同一个包内的不同子包，但不能是不同包之间引用。

## Running a Package submodule as a script

组织成 package 的代码的 runtime 运行时环境和简单的脚本并不相同。
存在一个外层包名、子模块以及相对导入的使用。
其中一个不再支持的特性是无法直接在包的源文件上运行 Python 程序。
例如，`graphics/graph2d/plot2d.py` 文件有一些测试代码：

```Python
# graphics/graph2d/plot2d.py
from ..primitives import lines, text

class Plot2D:
    ...

if __name__ == '__main__':
    print('Testing Plot2D')
    p = Plot2D()
    ...
```

如果直接运行该脚本会得到相对导入导致的崩溃：

```bash
$ python3 graphics/graph2d/plot2d.py

Traceback (most recent call last):
  File "graphics/graph2d/plot2d.py", line 1, in <module>
    from ..primitive import line, text
ValueError: attempted relative import beyond top-level package
```

也不能进入到模块中去执行：

```bash
$ cd graphics/graph2d/
$ python3 plot2d.py

Traceback (most recent call last):
  File "plot2d.py", line 1, in <module>
    from ..primitive import line, text
ValueError: attempted relative import beyond top-level package
```

如果要将子模块作为 main script 运行，需要使用 `-m` 参数：

```bash
$ python3 -m graphics.graph2d.plot2d

Testing Plot2D
```

`-m` 参数将模块或包作为主程序，python 会使用正确的环境运行该模块，从而确保 import 能正常工作。
许多 python 的内置模块拥有的 “秘密” 特性可以通过 `-m` 运行。
最常见的是通过 `python3 -m http.server` 在当前目录运行一个 web 服务器。
你可以通过自己的包提供类似的功能。
如果 `python -m name` 对应一个包目录，则 python 会查找 `__main__.py` 文件，并将其作为脚本运行。

## Controlling the Package Namespace

包的主要目的的作为代码的顶层容器。
有时候用户会只导入顶层名称。

```Python
import graphics
```

这样导入不指定任何子模块，这会导致这样的代码失效：

```Python
import graphics

graphics.primitive.fill.floodfill(img, x, y, color)  # Fails!
```

当仅给出定义包导入时,唯一导入的文件是关联的 `__init__.py` 文件。
在这个例子中，就是 `graphics/__init__.py` 文件。

`__init__.py` 文件的主要目的就是创建与管理顶级包命名空间的内容。
通常这涉及到从更低层级的子目录中导入函数、类和其他对象。
例如，假设 graphics 包含有数百个低级函数，但大部分细节被封装在少数几个高级类中。
`__init__.py` 文件可能选择值暴露这些类：

```Python
# graphics/__init__.py
from .graph2d.plot2d import Plot2D
from .graph3d.plot3d import Plot3D
```

通过 `__init__.py` 文件，只有 `Plot2D` 和 `Plot3D` 会出现在包顶层。
用户可以这样使用该类：

```Python
from graphics import Plot2D

plt = Plot2D(100, 100)
plt.clear()
```

这样用户使用起来就十分方便，无需知道你的代码组织结构。

## Controlling Package Exports

一个组织上的问题涉及 `__init__.py` 文件与低处子模块之间的交互。
例如，一个包的不同子模块通常知道那些符号需要导出到顶层。
然而，实际的工作是在 `__init__.py` 中完成的。
这使得阅读包的代码并理解其组织结构变得困难。

为了更好地管理子模块，通常会定义一个 `__all__` 列表。
该列表是命名空间中需要提升一级的包。

```Python
# graphics/graph2d/plot2d.py

__all__ = ['Plot2D']

class Plot2D:
    ...
```

相关的 `__init__.py` 文件通过 `*` 号导入：

```Python
# graphics/graph2d/__init__.py
# Only loads names explicitly listed in __all__ variables
from .plot2d import *

# Propagate the __all__ up to next level
__all__ = plot2d.__all__
```

这种提升过程会一直持续到顶层的 `__init__.py`，例如：

```Python
# graphics/__init__.py
from .graph2d import *
from .graph3d import *

# Consolidate exports
__all__ = [
    *graph2d.__all__,
    *graph3d.__all__,
]
```

值的注意的是，尽管在用户代码中使用 `*` 导入并不被推荐，但在包的 `__init__.py` 中这种做法非常普遍。

## Package data

有时候包会包含一些数据文件（非代码）。
在包中，`__file__` 变量会给予关于源文件位置的信息。
但是包会被编译，他们可能被打包成 `.zip` 文件或其他不常见的格式。
文件中的 `__file__` 变量可能并不可靠，甚至根本没有定义。
结果导致读取数据文件并不容易，通过文件名和内置函数 `open()` 也不方便。

为了读取包数据，使用 `pkgutil.get_data(package, resource)`。
例如包结构如下：

```bash
mycode/
    resources/
        data.json
    __init__.py
    spam.py
    yow.py
```

为了导入 `data.json` 文件，可以这样：

```Python
# mycode/spam.py
import pkgutil
import json

def func():
    # rawdata: bytes 字节流 | None
    rawdata = pkgutil.get_data(__package__, 'resources/data.json')
    textdata = rawdata.decode('utf-8')
    data = json.loads(textdata)
    print(data)
```

实际上，现代 Python 版本更加推荐使用 `importlib.resources`，例如：

```Python
from importlib import resources

# 1. 获取资源句柄（并未读取文件内容）
source = resources.file('my_package').joinpath('data/config.json')

# 2. 读取文件内容
content = source.read_text(encoding='utf-8')

# 3. 第三方库如果需要无物理路径
with resources.as_file(source) as physical_path:
    # 即使在压缩包中，Python 也会创建一个临时物理文件
    print(physical_path)
```

## Module Objects

模块是一等公民，下表列出了常用的模块属性：

| Attribute           | Description              |
| ------------------- | ------------------------ |
| \_\_name\_\_        | 完整的模块名称           |
| \_\_doc\_\_         | 文档字符串               |
| \_\_dict\_\_        | 模块字典                 |
| \_\_file\_\_        | 模块定义所在的文件名     |
| \_\_package\_\_     | 所在包的名称（如果有）   |
| \_\_path\_\_        | 搜索包子模块的子目录列表 |
| \_\_annotations\_\_ | 模块级别的类型提示       |

这里的 `__dict__` 属性是一个字典，代表模块的命名空间。
所有再模块内定义的都在这里。

`__name__` 属性是通常用于脚本编写，通常会进行类似 `if __name__ == '__main__'` 的检查。
以判断文件十分作为主程序运行。

`__package__` 属性包含所在的包 (enclosing package) 名称。
如果设置了该属性，`__path__` 属性是目录列表，用于定位包的子模块。
通常，它包含一个单一条目，即包的位置。
有时，大型框架会操作 `__path__` 以包含额外的目录，目的是支持插件和其他高级功能。

并非所有的模块都具备所有属性。
例如，内置模块可能没有设置 `__file__` 属性。
同样地，对于顶级模块，与包相关的属性不会被设置。

`__doc__` 属性是模块的 docstring，这是文件中第一行出现的语句。
`__annotations__` 属性是一个模块级别的类型提升字典，类似这样：

```Python
# mymodule.py

'''
The doc string
'''

# Type hints (placed into __annotations__)
x: int
y: float
...
```

和其他类型提升相比，模块级别的类型提升不会改变 Python 的行为。
他们实际上也不定义变量，纯粹是元数据，其他工具可以根据需要选择查看。

## Deploying Python Packages

最后关于模块和包的问题是将代码发给他人。
这一个广泛的话题，多年来一直是持续活跃开发的重点。
具体流程参考：[Python 官方文档](https://packaging.python.org/en/latest/tutorials/packaging-projects/)

尝试给包起一个独特的名称，以免和其他可能的依赖项冲突。
可以参考 [Python 包索引网站](https://pypi.org)来帮助选择名称。
在组织代码结构时，尽量保持简洁。

考虑到绝对简洁性，分发纯 Python 代码最极简的方式是使用 `setuptools` 模块。
假设编写来一些代码，项目结构如下：

```text
spam-project/
    README.txt
    Documentation.txt
    spam/  # A package of code
    __init__.py
    foo.py
    bar.py
    runspam.py  # A script to run as: python runspam.py
```

为了创建分发，在顶层目录创建一个 `setup.py` 文件，并编写如下代码：

```Python
# setup.py
from setuptools import setup

setup(
    name='spam',
    version='0.0',
    packages=['spam'],
    scripts=['runspam.py'],
)
```

在这个 `setup()` 调用中，`packages` 是所有包目录的列表，`scripts` 是脚本文件列表。
如果包没有相关内容则省略，`name` 是包名称，`version` 是字符串版本号。

使用 `setup.py` 文件创建软件分发包足够了，使用下面命令来创建一个源码分发包：

```bash
python3 setup.py sdist
```

这会创建一个存档文件，例如 `spam-1.0.tar.gz` 或 `spam-1.0.zip` 到目录 `spam/dist` 下面。
这就是你给予他人按照的文件。
如果要安装该软件包，使用命令：

```bash
python3 -m pip install spam-1.0.tar.gz
```

这会将软件安装到本地 Python 环境。
一般会放在 Python 库的 `site-packages` 目录下面，可以通过 `sys.path` 查看。

Scripts 脚本通常会安装在相同的目录下。
如果脚本第一行以 `#!` 开头，且包含 `python` 文本，这安装器会重写该行指向本地 Python 地址。
因此，即使硬编码了 Python 地址，也应该仍然能工作。

这里描述的 `setuptools` 使用方式是最简化的。
大型项目可能涉及 C/C++ 扩展、复杂的包结构、示例等。
涵盖所有工具以及部署此类代码的方式可以查阅 [https://python.org](https://python.org)和 [https://pypi.org](https://pypi.org) 以获取最新建议。
