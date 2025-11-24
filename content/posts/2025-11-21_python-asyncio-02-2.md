+++
date = '2025-11-21T8:00:00+08:00'
draft = false
title = 'Python Asyncio 02: Asyncio Basics Part 2'
tags = ['Python', 'Asyncio']
+++

## Tasks, coroutines, furtures, and awaitables

Coroutines 和 tasks 都是 await 表达式，那他们的相同线程是哪个？
下面介绍 `future` 也被称作 `awaitable`，理解 futures 是理解 asyncio 内部工作的重点。

### Introducing futures

Future 代表一个尚未完成的异步操作的最终结果。

```Python
from asyncio import Future


my_future = Future()
print(f"Is my_future done? {my_future.done()}")

my_future.set_result(42)
print(f"Is my_future done? {my_future.done()}")

print(f"What is the result of my_future? {my_future.result()}")
```

输出为

```text
Is my_future done? False
Is my_future done? True
What is the result of my_future? 42
```

使用构造器 `Future` 来创建 future，这时 future 没有值，因此调用 `done` 结果是 False。
然后使用 `set_result` 设置值，这将 future 标记为 done。
相似的，如果想要在 future 中设置异常，使用 `set_exception` 方法。

注意：在设置值之前如果调用 `result` 方法，会抛出 invalid state 的报错。

future 也可以在 await 表达式中使用，如果 await 一个 future，就是在说“暂停直到 future 被设置值，并且一但获取值后，就开始处理它”。

下面是一个网络请求的例子，该请求返回一个 future。
网络请求应该马上完成，但请求会消耗一点时间，在请求完成之前，future 此时并不会被定义。
后面一但请求完成，结果会被设置好，之后就能访问它了。

这个概念 JavaScript 里的 promises 很像，在 Java 中被称为 completable futures

```Python
import asyncio
from asyncio import Future


def make_request() -> Future:
    future = Future()
    asyncio.create_task(set_future_value(future))  # Create a task asynchronusly set the value of the future
    return future

async def set_future_value(future) -> None:
    await asyncio.sleep(1)  # waiting 1 second before setting the value of the future
    future.set_result(42)

async def main():
    future = make_request()
    print(f"Is the future done? {future.done()}")

    value = await future  # Pause main until the future's value is set
    print(f"Is the future done? {future.done()}")
    print(value)

asyncio.run(main())
```

输出如下

```text
Is the future done? False
Is the future done? True
42
```

实际上在 asyncio 的世界中，很少会需要处理 futures.
例如，会有一些返回 futures 的 asyncio API，和一些要求 futures 的基于回调的代码。
也可能会需要调试一些 asyncio API 代码，asyncio API 严重依赖 futures，因此理解其基本的工作方式很重要。

### The relationship between futures, tasks, and coroutines

实际上，task 直接继承于 future。

- future 可以被看成是一个一段时间内不会拥有的值
- task 可以被看作是 future 和 coroutine 的结合

当创建一个 task 的时候，实际上创建了一个运行 coroutine 的空 future。
当 coroutine 完成，将无论是结果还是 exceptinon 都会将其设置到 future。

Task 和 coroutine 都可以使用 await 关键字，他们都继承于 `Awaitable` 抽象基类 (abstract base class)。
该方法实现了一个抽象 dunder 双下划线 (double underscore) 方法 `__await__`。
coroutine 和 future 直接继承了 `Awaitable`，task 扩展了 future。

### Measuring coroutine execution time with decorators

首先，可以将每个 await 语句都包装起来，从而跟踪协程的开始和结束时间。

```Python
async def main():
    start = time.time()
    awati asyncio.sleep(1)
    end = time.time()
    print(f"Sleeping took {end - start} seconds")
```

但在有多个协程的情况下，这种方式就会十分混乱。
我们可以创建一个装饰器 decorator 来实现对每个协程在追踪，就叫做 `async_timed`。

装饰器是 Python 中的一种模式可以修改函数功能的同时，无需修改函数代码。

