+++
date = '2025-11-20T8:00:00+08:00'
draft = false
title = 'Python Asyncio 02: Asyncio Basics Part 1'
tags = ['Python', 'Asyncio']
+++

### Introducing coroutines

创建一个协程 coroutine 而不是创建一个函数类型，使用 `async def` 关键字，而不是 `def`:

```Python
async def coroutine_add_one(number: int) -> int:
    return number + 1

def add_one(number: int) -> int:
    return number + 1

function_result = add_one(1)
coroutine_result = coroutine_add_one(1)

print(f"Function result is {function_result} and the type is {type(function_result)}")
print(f"Coroutine result is {coroutine_result} and the type is {type(coroutine_result)}")
```

输出如下

```text
Function result is 2 and the type is <class 'int'>
Coroutine result is <coroutine object coroutine_add_one at 0x103000a00> and the type is <class 'coroutine'>
```

可以看到，协程返回的不是值，而是一个协程对象。
这里协程并没有执行，而是创建了一个协程对象可在之后运行，要运行一个协程则必须显式地在一个事件循环中运行它。
在 Python 3.7 之后的版本，必须创建事件循环来运行它。
asyncio 库添加了多个函数，抽象了事件循环的管理，例如 `asyncio.run()`，可以使用它来运行协程：

```Python
coroutine_result = asyncio.run(coroutine_add_one(1))
```

`asyncio.run()` 在此场景下运行了多个重要的事情。
首先，它创建了一个全新的时间循环，一但成功创建，就会去运行所有的协程直到结束，并返回结果。
该函数还会在主协程完成后清理任何可能仍在运行的内容。
等运行结束后，它会关闭并终止事件循环。

关于 `asyncio.run()` 最重要的是，它期望成为程序的主入口，它只执行一个协程。

使用 `await` 关键字会让后面的协程运行，不同于普通的函数调用，`await` 关键字可以暂停协程并获取到结果，而不是一个协程对象。

例如下面这样：

```Python
import asyncio

async def add_one(number: int) -> int:
    return number + 1

async def main() -> None:
    one_plus_one = await add_one(1)  # 暂停并等待 add_one(1) 结果
    two_plus_one = await add_one(2)  # 暂停并等待 add_one(2) 结果
    print(one_plus_one)
    print(two_plus_one)

asyncio.run(main())
```

### Introducing long-running coroutines with sleep

首先创建一个通用的 `delay()` 函数，这里在 `asynco.sleep()` 前后分别输出以观察暂停协程的时候，是否有其他代码在运行。

```Python
import asyncio

async def delay(delay_seconds: int) -> int:
    print(f"sleeping for {delay_seconds} second(s)")
    await asyncio.sleep(delay_seconds)
    print(f"finished sleeping for {delay_seconds} second(s)")
    return delay_seconds
```

现在将这个函数打包到 `util/delay_functions.py` 里面

```python
import asyncio
from util.delay_functions import delay

async def add_one(number: int) -> int:
    return number + 1

async def hello_world_message() -> str:
    await delay(1)
    return "Hello World!"

async def main() -> None:
    message = await hello_world_message()  # 1: Pause main until hello_world_message() returns
    one_plus_one = await add_one(1)  # 2: Pause main until add_one() returns
    print(one_plus_one)
    print(message)

asyncio.run(main())
```

上面这份代码实际上仍然会顺序运行，先暂停一秒，然后再加 1，因为 `await` 会完全暂停主函数。

如果想要暂停和加 1 并发运行，需要使用 tasks。

---

Tasks 是协程的 wrappers 他们会尽可能快地将协程调度到事件循环中去。
这种调度与执行以非阻塞方式进行，意味着一但创建任务，我们就可以在任务运行的同时执行任意其他代码。
使用 `await` 关键字的方法会阻塞代码，这意味着在获取返回结果前，该协程将完全被暂停。

下面创建两个任务

```Python
import asyncio
from util.delay_functions import delay

async def main():
    sleep_for_three = asyncio.create_task(delay(3))
    print(type(sleep_for_three))
    result = await sleep_for_three
    print(result)

asyncio.run(main())
```

输出如下

```
<class '_asyncio.Task'>
sleeping for 3 second(s)!
finished sleeping for 3 second(s)
3
```

1. task 类型是 `_asyncio.Task` 和 `coroutine` 不同
2. print 语句立刻就输出了 sleeping ...，如果使用 `await` 则要等 3 秒才会显示。
3. 在程序的运行环节中，应使用 `await` 关键字，如果不使用，虽然任务会运行，但是当 `asyncio.run` 结束的时候，任务会被立刻清理。

### Running multiple tasks concurrently

