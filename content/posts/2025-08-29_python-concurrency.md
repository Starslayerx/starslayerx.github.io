+++
date = '2025-08-29T8:00:00+08:00'
draft = false
title = 'Asyncio vs Gevents in Python'
categories = ['Blog']
tags = ['Python', 'Concurrency']
+++

python 中 asyncio 和 gevent 是两种协程(在一个线程内实现并发)的实现, 这篇文章对比介绍这两者实现.  
下面先介绍一下基础概念:

### Coroutines 协程

在 Python 中, 协程是可以暂停和继续运行的函数, 使得其是否适合并发编程. 定义使用 `async def` 语法, 协程运行编写非阻塞的操作. 在协程内, `await` 关键字用于暂停执行, 直到给定的任务完成, 从而运行其他协程在此其间并发运行.

### Event Loop 事件循环

事件循环是一种控制结构, 它不断地处理一系列事件, 处理任务并管理程序的执行流程. 等待事件发生, 处理后再等待下一个事件. 这种机制确保程序能够以高效有序的方式响应事件, 例如用户输入、计时器或者消息.

下面是事件循环如何管理协程:

- 任务提交: 当向事件循环提交一个协程时, 其被封装在一个 `Task` 对象中, 然后任务被安排在事件循环上运行.
- 内部队列: 事件循环使用几个内部数据结构来管理和调度这些任务
  - 就绪队列 (Ready Queue): 包含可以立即运行的任务.
  - I/O 选择器 (I/O Selector): 监控文件描述符, 并根据 I/O 准备情况调度任务
  - 计划回调 (Scheduled Callbacks): 管理计划在一定延迟后运行的任务.

- 调度: 事件循环不断检查这些队列和数据结构, 以确定哪些任务已准备好执行. 然后它运行这些任务, 在遇到 await 语句时, 根据需要暂停和恢复它们.
- 并发管理: 通过交错执行多个协程, 事件循环无需多个线程即可实现并发. 在任何时候, 只有一个任务会运行, 但如果一个任务是 I/O 密集型的, 它会切换到另一个任务, 给人一种并行的错觉.

### Asyncio In Action

```Python
import asyncio
import time

async def task1():
    print("Task 1 started")
    await asyncio.sleep(1)  # 将控制权让给事件循环
    print("Task 1 resumed")
    await asyncio.sleep(1)  # 将控制权让给事件循环

async def task2():
    print("Task 2 started")
    await asyncio.sleep(1)  # 将控制权让给事件循环
    print("Task 2 resumed")
    await asyncio.sleep(1)  # 将控制权让给事件循环

async def main():
    await asyncio.gather(task1(), task2())

start_time = time.time()
asyncio.run(main())
end_time = time.time()

print(f"Total time: {end_time - start_time:.2f} seconds")

'''
任务 1 启动，并使用 await asyncio.sleep(1) 让出控制权。
任务 2 启动，并使用 await asyncio.sleep(1) 让出控制权。
1 秒后，两个任务都恢复。
任务 1 恢复，并使用 await asyncio.sleep(1) 让出控制权。
任务 2 恢复，并使用 await asyncio.sleep(1) 让出控制权。
又过了 1 秒，两个任务都完成。
总耗时为 2 秒。
'''
```

通过上面的例子, 可以看到如何在进行 I/O 操作时通过切换任务来获得好处. 同样的逻辑如果按顺序执行需要 4 秒, 但使用 asyncio，可以将时间缩短一半.
在提供的代码中, 事件循环就像一个在单个线程上运行的管理器. 它跟踪 task1 和 task2 这样的任务, 确保它们轮流运行.
CPU 逐一处理这些任务, 但当一个任务等待某事时(例如使用 `await asyncio.sleep` 暂停), 它会将控制权交给事件循环.
这使得事件循环可以切换到另一个准备好运行的任务.
这样, 即使所有事情都在一个线程中发生, 任务也能高效且并发地执行, 而无需等待彼此完全完成.

**Asyncio 术语**