```Python
import time
import functools
from typing import Callable, Any


def async_timed():
    def wrapper(func: Callable) -> Callable:
        @functools.wraps(func)
        async def wrapped(*args, **kwargs):
            start = time.time()
            try:
                return await func(*args, **kwargs)
            finally:
                end = time.time()
                total = end - start
                print(f"finished {func} in {total:.4f} second(s)")
        return wrapped
    return wrapper
```

现在可以将装饰器作用于任何协程上，这样就能看到运行时间了

```Python
import asyncio
from util import delay, async_timed


@async_timed()
async def main():
    task_one = asyncio.create_task(delay(2))
    task_two = asyncio.create_task(delay(3))

    await task_one  # 注意：task 这里不要加括号
    await task_two

asyncio.run(main())
```

输出文本如下

```text
Starting <function main at 0x100625120> with args () {}
sleeping for 2 second(s)!
sleeping for 3 second(s)!
finished sleeping for 2 second(s)
finished sleeping for 3 second(s)
Finished <function main at 0x100625120> in 3.0014 second(s)
```

## The pitfalls of coroutines and tasks

上面例子可以看到，并发运行能够提升速度，但是如果只是简单地使用 `async` 来包装成任务，并不一定能够加速运行，反而有时候会降低程序性能。

1. CPU-bound 代码在协程/任务中未使用多进程
2. 阻塞 I/O-bound 没有使用多线程

### Running CPU-bound code

如果函数运行计算密集任务，例如遍历一个字典或数学计算，如果使用 tasks，将仍然受到 GIL 的限制：

```Python
import asyncio
from util import async_timed


@async_timed()
async def cpu_bound_work() -> int:
    counter = 0
    for _ in range(100_000_000):
        counter += 1
    return counter

@async_timed()
async def main():
    task_one = asyncio.create_task(cpu_bound_work())
    task_two = asyncio.create_task(cpu_bound_work())

    await task_one
    await task_two

asyncio.run(main())
```

输出是顺序的

```text
Starting <function main at 0x102bf7420> with args () {}
Starting <function cpu_bound_work at 0x102639120> with args () {}
Finished <function cpu_bound_work at 0x102639120> in 2.7105 second(s)
Starting <function cpu_bound_work at 0x102639120> with args () {}
Finished <function cpu_bound_work at 0x102639120> in 2.7229 second(s)
Finished <function main at 0x102bf7420> in 5.4336 second(s)
```

查看上面输出，可能会人为代码都没有任何问题。但实际上，这会造成性能下降。
尤其是在有其他 coroutines 或 tasks 的情况下。例如，`delay` 协程。

```Python
import asyncio
from util import async_timed, delay


@async_timed()
async def cpu_bound_work() -> int:
    counter = 0
    for _ in range(100_000_000):
        counter += 1
    return counter

@async_timed()
async def main():
    task_one = asyncio.create_task(cpu_bound_work())
    task_two = asyncio.create_task(cpu_bound_work())
    delay_task = asyncio.create_task(delay(4))

    await task_one
    await task_two
    await delay_task

asyncio.run(main())
```

输出如下

```text
Starting <function main at 0x1028e32e0> with args () {}
Starting <function cpu_bound_work at 0x10227d120> with args () {}
Finished <function cpu_bound_work at 0x10227d120> in 2.7227 second(s)
Starting <function cpu_bound_work at 0x10227d120> with args () {}
Finished <function cpu_bound_work at 0x10227d120> in 2.7483 second(s)
sleeping for 4 second(s)!
finished sleeping for 4 second(s)
Finished <function main at 0x1028e32e0> in 9.4727 second(s)
```

这里的 CPU-bound 任务会阻塞事件循环，这意味着任务会变成两个 2s+ 的 CPU 任务，和一个 4s 的 delay 任务相加。
如果想要在 CPU-bound 仍然使用 `async/await`，这是可行的，但需要使用 multiprocessing，并告诉 asyncio 在进程池中运行任务。
这个后面章节会介绍。

