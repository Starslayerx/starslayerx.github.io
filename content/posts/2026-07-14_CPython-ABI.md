+++
date = '2026-07-14T8:00:00+01:00'
draft = true
title = 'What Every Python Developer Should Know About the CPython ABI'
categories = ['Article']
tags = ['Python']
+++
[原文](https://labs.quansight.org/blog/python-abi-abi3t)

CPython Application Binary Interface (ABI) 支持了 Python 的主要超能力:
它能够轻松调用 C, C++, Rust 或 Fortran 代码, 而这些代码也能反过来调用解释器, 并更新 Python 对象的状态.

ABI 是什么? 它和 API 有什么区别?
为什么说这是每个 Python 开发者都需要知道的呢?
这些内容不应该是在使用 Python 这种高级语言的时候可忽略的吗?

这篇文章, 我希望回答所有这些问题, 并帮你建立对这些话题的直觉理解.
我也希望你能学到一些有用的信息, 比如包含原生扩展的 Python 项目是如何分发的.
wheel 文件名里出现的 ABI 兼容性标签是什么意思, 以及项目如何根据自己想要做的权衡来选择针对不同的 Python ABI.

你还会了解到 Python 的 C API 和 ABI 近年来的如何演化的,以及新的自由线程解释器在这一过程中扮演了什么角色.
我还会分享 Python 3.15 将如何退出一套新的 ABI, 以修复自由线程构建所引入的问题.

确实, 你可以快来地写上好几年的 Python 代码, 根本不需要考虑这篇文章里的任何内容.
所以你可能会觉得标题说这东西是每个Python开发者都应该知道的, 有点夸张.
然而, 当你交付一个包, 调试一个 wheel 为什么无法安装, 或者导入或调用某个函数出现了段错误 *segfaults* - 这些细节就开始重要了.
哪怕是编写和维护单文件脚本, 也会让你想象中更接近代码的分发.

## The CPython C API and the Python ABI

当实现下面代码的时候 Python 解释器会做一点魔法.
`np.array` 函数可以作为 Python 代码调用, 但实际上这个函数是用 C 写的.
CPython 解释器会透明处理这个.

```Python
import numpy
np.array([1, 2])
```

这个魔法是如何实现的呢? 通过 CPython C API 和对应的 Python ABI.
当前, 这并不能解释这些术语什么意思.

### The CPython C API

正如 [CPython](https://github.com/python/cpython) 的名称一样, 他是由 C 语言实现的.
为了访问解释器内部, 只需要在 C 程序中 `#inlucde Python.h`.
这样就能轻松获取解释器内部状态, 并通过 CPython 的 C API 来调用解释器, 而解释器本身用的也是这套 C API.

C API 到底是什么? 他是一堆宏 *macros*, 类型定义 *typedefs*, 函数和结构 *structs*, 并定义了 C 头文件 `Python.h`.
可以写 C 代码来模仿任何 Python 脚本的行为, 代价是需要编译, 代码冗长, 还得小心 C 语言的各种坑.
好处是速度很快.

把 Python 代码转换成能调用 C API 的编译型语言, 通常能让速度提升一个数量级.
[Cython](https://cython.readthedocs.io/en/latest/) 编程语言利用了这一点,
作法之一是把 Python 代码先编译成 C 源代码, 然后再把这些 C 代码编译并链接到更大的程序里, 最后运行起来就跟原来生成的 Python 代码一模一样.
Cython 特别受益于知道类型信息, 这样就能生成高效的 C 代码.
这种翻译并不完美, 但足够支持这些流行库, 例如 Pandas, scikit-image 或 scikit-learn.

### The Distinction Between a C API and an ABI

什么是 C API 所不是的?
C API 纯粹是 C 编程语言的一种构建产物.
用非 C 语言编写的代码也可以调用 C API, 但这只能通过使用所有在 CPU 上直接执行的代码所使用的、特定于平台和架构的调用约定来实现, 从而将无法用 C 语言表达的功能抽象掉.

一个 C API 包含了 C 头文件的所有细节, 包括函数签名 *signatures (prototype 去掉参数)*, 宏 *macros*, 类型定义 *typedefs* 和 内联函数 *inline functions*. 也包括头本身.

C ABI 覆盖了编译好的程序要正确调用一个库所需要的所有东西:
库导出的符号, 还有双方交换数据时内存的布局方式.
符号就是一个名称, 像 `PyDict_GetItemRef`, 链接器会把这个名字解析成函数实现所在的地址.

重要的是, 函数签名的任何部分都不会被编码为二进制的:
调用方在编译时, 必须使用一个声明了正确参数类型和返回类型的文件.
结构体的布局, 也就是每个字段的顺序, 大小和偏移量, 同样在编译的时候就固化到两个二进制文件里了.
这就是为什么 ABI 不匹配会这么讨厌: 链接的时候啥毛病没有, 运行起来却会读错数据.

C ABI 并不包含 C API 里面的那些东西.
这包含所有 C 语言或 C 预处理器有的那些约定俗成的东西, 比如宏 *macros*, 类型定义 *typedef* 和内联函数 *inline functions*.
Python ABI 其实也不看函数参数的名字, 它只关心怎么在内存里存放和排列那些在 C API 里暴露出来的不同类型的实例.

当使用 CPython 的 C API 时, 要是琢磨 ABI 和 API 的区别, 可能会有点绕.
因为 C API 里很多名字虽然是用宏实现的, 但逻辑上他们就和函数一样.
因为他们是作为宏 *macros* 实现的, 故他们不会在 Python ABI 中出现.
这意味着, 像 Rust 这种不能直接编译成 C 语法的编程语言, 就无法使用那些被定义为 C 宏或者内联函数的 C API 项目.
相反, Rust 扩展依赖于用 Rust 重新实现 C 接口所提供的宏和静态内联函数, 而这些宏和函数都是基于 Python ABI 中的内容.
C++ 与 C 预处理器是兼容的, 所以可以在 C++ 扩展里直接使用 CPython C API
暴露出来的类型定义 *typedefs*, 宏 *macros* 和内敛函数 *inline functions*.

特别是 C 语言 ABI 的这些细节, 在实践中对 C++ 和 Rust 都非常重要.
Rust 没有任何 ABI 稳定性保证:
除非使用 `#[repr(C)]` 显示地把结构体设置为遵循 C 语言 ABI 约定,
否则编译期不会保证结构体成员的内存布局和顺序.
C++ 虽然允许定义具有稳定 ABI 的 C++ API, 
但由于各种 C++ 标准库实现中关于 ABI 稳定性的细节,
以及名称修饰这类复制问题, 实际操作起来要麻烦得多.
实际上, C 语言 ABI 是现代计算环境中的底层通用语言,
大多数流行编程语言都靠它来实现二进制层面的互操作性.

## The Python ABI

当谈论一个针对特定 CPU 和操作系统的具体 Python 扩展模块时, 把 ABI 定义从概念上分成两层会很有帮助: 平台 ABI (平台相关细节) 和 Python ABI (Python 相关细节), 接下来会一次讨论这两层.

### What Is a Platform ABI?

平台 ABI 规定了机器在不同架构上执行时的具体西吉:
比如编译期如何把调用像 `PyDict_GetItemRef` 这样的 C 函数翻译成一族具体的机器码,
这些机器码会根据代码最终运行的平台规范, 设置 CPU 的寄存器以及其他和平台相关的细节.

一个平台的 ABI 决定了诸如机器码如何向函数传递参数的具体方式.
例如在 x86-64 架构上, Linux 和 macOS 采用的 System V ABI 中,
函数的前六个非浮点参数通过寄存器传递, 其余参数则通过栈转递.
而同一 CPU 上的 Windows x64 调用约定, 只有前四个参数通过寄存器传递.

通常, 开发者不需要操心这些细节, 编译器会自动生成适合开发者想针对任何平台 ABI 的机器码.
不过, 有一点很重要: 底层平台 (也就是操作系统和 CPU 机构的组合) 有自己独特的 ABI,
为一个平台 ABI 编译的代码, 在另一个平台 ABI 完全没法用.
这一事实很大程度上决定了二进制 wheel 分发格式的设计, 后面会详细讨论.

最大的后果是, 每个平台和每种 CPU 架构都有自己独特的平台 ABI, 所以需要格子独立的构建版本.
这也是像 NumPy 这样的项目每次发布都要发布那么多二进制 wheel 的原因,
项目需要为没一个他们想支持的平台 ABI 都编译出对应的二进制文件.

我在这里稍微简化一些细节哈:
比如系统 libc 库的实现 (Linux 上 musl 和 glibc 的区别——这背后就是 manylinux 和 musllinux 平台标签的不同), 
或者编译器的实现 (Windows 上 MSVC 和 mingw-w64 的区别), 也会影响 ABI (应用程序二进制接口).
在很多情况下, 这种复杂性会通过所谓的 “目标三元组” (比如 aarch64-apple-darwin) 来参数化.
每个不同的目标三元组都对应一组不同的 ABI 约定, 编译器工具链必须支持这些约定才能生成有效的二进制文件.


### ABI Tags and Wheel Filenames: Understanding Wheel Compatibility
大多数流行的 Python 包都会以二进制轮转 wheel 的形式,
把编译好的二进制文件上传到 PyPI 上.
wheel 就是一个用 zip 压缩的文件夹, 里面包含了 Python 代码, 有时候还带有编译好的产物.

这个帖子写的更加详细 [Python Wheels: from Tags to Variants](https://labs.quansight.org/blog/python-wheels-from-tags-to-variants).
这里特别关注的是 wheel 文件的文件名, 当在一台 ARM 架构的 Mac 笔记本的环境运行 `pip install cryptography` 会发送什么:
```
$  pip install cryptography
Collecting cryptography
  Downloading cryptography-48.0.0-cp311-abi3-macosx_10_9_universal2.whl.metadata (4.3 kB)
Collecting cffi>=2.0.0 (from cryptography)
  Downloading cffi-2.0.0-cp314-cp314-macosx_11_0_arm64.whl.metadata (2.6 kB)
Collecting pycparser (from cffi>=2.0.0->cryptography)
  Downloading pycparser-3.0-py3-none-any.whl.metadata (8.2 kB)
Downloading cryptography-48.0.0-cp311-abi3-macosx_10_9_universal2.whl (8.0 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 8.0/8.0 MB 11.6 MB/s  0:00:00
Downloading cffi-2.0.0-cp314-cp314-macosx_11_0_arm64.whl (181 kB)
Downloading pycparser-3.0-py3-none-any.whl (48 kB)
Installing collected packages: pycparser, cffi, cryptography
Successfully installed cffi-2.0.0 cryptography-48.0.0 pycparser-3.0
```

特别留意一下 wheel 文件名, 因为 wheel 文件里面大多数信息都和 ABI 兼容性有关.

```
{distribution} - {version} - {python} - {abi} - {platform}

cryptography   - 48.0.0    - cp311    - abi3  - macosx_10_9_universal2
cffi           - 2.0.0     - cp314    - cp314 - macosx_11_0arm64
pycparser      - 3.0       - py3      - none  - any
```

第三, 第四和第五部分, 也就是兼容性标签, 才是对 ABI 兼容性至关重要的.
其中, python 这个标签表示这个 wheel 文件支持的具体版本, 或者是最低支持的版本.
至于它表示的是最低版本还是具体版本, 得看下一个 ABI 标签.
比如 pycparser 用的那个 py3 标签, 表示这个 wheel 可以在任何运行任何Python 版本 Python3 解释器上使用, 不一定是 CPython.
而完整的 py3-none-any 三元组则意味着这个 wheel 兼容任何 Python3 版本和任何平台:
它只包含纯 Python 代码, 没有编译过的扩展.
在看 cp311 和 cp314 标签里的 cp, 表示 cryptography和cffi这两个wheel只能在CPython上使用.
其他 Python 实现就得用不同的 wheel 了.

`cp311-abi3` 和 `cp314-cp314` 这两个标签表示这些 whl 文件里包含了编译好的扩展.
这些标签指明了这个 whl 是针对 CPython 的哪个 ABI 版本的;
后面会解释这是什么意思, 但要讲清楚, 得先说说 CPython 的 C API 是怎么组织的.

## The Layers of the CPython C API


