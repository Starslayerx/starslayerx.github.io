+++
date = '2026-03-02T8:00:00+01:00'
draft = true
title = 'Python Tricks Part 9: Input and Output'
categories = ['Note']
tags = ['Python']
+++

## Data Representation

I/O 的主要问题在于外部世界。
为了与其通信，数据必须合适地表示以便处理。
在最底层，Python 处理数据两种基本数据类型：字节 bytes，代表任何类型的原始未解释数据；以及文本 text，代表 Unicode 字符。

为了表示字节，使用两种内置类型：`bytes` 和 `bytearray`。
`bytes` 是一个不可变的整数字节值字符串，`bytearray` 是一个可变的字节数组，其行为类似于字节字符串和列表的结合。
它的可变性使其适合以更渐进的方式构建字节组，这在从片段组装数据时经常发生。

```Python
# Specify a bytes literal (note: b'prefix)
a = b'hello'

# Specify bytes from a list of integers
b = bytes([0x68, 0x65, 0x6c, 0x6c, 0x6f])

# Create and populate a bytearray from parts
c = bytearray()
c.extend(b'world')  # world
c.append(0x21)      # world!

# Access bytes values
print(a[0])  # 104

for x in b:  # 104 101 108 108 111
    print(x)
```

访问字节和字节数组对象的单个元素会产生整数字节值，而不是但字符的字节字符串。
这与文本字符串不同，是一个常见的用法错误。

文本是 `str` 数据类型，并存储为 Unicode 码。

```Python
d = 'hello'
len(d)
print(d[0])  # 'h'
```

Python 对 bytes 和 text 有着严格的区分。
两者之间没有自动转换，涉及混合类型的比较结果均为 `False`，任何混合两者的结果都是错误。

例如：

```Python
a = b'hello'  # bytes
b = 'hello'   # text
c = 'world'   # text

print(a == b) # False
d = a + c     # TypeError: can't contact str to bytes
e = b + c     # 'helloworld'
```

当执行 I/O 是，要确保在处理正确的数据表示。
如果是文本数据，使用 text strings。
如果是二进制数据，使用 bytes。

## Text Encoding and Decoding

如果要处理 text，所有输入数据都需要 decode 解码，且所有输出数据都要 encode 编码。
来进行 text 和 bytes 之间的转换，有 `encode(text [,errors])` 和 `decode(bytes [,errors])` 方法，例如：

```Python
a = 'hello'  # Text
b = a.encode('utf-8')  # Encode to bytes

c = b'world' # Bytes
d = c.decode('utf-8')  # Decode to text
```

无论是 `encode` 还是 `decode` 都需要指定 `utf-8` 或 `latin-1` 这样的编码参数。
此外，编码方法接受一个可选的 errors 参数，用于指定出现编码错误时的处理方式。

| Value                 | Description                                                                         |
| --------------------- | ----------------------------------------------------------------------------------- |
| `'strict'`            | 抛出 `UnicodeError` 的编码异常                                                      |
| `'ignore'`            | 忽略无效字符串                                                                      |
| `'replace'`           | 替换无效字符串 (`U+FFFF` in Unicode, `b'?'` in bytes).                              |
| `'backslashreplace'`  | 将无效字符替换为 Python 字符转义序列。例如 `U+1234` 被替换为 `\u1234` (仅 encoding) |
| `'xmlcharrefreplace'` | 将无效字符替换为 XML 字符引用。例如 `U+1234` 被 `&#4660;` 替换 (仅 encoding)        |
| `'surrogateescape'`   | 将无效字节 `\xhh` 替换为 `U+DChh`                                                   |

其中，`backslashreplace` 和 `xmlcharrefreplace` 错误处理策略可以将无法表示的字符转换为特定格式，使其更容易以简单的 ASCII 或 XML 字符引用形式查看。在调试过程非常有用。

`surrogateescape` 错误处理策略允许退化的字节数据，在解码/编码的往返过程中保持完整，无论使用何种文本编码。
具体来说如 `s.decode(enc, 'surrogateescape).encode(enc, 'surrogateescape) == s`。
这种数据的往返保存对于某些系统接口非常有用，这些接口期望文本编码，但由于 Python 无法控制的外部问题，无法绝对保证编码的一致性。
Python 没有因为错误的编码 (Bad Encoding) 而丢弃或顺坏数据，而是使用 "代理编码" (Surrogate Encoding) 将其原样嵌入。

下面的例子展示了不当 UTF-8 字符编码：

