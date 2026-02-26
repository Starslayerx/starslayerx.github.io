+++
date = '2025-10-30T8:00:00+08:00'
draft = true
title = 'Python Standrad Library - Python Runtime Services - sys'
categories = ['Blog']
tags = ['Python']
+++

# sys - System-specific pararmeters and functions

该模块提供解释器使用或报错的变量的访问，以及一些和解释器强互动的函数。

- `sys.abiflags`  
  Python 解释器的 ABI (Application Binary Interface) 标志字符串，用于表示 Python 解释器的编译配置。

  API 是应用程序二进制接口，定义了程序在二进制级别如何交互。

- `sys.addaudithook(hook)`  
  将可调用的对象 hook 附加到当前（子）解释器的审计钩子 (Audit Hook) 列表中，用于监控和记录 Python 运行时的敏感操作。

  这里的 hook 是一个回调函数 (Callback Function): 把一个函数作为参数传递给另一个函数，并在合适的时机”回头调用“它。

- `sys.argv`  
  一个列表，其中包含了被传递给 Python 脚本的命令行参数。
  `argv[0]` 为脚本名称，如果通过命令参数 `-c` 执行，则 `argv[0]` 会被设置为字符串 `-c`。
  如果没有脚本名，则为空字符串。

- `sys.audit(event, *args)`  
  引发一个审计事件并触发任何激活的审计钩子。

- `sys.base_exex_prefix`

- `sys.base_prefix`

| 属性                 | 系统 Python | 虚拟环境      | 说明                         |
| -------------------- | ----------- | ------------- | ---------------------------- |
| sys.prefix           | /usr/local  | /path/to/venv | 当前环境的纯 Python 文件路径 |
| sys.base_prefix      | /usr/local  | /usr/local    | 原始安装的纯 Python 文件路径 |
| sys.exec_prefix      | /usr/local  | /path/to/venv | 当前环境的二进制文件路径     |
| sys.base_exec_prefix | /usr/local  | /usr/local    | 原始安装的二进制文件路径     |

- `sys.byteorder`  
  本地字节顺序的指示符，在大端序（最高有效位有限）操作系统上值为 `'big'`，小端序操作系统为 `'little'`

- `sys.call_tracing(func, args)`  
  在追踪函数内部，重新启用追踪，临时恢复追踪，用于调速器、性能分析工具、代码覆盖率工具等。
  当启动跟踪时，调用 `func(*args)`，跟踪状态码将被保留，并在以后恢复。

- `sys.copyright`  
  字符串，包含了 Python 解释器相关的版权信息。

- `sys.breakpointhook()`  
  本钩子函数由内建函数 breakpoint() 调用，默认情况下，会进入 pdb 调试器，可以将其修改为任何其他函数，已选择使用哪个调试器。

- `sys.dllhandle`  
  指向 Python DLL 句柄的整数 (windows)。

- `sys.displayhook(value)`  
  如果 value 不是 `None`，则将 `repr(value)` 打印至 `sys.stdout`，并将 value 保存在 `builtins._` 中。
  如果 `repr(value)` 无法用 `sys.stdout.errors` 错误处理方案编码为 `sys.stdout.encoding`，则用 `'backslashreplace'` 错误处理方案将其编码为 `sys.stdout.encoding`。

- `sys.dont_write_bytecode`  
  若为 true, 则 Python 导入源码模块时将不会尝试写入 `.pyc` 文件，该值会被初始化为 `True` 或 `False`，依据 `-B` 命令选项和 `PYTHONDONTWRITEBYTECODE` 环境变量，可以自行设置是否生成字节码文件。

- `sys.pycache_prefix`  
  如果设置了该值 (不能为 None)，Python 会将字节码缓存文件 `.pyc` 写入到以该值指定的目录为根的并行目录树中（并从中读取），而不是在源代码树的 `__pycache__` 目录下读写。

- `sys.excepthook(type, value, traceback)`  
  本函数会将所有的回溯和异常输出到 `sys.stderr` 中。

- `sys.execption()`  
  当此函数在某个异常处理器执行过程中被调用时，将返回被该处理器所捕获的异常实例。
  当有多个异常处理器嵌套时，只有最内层处理器所处理的异常可以被访问到。
  如果没有异常处理器在执行，此函数将返回 `None`。

- `sys.exc_info()`  
  此函数返回被处理异常的旧表示形式，如果异常 `e` 已被处理，则将返回元组。

- `sys.exec_prefix`  
  一个字符串，提供特定领域的目录前缀

- `sys.executable`  
  字符串，提供 Python 解释器可执行二进制文件的绝对路径。

- `sys.exit([arg])`  
  引发一个 SystemExit 异常，表达打算退出解释器。

- `sys.flags`  
  具名元组 flags 含有命令行标志的状态，这些属性只读。

- `sys.float_info`  
  具名元组，存储浮点型相关数据。

- `sys.float_repr_style`  
  字符串，反因 `repr()` 函数在浮点数上的行为。

- `sys.getalllocatedblocks()`  
  返回当前解释器的内存块数量，无论其大小。

- `sys.getunicodeinternalsize()`  
  返回已被处置的 unicode 对象数量

