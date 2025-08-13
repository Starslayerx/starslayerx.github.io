+++
date = '2025-08-13T8:00:00+08:00'
draft = false
title = 'Python Strings'
+++

这篇文章总结一下 Python 中字符串的类型

### Unicode String 字符串
u 在 Python3 中是多余的, 因为所有的普通字符串默认都是 Unicode, 但在 Python2 中, u 用来显示的表示 Unicode 字符串, 现在保留这个是为了向后兼容


### Fromatted String 格式化字符串
f 前缀用于创建格式化字符串, 这是最常见的字符串格式方法, 运行在字符串中嵌入表达式, 在求值时转换为普通的 `str`
```Python
name = "World"
greeting = f"Hello, {name}!"  # 结果: "Hello, World!"
```

### Raw String 原始字符串
r 前缀用于创建原始字符串, 会忽略反斜杠 `\` 的转义功能, 在编写文件路径或正则表达式的时候非常有用, 可以避免大量的反斜杠转义
```Python
path = r"C:\Users\Documents"    # 单个反斜杠 '
regex = r"\bword\b"             # \b 不会被转义
```

### Bytes String 字节串
b 前缀用于创建字节串字面量, 表示一个不可变的字节序列, 而不是 Unicode 文本, 字节串主要用于二进制数据, 例如图像文件、网络数据或压缩文件等
```Python
binary_data = b"Hello"          # 存储的是 ASCII 编码的字节
```

### Template String 模板字符串
t 前缀用于创建模板字符串, 这是 Python 3.14 引入的新功能, 由 [PEP 750](https://peps.python.org/pep-0750/) 通过.

不同于 f-string, t-string 不会立即求值为 `str`, 而是求值为一个 Template 对象, 这为开发者提供了将在将字符串和插值组合之前进行处理(和安全转义)的能力

```Python
from string.templatelib import Template
template = t"Hello, {name}"  # template 是一个 Template 对象
```


### 组合使用
| 前缀 | 含义   | 用途 |
| :--- | :----- | :--- |
| f    | 格式化 | 嵌入变量和表达式 |
| r    | 原始 | 忽略反斜杠转义 |
| b    | 字节 | 	处理二进制数据 |
| t    | 模板 | 在组合前处理插值 |
| u    | Unicode | Python 3 中默认开启 |
| fr / rf | 格式化+原始 | 在正则表达式中嵌入变量 |
| br / rb | 字节+原始 | 忽略二进制数据中的转义 |
| tr / rb | 模板+原始 | 模板中处理原始文本 |