```Python
a = b'Spicy Jalape\xflo'  # Invalid UTF-8
a.decode('utf-8')  # UnicodeDecodeError

a.decode('utf-8', 'surrogateescape')  # 'Scipy Jalape\udcflo'
_.encode('utf-8', 'surrogateescape')  # b'Scipy Jalape\xflo'
```

## Text and Byte Formatting

处理文本和字节字符串时，一个常见的问题的字符串转换和格式化。
例如，使用 `format()` 函数格式化浮点数一定的长度和精度。

```Python
x = 123.456
format(x, '0.2f')    # '123.46'
format(x, '10.4f')   # '  123.4560'
format(x, '<*10.2f') # '123.46****'
```

`format()` 的第二个参数通常是 `[fill[align]][sign][0][width][,][.precision][type]` 其中每部分 `[]` 都是可选的。
`with` 指定符指定了最小长度，对齐指定符是 `<, >` 或 `^`，分别对应左对齐，右对齐或居中。
可选的 `fill` 用于替换空格：

```Python
name = 'Elwood'
r = format(name, '<10')  # r = 'Elwood    '
r = format(name, '>10')  # r = '    Elwood'
r = format(name, '^10')  # r = '  Elwood  '
r = format(name, '*^10') # r = '**Elwood**'
```

`type` 指定符指定数据类型，如果未指定默认是 `s` 代表字符串，`d` 代表整数，`f` 代表浮点数。
格式说明符中的符号部分可以是 `+`, `-` 或空格。
`+` 表示所有数字前都应该使用符号，`-` 是默认选项，表示仅对负数添加符号字符。
空格会在正数前添加一个前导空格。

在 width 和 precision 之前可能有个逗号 comma (,)。
为每千位分隔符，例如；

```Python
x = 123456.78
format(x, '16,.2f')  # '      123,456.78'
```

说明符的 precision 精度部分指定了用于小数时保留的位数。
如果在数字的字段宽度前加前导 0，数值会用前导 0 填充以占满空间。
下面是一些例子：

```Python
x = 42
r = format(x, '10d')  # '        42'
r = format(x, '10x')  # '        2a'
r = format(x, '10b')  # '    101010'
r = format(x, '010b') # '0000101010'

y = 3.1415926
r = format(y, '10.2f')   # '      3.14'
r = format(y, '10.2e')   # '  3.14e+00'
r = format(y, '+10.2f')  # '     +3.14'
r = format(y, '+010.2f') # '+000003.14'
r = format(y, '+10.2%')  # '  +314.16%'
```

对于更加复杂的字符串格式化，使用 f-strings：

```Python
x = 123.456

f'Value is {x:0.2f}'       # 'Value is 123.456'
f'Value is {x:10.4f}'      # 'Value is   123.4560'
f'Value is {2*x:*<10.2f}'  # 'Value si 246.91****'
```

当 f 字符串这样表示 `{expr:spec}` 会被替换为 `format(expr, spec)`。
expr 可以是任意表达式，只要不包含 `{, }` 或 `\` 字符串。
格式说明符的某些部分可以选择性地由其他表达式提供。

```Python
y = 3.1415926
width = 8
precision = 3
r = f'{y:{width}.{precision}}'  # '   3.142'
```

如果以等号结束表达式，那么表达式的字面文本也会包含在结果中。例如：

```Python
x = 123.456

f'{x=:0.2f}'   # 'x=123.46'
f'{2*x=:0.2f}' # '2*x=246.91'
```

如果在值后面添加 `!r`，则会通过 `repr()` 格式化。
如果使用 `!s`，则输出会通过 `str()` 格式化。
例如：

```Python
f'{x!r:spec}'  # Calls (repr(x).__format__('spec'))
f'{x!s:spec}'  # Calls (str(x).__format__('spec'))
```

也可以使用 `.format()` 替代 f-string。

```Python
x = 123.456

'Value is {:0.2f}'.format(x)           # 'Value is 123.46'
'Value is {0:10.2f}'.format(x)         # 'Value is   123.4560'
'Value is {val:<*10.2f}'.format(val=x) # 'Value is 123.46****'
```

f 字符串的格式 `{arg:spec}` 会被替换为 `format(arg, spec)`。
如果 arg 省略，则会按照顺序调用：

```Python
name = 'IBM'
shares = 50
price = 490.1
r = '{:>10s} {:10d} {:10.2f}'.format(name, shares, price)  # '       IBM        50    490.10'
```

`arg` 也可以作为参数或名称，例如：

```Python
tag = 'p'
text = 'hello world'