- `asyncio.run(coro)`
  - 运行主协程 `coro` 并管理事件循环, 创建一个新的事件循环, 运行协程直到完成, 然后关闭循环
  - 被设计用于异步函数外部, 通常在程序的入口点调用
  - 不能在已存在的事件循环内部运行

- `asyncio.create_task(coro)`
  - 安排协程 `core` 并发运行, 并返回一个 `Task` 对象, 这个函数对启动多个协程非常有用
  - 此命令需要一个已存在的事件循环才能执行
  - 用于启动一个应该与其他任务并发运行的协程, 非常适合需要与其他异步操作并行运行的任务

- `asyncio.gather(*coros)`
  - 并发运行多个协程并等待它们全部完成, 它将它们的结果收集到一个列表中
  - 需要一个活动的事件循环来管理协程

- `event_loop.run_untill_complete(core)`
  - 使用已存在的事件循环运行协程, 直到完成. 会阻塞直到协程完成并返回结果.
  - 不应该在异步函数内部使用, 它旨在运行协程直到其完成, 应在异步函数外部使用, 通常在同步上下文中

### Greenlets 和 Gevent

Coroutines 协程 和 greenlets(green threds 绿色线程) 都是管理并发执行的方法, 但它们在实现、控制和使用场景方面有明显的区别.  
Greenlets 是由 Python 的 greenlet 库提供的低级、用户空间协程实现

```Python
from greenlet import greenlet
import time

def task1():
    start_time = time.time()
    print("Task 1 started")
    time.sleep(1)  # 模拟工作
    print("Task 1 yielding")
    gr2.switch()  # 将控制权让给 task2
    print("Task 1 resumed")
    time.sleep(1)  # 模拟更多工作
    end_time = time.time()
    print(f"Task 1 completed in {end_time - start_time:.2f} seconds")

def task2():
    start_time = time.time()
    print("Task 2 started")
    time.sleep(1)  # 模拟工作
    print("Task 2 yielding")
    gr1.switch()  # 将控制权让给 task1
    print("Task 2 resumed")
    time.sleep(1)  # 模拟更多工作
    end_time = time.time()
    print(f"Task 2 completed in {end_time - start_time:.2f} seconds")

# 创建 greenlets
gr1 = greenlet(task1)
gr2 = greenlet(task2)

# 启动 task1 并切换到 task2
start_time = time.time()
gr1.switch()
gr2.switch()
end_time = time.time()

print(f"Total execution time: {end_time - start_time:.2f} seconds")

'''
Task 1 started
Task 1 yielding
Task 2 started
Task 2 yielding
Task 1 resumed
Task 1 completed in 3.01 seconds
Task 2 resumed
Task 2 completed in 3.01 seconds
Total execution time: 4.02 seconds
'''
```

Greenlet 在协作式多任务处理方式中为用户提供了完全的灵活性, 可以切换不同的执行上下文, 但它缺乏对异步 I/O 操作的内置支持.

Gevent 是一个构建在 Greenlet 之上的更高级的库, 提供对非阻塞 I/O 的内置支持和更高级的抽象, 适用于 I/O 密集型应用.
Gevent 抽象了上下文切换的复杂性, 并提供了对非阻塞 I/O 操作的内置支持.

```Python
import gevent
import time

def task1():
    print("Task 1 started")
    gevent.sleep(1)
    print("Task 1 resumed")
    gevent.sleep(1)

def task2():
    print("Task 2 started")
    gevent.sleep(1)
    print("Task 2 resumed")
    gevent.sleep(1)

start_time = time.time()

# 创建 greenlets
g1 = gevent.spawn(task1)
g2 = gevent.spawn(task2)

# 启动 greenlets 并等待它们完成
gevent.joinall([g1, g2])

end_time = time.time()

print(f"Total time: {end_time - start_time:.2f} seconds")

'''
Task 1 started
Task 2 started
Task 1 resumed
Task 2 resumed
Total time: 2.03 seconds
'''
```

下面对比 gevent 和 asyncio :