- `sys.getdefaultencoding()`  
  返回 `'utf-8'`，默认字符编码的名称

- `sys.getfilesystemcoding()`  
  获取文件系统编码格式：该编码格式与 文件系统错误处理器 一起使用以便在 Unicode 文件名和字节文件名之间进行转换。

- `sys.getfilesystemcodeerrors()`  
  获取文件系统错误处理器：该错误处理器与 文件系统编码格式 一起使用以便在 Unicode 文件名和字节文件名之间进程转换。

- `sys.get_int_max_str_digits()`  
  返回 整数字符串转换长度限制 的当前值。

- `sys.getrefcount(object)`  
  获取 object 的引用计数，返回数量通常比预期多，因为它包括了作为 `getrefcount()` 参数在这一次引用。

- `sys.getrecursionlimit()`  
  返回当前的递归限制，即 Python 解释器堆栈的最大深度。
  此限制可防止无限递归导致的 C 堆栈溢出和 Python 崩溃。

- `sys.getsizeof(object, [, default])`  
  返回对象的大小（以字节为单位）。
  该对象可以是任何类型。
  所有内建对象返回的结果都是正确的，但对于第三方扩展不一定正确，因为这与具体实现有关。

- `sys.getswitchinterval()`  
  返回解释器的以秒为单位的“线程切换间隔时间”

- `sys.getobjects(limit[, type])`  
  此函数仅当 CPython 使用专门的配置选项 `--with-trace-refs` 构建时才存在。 它仅针对调试垃圾回收问题而设计。

- `sys.getprofile()`  
  返回由 `setprofile()` 设置的性能分析函数。

- `sys.gettrace()`  
  返回由 `settrace()` 设置的跟踪函数。

- `sys.get_coroutine_origin_tracking_depth()`  
  获取由 `set_coroutine_origin_tracking_depth()` 设置的协程来源的追踪深度。

- `sys.hash_info`  
  一个具名元组，给出数字类型的哈希的实现参数。

- `sys.hexversion`  
  编码为单个整数的版本号。

- `sys.implementation`  
  一个对象，该对象包含当前运行的 Python 解释器的实现信息。

- `sys.int_info`  
  一个 具名元组，包含 Python 内部整数表示形式的信息。这些属性是只读的。

- `sys.intern(string)`  
  将 string 插入 "interned" 字符串表，返回被插入的字符串。

- `sys.is_finalizing()`  
  如果 Python 解释器 正在关闭 则返回 `True`，否则返回 `False`。

- `sys.last_exc`  
  该变量并非总是会被定义；当有未处理的异常时它将被设为相应的异常实例并且解释器将打印异常消息和栈回溯。
  它的预期用途是允许交互用户导入调试器模块并进行事后调试而不必重新运行导致了错误的命令。

- `sys.maxsize`  
  一个整数，表示 Py_ssize_t 类型的变量可以取到的最大值。在 32 位平台上通常为 `2**31 - 1`，在 64 位平台上通常为 `2**63 - 1`。

- `sys.maxunicode`  
  一个整数，表示最大的 Unicode 码点值

- `sys.modules`  
  这是一个字典，它将模块名称映射到已经被加载的模块。

- `sys.orig_argv`  
  传给 Python 可执行文件的原始命令行参数列表。

- `sys.path`  
  一个字符串组成的列表，用于指定模块的搜索路径。
  初始化自环境变量 PYTHONPATH，再加上一条与安装有关的默认路径。

- `sys.path_hooks`  
  一个由可调用对象组成的列表，这些对象接受一个路径作为参数，并尝试为该路径创建一个查找器。
  如果成功创建查找器，则可调用对象将返回它，否则将引发 ImportError 异常。

- `sys.path_importer_cache`  
  一个字典，作为 查找器 对象的缓存。

- `sys.platform`  
  本字符串是一个平台标识符，该标识符可用于将特定平台的组件追加到 sys.path 中。

- `sys.prefix`  
  一个指定用于安装与平台无关的 Python 文件的站点专属目录前缀的字符串。

- `sys.ps1`, `sys.ps2`  
  字符串，用于指定解释器的首要和次要提示符，仅当交互模式时才有定义。
  初值为 `>>>` 和 `...`。

- `sys.set_int_max_str_digits`  
  设置解释器所使用的 整数字符串转换长度限制。

- `sys.setprofile(profilefunc)`  
  设置系统的性能分析函数，该函数使得在 Python 中能够实现一个 Python 源码性能分析器。

- `sys.setrecursionlimit(limit)`  
  将 Python 解释器堆栈的最大深度设置为 limit。
  此限制可防止无限递归导致的 C 堆栈溢出和 Python 崩溃。

- `sys.setswitchinterval(interval)`  
  设置解释器的线程切换间隔时间（单位为秒）。
  该浮点数决定了“时间片”的理想持续时间，时间片将分配给同时运行的 Python 线程。

- `sys.settrace(tracefunc)`  
  设置系统的跟踪函数，使得用户在 Python 中就可以实现 Python 源代码调试器。
  该函数是特定于单个线程的，所以要让调试器支持多线程，必须为正在调试的每个线程都用 `settrace()` 注册一个跟踪函数，或使用 `threading.settrace()`。
