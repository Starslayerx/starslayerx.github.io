+++
date = '2025-12-15T8:00:00+08:00'
draft = false
title = 'Python Tricks Part 1: Basis'
categories = ['Note']
tags = ['Python']
+++

这篇文章总结一些平时容易被忽略的 Python 知识

### Primitives, Variables and Expressions

```Python
print(f"{year:>3d} {principal:0.2f}")
```

- `>3d` 指至少 3 位十进制数，右对齐
- `0.2f` 指精度为 2 位的浮点数

### Arithmetic Operators

`round(x, [n])`: 该函数采用 Banker's Rounding 银行家舍入法，也叫 四舍六入五成双，当要舍弃的数字正好是 5 时

- 前一位是偶数 → 向下舍去（向偶数靠拢）
- 如果前一位是奇数 → 向上进位（向偶数靠拢）

这样做的目的是减少舍入误差的累积，在统计学和金融计算中更为公平。

```Python
# 常规四舍五入（Python实际行为是银行家舍入）
print(round(1.5))   # 2  （1是奇数，5进位）
print(round(2.5))   # 2  （2是偶数，5舍去）
print(round(3.5))   # 4  （3是奇数，5进位）
print(round(4.5))   # 4  （4是偶数，5舍去）

# 更复杂的例子
print(round(1.25, 1))  # 1.2 （2是偶数，5舍去）
print(round(1.35, 1))  # 1.4 （3是奇数，5进位）
print(round(1.251, 1)) # 1.3 （因为后面还有1，不是正好5，正常进位）
```

银行家舍入法是 IEEE 754 标准推荐的方式，Python、R、NumPy 等都采用这种舍入方式，能有效减少大量数据计算时的统计偏差。

---

Python 二进制运算符会将整数视为 2's complement binary representation 二进制补码，并且符号位会在左侧无限扩展。
此外，Python 不会截断二进制，也不会溢出。

### Text Strings

Immediately adjacent string literals 紧邻的字符串 会被拼接成单个字符串：

```Python
print(
'Content-type: text/html\n'
'\n'
'<h1> Hello World </h1>\n'
'Clock <a href="http://www.python.org">here</a>\n'
)
```

---

`str()` 和 `repr()` 都输出一个字符串，但是并不相同

- `str()` 输出的是和 print() 一样的内容
- `repr()` 生成的字符串则是以可被程序直接解析的形式

```Python
s = "hello\nworld"
print(str(s))
# hello
# world

print(repr(s))
# 'hello\nworld'
```

### File Input and Output

一行一行读取文件

```Python
with open('data.text') as file:
    for line in file:
        print(line, end='') # end='' omits to printing extra newline
```

- `open()` 函数返回一个文件对象

如果不使用 with 语句，代码应该这样写

```Python
file = open('data.text')
for line in file:
    print(line, end='') # omits to printing extra newline
file.close()
```

但这样很容易忘记关闭文件，因此更加推荐上面写法

如果要读取整个文件，使用 `read()` 方法

```Python
with open('data.text') as file:
    data = file.read()
```

如果说要读取一个大文件，需要给 `read()` 方法一个参数

```Python
with open('data.text') as file:
    while (chunk := file.read(10000)):
        print(chunk, end='')
```

这里使用 `:=` 运算符来获取并返回读取的值，以便在文件结束时（会读取到空字符）退出循环。

注意使用海象运算符的时候，应该总是将表达式使用括号括起来。

每次调用 `file.read(10000)` 后，文件内部的“读取指针”会自动向前移动 10000 个字符的位置。

如果不使用海象运算符，代码会更加冗余

```Python
with open('data.text') as file:
    while True:
        chunk = file.read(10000)
        if not chunk:
            break
        print(chunk, end='')
```

如果要将程序内容输出到文件中，在 print() 中指定

```Python
with open('outupt.text', 'wt') as out:
    print('...\n', file=out)
```

此外，还可以使用 `write()` 方法写入字符串到文件中

```Python
out.write('...\n')
```

默认文件都使用 UTF-8 编码，如果使用不同的编码，使用 encoding 参数指定：

```Python
with open('data.text', encoding='latin-1') as file:
    data = file.read()
```

### Lists

初始化空列表有两种方式：

```Python
names = []
names = list()
```

一般和使用 `[]` 是更加地道的方法，`list()` 更多用于类型转换：

```Python
letters = list('Dave')
# ['D', 'a', 'v', 'e']
```

### Tuples

`tuple /'tʌpl/` 初始化 0 个和 1 个元素的方法

```Python
a = ()
b = (item, ) # 注意这里的尾逗号
```

元组元素可以和数组一样使用索引获取，但更加常见的方法是解包

```Python
host, port = addresss
```

由于元组的不可变性，最好将其视为有不同部分的不可变类型，而不是列表那种独立类型的集和。

> Lists vs Tuples

结构体定义

```c
// listobject.h
typedef struct {
  PyObject_VAR_HEAD
  PyObject **ob_item;   // 指向元素数组的指针
  Py_ssize_t allocated; // 已分配的槽位数量
} PyListObject;

// tupleobject.h
typedef struct {
  PyObject_VAR_HEAD
  PyObject *ob_item[1]; // 元素直接内联存储 (flexible array)
} PyTupleObject;
```