r = '<{0}>{1}</{0}>'.format(tag, text)
r = '<{tag}>{text}</{tag}>'.format(tag='p', text='hello world')
```

于 f 字符串不同，这里的 `arg` 值不能是一个表达式。
因此，`format()` 不是特表有表达性。
然而，该方法能执行有限的属性查找、索引和嵌套替换操作。

```Python
y = 3.1415926
width = 8
precision = 3

r = 'Value is {0:{1}.{2}f}'.format(y, width, precision)
d = {
    'name': 'IBM',
    'shares': 50,
    'price': 490.1
}
r = '{0[shares]:d} shares of {0[name]} at {0[price]:0.2f}'.format(d)  # '50 shares of IBM at 490.10'
```

`bytes` 和 `bytearray` 可以使用 `%` 符号格式化。
该符号的语义模仿 C 编程中的 `sprintf()` 函数。
下面有一些例子：

```Python
name = b'ACME'
x = 123.456

b'Value is %0.2f' % x     # b'The value is 123.46'
b'%s = %0.2f' % (name, x) # b'ACME = 123.46'
```

在这种格式化方式中，形如 `%spec` 的序列会按顺序被替换为 `%` 运算符第二个操作数中的值。
基本格式码与 `format()` 函数使用的相同。
然而，更高级的功能要么缺失，要么略有变化。
例如，要调整对齐格式可以这样：

```Python
x = 123.456
b'%10.2f' % x  # b'    123.46'
b'$-10.2f' % x # b'123.45    '
```

使用 `%r` 格式代码会产生 `ascii()` 的输出，这在调试和日志记录中非常有用。
当使用 bytes 的时候，要谨慎 text 字符串是不支持的。
相反，他们需要被编码 encode:

```Python
name = 'Dave'

b'Hello %s' % name  # TypeError
b'Hello %s' % name.encode('utf-8')  # OK
```

这种方法也可以用于字符串格式化，但一般被任务的旧代码风格。
但仍然能在一些库中见到，例如 `logging` 库：

```Python
import logging
log = logger.getLogger(__name__)

log.debug('%s got %d', name, value)  # '%s got %d' % (name, value)
```

## Reading Command-Line Options

当 Python 启动的时候，命令行选项会放在 `sys.argv` 里作为文本字符。
第一个参数的程序名称，随后是程序名称后面的各种选项。
下面程序展示了一个最小手动处理命令行参数的原型：

```Python
import sys

def main(argv):
    if len(argv) != 3:
        raise SystemExit(f'Usage: python {argv[0]} inputfile outputfile\n')
    inputfile = argv[1]
    outputfile = argv[2]

if __name__ == '__main__':
    main(sys.argv)
```

为了更好的代码组织、测试，通常会编写 `main()` 函数直接读取 `sys.argv`。
`sys.argv[0]` 包含要执行的脚本名称。
当报错的时候输出一份报错语句，并抛出 `SystemExit` 是实践标准。

简单脚本可以手动处理命令选项，对于复杂命令行，建议使用 `argparse` 模块。

```Python
import argparse

def main(argv):
    p = argparse.ArgumentParser(description='This is some program')

    # A positional argument
    p.add_argument('infile')

    # An option taking an argument
    p.add_argument('-o', '--output', action='store')

    # An option that sets a boolean flag
    p.add_argument('-d', '--debug', action='store_true', default=False)

    # Parse the command line
    args = p.parse_args(args=argv)

    # Retrive the option settings
    infile = args.infile
    output = args.output
    debugmode = args.debug

    print(infile, output, debugmode)

if __name__ == '__main__':
    import sys
    main(sys.argv[1:])
```

此外，还有一些第三方组件例如 `click` 和 `docopt` 可以简化编写更加复杂的命令行解析器。
最后，有可能通过无效的文本编码向 Python 提供命令行选项。
这类参数仍然会被接受，但会使用 `surrogateescape` 错误处理方式进行编码。
如果这些参数后来被包含在任何类型的文本输出中，则需要意识到这一点，这对避免程序崩溃至关重要。

## Environment Variables

有时候数据是通过 shell 中设置的环境变量传递给程序。
例如，一个 Python 程序可能这样启动：

```bash
$ env SOMEVAR=somevalue python3 somescript.py
```

环境变量通过 `os.environ` 字符串映射访问：

```Python
import os

path = os.environ['PATH']
user = os.environ['USER']
editor = os.environ['EDITOR']
val = os.environ['SOMEVAL']
```

可以这样修改环境变量

```Python
os.environ['NAME'] = 'VALUE'
```

修改的环境变量会影响该进程和其所有子进程。
与命令行选项类似，编码错误的环境变了可能会生成使用 `surrogateescape` 错误处理策略的字符串。

## Files and File Objects