考虑到 tasks 被创建并安排后会尽可能快地开始运行，这将允许同时并发运行多个 tasks。

```Python
import asyncio
from util.delay_functions import delay


async def main():
    sleep_for_three = asyncio.create_task(delay(3))
    sleep_again = asyncio.create_task(delay(3))
    sleep_once_more = asyncio.create_task(delay(3))

    await sleep_for_three
    await sleep_again
    await sleep_once_more

asyncio.run(main())
```

`asyncio.create_task()` 会启动协程并立即让事件循环调度它，这里的 `await` 做的只是等待 task 完成，在等待期间，事件循环仍然会继续运行。
上面代码一共会消耗 3 秒的时间，如果启动 10 个任务，也只会消耗 3 秒多时间，这就比顺序执行快了 n 倍！

并且不止于此，在等待的时间里，还可以执行其他代码，例如下面每秒输出状态消息：

```Python
import asyncio
from util.delay_functions import delay


async def hello_every_second():
    for _ in range(2):
        await asyncio.sleep(1)
        print("I'm running other code while I'm waiting!")


async def main():
    first_delay = asyncio.create_task(delay(3))
    second_delay = asyncio.create_task(delay(3))
    await hello_every_second()
    await first_delay
    await second_delay

asyncio.run(main())
```

输出这样：

```text
sleeping for 3 second(s)!
sleeping for 3 second(s)!
I'm running other code while I'm waiting!
I'm running other code while I'm waiting!
finished sleeping for 3 second(s)
finished sleeping for 3 second(s)
```

上面存在的一种问题是，这些任务可能消耗不确定长的时间完成。
我们可能希望停止运行时间过长的任务，这可以通过取消来实现。

### Canceling tasks and setting tiemouts

网络连接是不可靠的，用户的网速也可能下降，web 服务器可能崩溃使得请求不可达。
在上面的例子中，一个 task 可能永远也不会结束，程序可能卡在 `await` 语句等待不存在的回复。

Task 对象都有一个 `cancel` 方法，任何时候想要停止任务就可以调用该方法。
取消 task 会导致一个 `CancelledError` 的报错，当 `await` 任务的时候，必须要能处理该错误。

下面假设最多运行 5 秒任务：

```Python
import asyncio
from asyncio import CancelledError
from util.delay_functions import delay

async def main():
    long_task = asyncio.create_task(delay(10))
    seconds_elapsed = 0

    while not long_task.done():
        print("Task not finished, checking again in a second.")
        await asyncio.sleep(1)
        seconds_elapsed = seconds_elapsed + 1
        if seconds_elapsed == 5:
            long_task.cancel()
    try:
        await long_task
    except CancelledError:
        print("Our task was cancelled.")

asyncio.run(main())
```

`done` 方法返回 bool 任务是否已经完成，还有很重要的一点就是，`CancelledError` 这个报错只能在 `await` 中抛出。
也就是说，调用 `cancel` 并不会立即停止该任务，只有在下一次 `await` 时才真正停止任务。

实际上，asyncio 提供了 `wait_for` 函数，该函数接受一个协程 coroutine 或任务 task，以及一个超时时间 timeout， 单位为秒。
如果时间超过了预设的 timeout，则会抛出一个 `TimeoutException`，超过阀值 threshold 后任务会自动取消。

下面例子中，有个运行两秒的例子，但是超时时间为 1 秒：

```Python
import asyncio
from util.delay_functions import delay


async def main():
    delay_task = asyncio.create_task(delay(2))
    try:
        result = await asyncio.wait_for(delay_task, timeout=1)
        print(result)
    except asyncio.exceptions.TimeoutError:
        print("Got a timeout.")
        print(f"Was the task cancelled? {delay_task.cancelled()}")

asyncio.run(main())
```

输出超时信息

```text
sleeping for 2 second(s)!
Got a timeout.
Was the task cancelled? True
```

有时候我们可能并不想直接取消任务，而是当超时后通知用户，这个可以通过 `asyncio.shield` 将任务包装起来实现。
该函数会避免传入的协程被取消，提供给它一个 "shield" 从而忽略掉取消请求。

```Python
import asyncio
from util import delay


async def main():
    task = asyncio.create_task(delay(10))

    try:
        result = await asyncio.wait_for(asyncio.shield(task), 5)  # wrap task with shield
        print(result)
    except TimeoutError:
        print("Task took longer than 5 seconds, it will finish soon!")
        result = await task
        print(result)

asyncio.run(main())
```

输出如下

```text
sleeping for 10 second(s)!
Task took longer than 5 seconds, it will finish soon!
finished sleeping for 10 second(s)
10
```