| 方面         | list              | tuple                      |
| ------------ | ----------------- | -------------------------- |
| 内存分配策略 | 动态增长          | 一次性分配，大小固定       |
| 内存占用     | 较大 (有预留空间) | 较小 (紧凑存储)            |
| 创建速度     | 较慢              | 较快 (小 tuple 从缓存复用) |
| 哈希支持     | 不可哈希          | 可哈希 (元素都可哈希时)    |

可以看到，tuple 对缓存会更加友好，更加符号局部性原理，命中率更高，且固定长度可优化，因此访问速度会比 list 更快。

### Dict

```Python
prices = {
    'GOOG': 490.1,
    'APPL': 123.5,
    'IBM': 91.5,
    'MSFT': 52.13
}
```

获取一个元素可以这样

```Python
if 'IBM' in prices:
    p = prices['IBM']
else:
    p = 0.0
```

使用 `get()` 方法编写更加紧凑的版本

```Python
p = prices.get('IBM', 0.0)
```

字典初始化也是两种方法，同样一般使用大括号初始化空字典

```Python
prices = {}
prices = dict()
```

一般 `dict()` 更多用于处理键值对数据

```Python
pairs = [('IBM', 125), ('ACME', 50), ('PHP', 40)]
a = dict(paris)
```

可以使用列表获取字典的键

```Python
syms = list(prices)
# ['APPL', 'MSFT', 'IBM', 'GOOG']
```

或者使用 `keys()` 方法

```Python
syms = prices.keys()
```

区别在于，`keys()` 方法会返回一个 key view，为字典键的引用（即字典修改也会改变）

```Python
d = {'x': 2, 'y': 3}
k = d.keys()
# dict_keys(['x', 'y'])
d['z'] = 4
k
# dict_keys(['x', 'y', 'z'])
```

如果要获取值，使用 `dict.values()`。
如果想获取键值对，使用 `doct.items()`。

这样遍历字典

```Python
from sym, price in prices.items():
    print(f'{sym} = {prices})
```

## Script Writing

任何 py 文件都可以作为脚本运行，或者作为通过 `import` 导入的库。
为了更好地支持导入，脚本代码通常放在一个模块名 `__name__` 进行条件检查的语句块里。
`__name__` 是一个内置变量，其始终包括当前模块的名称。

如果要传递命令行参数，可以这样，例如给一个脚本传递一个文件名称

```Python
def main(argv):
    if len(argv) == 1:
        filename = input('Enter filename: ')
    elif len(argv) == 2:
        filename = argv[1]
    else:
        raise SystemExit(f'usage: {argv[0]} [filename]')
    portfolio = read_portfolio(filename)
    for name, shares, price in portfolio:
        print(f'{name:>10s'} {shares:10d} {price:10.2f})

if __name__ == '__main__':
    import sys
    main(sys.argv)
```

- `argv[0]` 是用户输入的脚本名或可执行文件名

  ```bash
  python my_script.py          # argv[0] = "my_script.py"
  ./my_program                 # argv[0] = "./my_program"
  python /path/to/script.py    # argv[0] = "/path/to/script.py"
  ```

- `10s`: 字符串格式，右对齐，宽度 10 个字符
- `10d`: 整数格式，右对齐，宽度 10 个字符
- `10.2f`: 浮点数格式，总宽度 10 个字符，保留 2 位小数

对于简单的程序而言，`sys.argv` 已经足够了，如果有更加复杂的需求，使用 `argparse` 库。

## Packages

在更大的项目中，一般会将代码组织成一个包。
包是一个模块的层次化集合，类似这样。

```text
tutorial/
    __init__.py
    readport.py
    pcost.py
    stack.py
    ...
```

目录应该有一个 `__init__.py` 文件，这样就可以导入该包里面的

## Structuring an Application

将大型代码库组织成包结构是一种标准做法，在进行操作时，需要为顶级目录选择一个唯一的包名。
包目录的主要目的是管理导入语句和编程时使用的模块命名空间。

除了源代码外，可能还有测试、示例、脚本和文档等附加内容。
这些附加材料存放在与源码不同的目录中。
常见的做法是为项目创建一个顶层的总目录，并将所有工作内容至于其下。

```text
tutorial-project/
    tutorial/
        __init__.py
        readport.py
        pcost.py
        stack.py
        ...
    tests/
        test_stack.py
        test_pcost.py
        ...
    examples/
        sample.py
        ...
    doc/
        tutorial.txt
        ...
```

## Managing Third Party Packages

使用下面命令去 PyPI 安装第三方软件包

```bash
python3 -m pip install
```

安装的包会在 `site-packages` 目录下面，可以使用 `sys.path` 看到。
在 Unix 机器上，目录可能在 `/usr/local/bin/python3.10/site-packages`。
如果想知道某个模块具体的位置，可以在导入后使用 `__file__` 变量查看：

```Python
import asyncio
asyncio.__file__
# '/Library/Frameworks/Python.framework/Versions/3.12/lib/python3.12/asyncio/__init__.py'
```

实际中，更加常见的是使用虚拟环境

```Python
python3 -m venv myproject
```

这样将创建一个项目目录叫 myproject，在该目录中将会创建下面的目录结构

```text
myproject
├── bin/                # Linux/Mac: 可执行文件目录
│   ├── python          # Python解释器
│   ├── pip             # 包管理工具
│   └── activate        # 激活脚本
├── lib/                # Python库文件
│   └── python3.x/      # 具体Python版本
│       └── site-packages/  # 安装的第三方包
└── pyvenv.cfg         # 环境配置文件
```

这里不建议直接使用 pip 安装包，而是这样

```Python
./myproject/bin/python3 -m pip install somepackage
```