### Running blocking APIs

我可能将现存的 I/O-bound 的库包装成协程，然而这会导致同样的问题。
在协程中调用一个 blocking API 会阻塞 main thread，这意味着我们需要暂停其他所有执行中的任务或协程。

阻塞的例子有 `requests` 或 `time.sleep`，通常来说，任何执行 I/O 的非协程或执行耗时的 CPU 操作都可能造成阻塞。
下面以 `requests` 访问 www.baidu.com 为例，期望应该是用差不多一次的时间，完成 3 次访问请求。

```Python
import asyncio
import requests
from util import async_timed


@async_timed()
async def get_example_status() -> int:
    return requests.get("http://baidu.com").status_code

@async_timed()
async def main():
    task_1 = asyncio.create_task(get_example_status())
    task_2 = asyncio.create_task(get_example_status())
    task_3 = asyncio.create_task(get_example_status())

    await task_1
    await task_2
    await task_3

asyncio.run(main())
```

输出如下

```text
Starting <function main at 0x104d363e0> with args () {}
Starting <function get_example_status at 0x102621120> with args () {}
Finished <function get_example_status at 0x102621120> in 0.2021 second(s)
Starting <function get_example_status at 0x102621120> with args () {}
Finished <function get_example_status at 0x102621120> in 0.1157 second(s)
Starting <function get_example_status at 0x102621120> with args () {}
Finished <function get_example_status at 0x102621120> in 0.1039 second(s)
Finished <function main at 0x104d363e0> in 0.4222 second(s)
```

可以看到实际上花了差不多平均时间的 3 倍，这是因为 requests 库是阻塞的。
如果使用的库不返回一个协程，并且不是使用 `await` 在自己的协程中，那很可能会导致一个阻塞调用。

上面的例子中，可以使用 `aoihttp` 库，该库使用非阻塞 sockets 并且返回协程。
如果要使用 `requests` 库，需要告诉 asyncio 使用 multiprocessing 库的进程池执行器。

## Accessing and manually managing the event loop

目前，已经介绍了 `asyncio.run` 来方便地运行应用，并在后台创建事件循环。
但可能有一些 `asyncio.run` 提供的功能与需要的功能不符，例如让任何剩下的任务完成，而不是等待。
如果想要直接操作 sockets 或控制 tasks 调用在未来特定的时间运行，这将需要访问事件循环。

### Creating an event loop manually

可以使用 `asyncio.new_event_loop` 方法手动创建一个事件循环，这会创建一个事件循环实例。
通过这种方式，我们可以访问所有 event loop 提供的底层方法。

事件循环的 `run_until_complete` 方法接受一个协程，运行它直到完成。
一旦事件循环完成，我们需要关闭和释放资源。
这里应该有一个 `finally` 块，以防有任何 exceptinons 导致循环没有正常关闭。

```Python
import asyncio

async def main():
    await asyncio.sleep(1)

loop = asyncio.new_event_loop()

try:
    loop.run_until_complete(main())
finally:
    loop.close()
```

这段代码类似 `asyncio.run`，区别在于它不会取消任何剩余的任务。
如果需要任何特殊的清理逻辑，可以在 `finally` 块中实现。

### Accessing the event loop

有时我们需要访问正在运行的循环，asyncio 提供 `asyncio.get_running_loop` 函数来获取当前事件循环。
`call_soon` 将在事件循环的下一次迭代中调度一个函数运行。

```Python
import asyncio
from util import delay


def call_later():
    print("I'm being called in the future.")

async def main():
    loop = asyncio.get_running_loop()
    loop.call_soon(call_later)
    await delay(1)

asyncio.run(main())
```

若当前没有运行中的事件循环，调用此函数会创建一个新的事件循环，这可能导致奇怪的行为。
建议是使用 `get_running_loop`，在没有事件循环的时候会抛出一个报错，从而避免“惊喜”。

## Using debug mode