- 事件循环管理
  - Gevent: 管理自己的事件循环, 并依赖猴子补丁(monkey patching)使标准 I/O 操作变为异步. 这意味着它会修改标准库模块的行为以支持其并发模型
  - Asyncio: 包含一个内置事件循环, 它是 Python 标准库的一部分. 提供对管理异步操作的原生支持, 无需猴子补丁

- 猴子补丁
  - Gevent: 需要显式猴子补丁来将阻塞的 I/O 操作转换为非阻塞的. 这涉及修改标准库模块以与 gevent 的事件循环集成
  - Asyncio: 不需要猴子补丁, 它使用 Python 原生的 async/await 功能, 该功能与标准库的异步 I/O 操作无缝集成

- 性能
  - Gevent: 对于 I/O 密集型任务是高效的, 特别是在已经使用猴子补丁的系统中. 由于需要猴子补丁, 它可能会增加开销, 但在许多场景下仍然有效
  - Asyncio: 通常通过对异步编程的原生支持提供高性能. 它针对现代应用进行了优化, 并提供高效的 I/O 密集型任务处理, 没有猴子补丁的开销

- 错误处理
  - Gevent: 由于使用了猴子补丁的库, 可能需要仔细管理异常. 错误处理需要在 greenlets 内部进行管理
  - Asyncio: 在协程中利用标准的 Python 错误处理, 原生的语法使得在异步代码中处理异常更容易

- 使用场景
  - Gevent: 非常适合将异步行为集成到现有的同步代码库中, 或与兼容 greenlets 的库一起工作, 它适用于需要将现有 I/O 操作打补丁为异步的应用
  - Asyncio: 最适合采用现代异步编程实践的新应用或代码库, 它非常适合高性能网络应用和 I/O 密集型任务, 这些任务从原生异步支持中受益

Asyncio 通常是新应用的首选, 因为它倾向于现代异步编程实践, 它与 Python 的标准库无缝集成, 非常适合网络应用、实时通信和需要高并发的服务.  
Gevent 通常是现有同步代码库的首选, 这些代码库需要进行改造以支持并发, 它能够对标准库模块进行猴子补丁, 使其非常适合需要将阻塞的 I/O 操作转换为非阻塞的应用, 例如在网络服务器、聊天应用和实时系统中.

### Examples

下面是一些使用 asyncio 和 gevent 的例子

- Web Servers
  - FastAPI: 虽然主要基于 Starlette 和 Pydantic 构建, 但 FastAPI 利用 asyncio 来处理异步请求, 使其成为一个用于构建 API 的高性能 Web 框架
  - Gunicorn with gevent workers: 一个流行的 Python 应用 WSGI HTTP 服务器, 可以使用 gevent workers 来高效地处理大量并发连接
  - Flask with gevent: 尽管 Flask 本身是同步的, 但将其与 gevent 结合可以并发处理多个请求, 使其适用于实时应用

- 实时通信
  - Discord.py: 一个 Discord 的 API 封装库, 它使用 asyncio 来高效地处理实时事件和交互

- 网络工具
  - AsyncSSH: 一个用于 SSHv2 协议实现的库, 在 asyncio 之上构建, 为使用 SSH、SFTP 和 SCP 提供了异步 API
  - ZeroMQ with gevent: 对于需要高性能消息传递的应用, gevent 经常与 ZeroMQ 一起使用, 以有效地处理异步通信模式

- 数据库访问
  - Gevent with SQLAlchemy: 对于需要异步数据库访问的应用, 将 gevent 与 SQLAlchemy 结合可以处理数据库查询而不会阻塞主线程

### Wrapping Up

总而言之, asyncio 和 gevent 都提供了在 Python 中实现并发的强大工具, 但它们满足不同的需求和使用场景.
Asyncio 是新应用的绝佳选择, 它利用了 Python 的原生异步能力, 而 gevent 则擅长将异步行为集成到现有的同步代码库中, 尤其是在处理 I/O 密集型任务时.
具体使用哪种还是要根据不同的开发环境判断.
