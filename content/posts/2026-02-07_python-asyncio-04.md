+++
date = '2026-01-26T8:00:00+01:00'
draft = true
title = 'Python Asyncio 04: Concurrent web requests'
categories = ['Note']
tags = ['Python', 'Asyncio']
+++

## Introducing aiohttp

aiohttp (Asynchronous HTTP Cilent/Server for asyncio and Python) 是一个解决非阻塞 socket.
aiohttp 是一个 aio-http 开源项目的一部分，其自称 "set of asyncio-based libraries built with high quality".
该库包含了完整的 web client/server 的功能，意味着它既可以发送 web 请求，也可以作为 web 服务器。
这里会着重介绍客户端的 aiohttp。

这里会使用到 asynchronous context manager 异步上下文管理器，该方法可以干净地发起和关闭 http 请求。

## Asynchronous Context Manager

在任何编程语言中，和资源交互都需要关闭和打开，例如文件。
当处理资源的时候，我们需要小心可能抛出的各种异常。
这是因为如果我们打开了一个资源，然后抛出了错误，我们可能永远无法执行释放的代码。
导致处理资源泄露的状态。

可以使用 `finally` 解决这个问题，即使不是特别 Pythonic:

```Python
file = open('example.txt')
try:
    lines = file.readlines()
finally:
    file.close()
```

这里代码解决了在 `file.readlines()` 出现异常的问题。
缺点是必须要记住将所有这类操作打包到 `try finally` 语句中，并且还要记住调用 `close()` 方法。

Python 有一种叫做 context manager 的语言特性，可以通过该方法抽象关闭逻辑：

```Python
with open('example.txt') as file:
    lines = file.readlines()
```

这种同步的不支持 coroutine 和 tasks，Python 引入了新的语言特征来支持，叫做 asynchronous context managers 异步上下文管理器。
异步上下文是实现了两种特殊方法的类，`__aenter__` 异步请求资源和 `__aexit__` 关闭资源。

```Python
import asyncio
import socket
from types import TracebackType
from typing import Optional, Type

class ConnectedSocket:
    def __init__(self, server_socket):
        self._connection = None
        self._server_socket = server_socket

    async def __aenter__(self):  # 进入 block 时调用协程，等待直到 client 连接并返回
        print('Entering context manager, waiting for connection')
        loop = asyncio.get_running_loop()
        connection, address = await loop.socket_accept(self._server_socket)
        self._connection = connection
        print('Accepted a connection')
        return self._connection

    async def __aexit__(  # 退出 block 时调用协程，清理资源，这里关闭连接
        self,
        exc_type: Optional[Type[BaseException]],
        exc_val: Optional[BaseException],
        exc_tb: Optional[TracebackType],
    ):
        print('Existing context manager')
        self._connection.close()
        print('Closed connection')

async def main():
    loop = asyncio.get_running_loop()

    server_socket = socket.socket()
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_address = ('127.0.0.1', 8000)
    server_socket.setblocking(False)
    server_socket.bind(server_address)
    server_socket.listen()

    async with ConnectedSocket(server_socket) as connection:  # 调用 __aenter__，等待客户端连接
        data = await loop.sock_recv(connection, 1024)
        print(data)  # print 之后会调用 __aexit__，然后关闭连接

asyncio.run(main())
```

假设使用 Telnet 发送 hi，会看到这样输出：

```text
Entering context manager, waiting for connection
Accepted a connection
b'hi\r\n'
Existing context manager
Closed connection
```

通常并不需要自己编写异步上下文管理器，但了解其工作原理很有帮助。

注意这里使用了更新的 `asyncio.get_running_loop()` 而不是 `asyncio.get_event_loop()`。
前者只会尝试去获取时间循环，后者会在没有事件循环的时候尝试创建事件循环，但这只限于主线程，其不会在子线程中创建事件循环。

### Making a web request with aiohttp

aiohtpp 和一般的网络请求都采用了 session 会话的概念。
在一个会话里面，会维护多个连接，每个连接都可以被重复使用。
这被称为 connection pooling 连接池。
连接池是基于 aiohttp 应用的重要概念，该功能能增加应用性能。
由于创建连接是资源密集的，创建可复用的连接池能够削减资源消耗。
一个 sesion 内部也会保存收到的 cookies，该功能可以自行关闭。

我们可以使用 `async with` 语句和 `aiohttp.ClientSession` 异步管理器：

```Python
import asyncio
import aiohttp
from aiohttp import ClientSession
from util import async_timed

BASE_URL = 'https://hacker-news.firebaseio.com/v0/'

@async_timed
async def fetch_status(session: ClientSession, url: str) -> int:
    async with session.get(url, ssl=False) as result:
        return result.status

@async_timed
async def main():
    async with aiohttp.ClientSession() as session:
        url = 'https://www.example.com'
        status = await fetch_status(session, url)
        print(f'Status for {url} was {status}')

asyncio.run(main())
```

上面访问 `https://www.example.com`，返回 200 响应码。
首先通过 `async with` 语句和 `aiohttp.ClientSession()` 创建一个会话。
一但创建了客户端会话，就可以发送任意网络请求。
在这个例子中，定义了一个获取状态码的方法 `fetch_status`，根据输入 session 和 URL，返回状态码。
在该函数内部，定义了另一个 `async with` 块，并使用 session 运行 `GET` HTTP 请求。

注意，`ClientSession` 默认会创建最多 100 个连接，为上线请求设置了隐性的上限。
要修改上限，需要创建一个 `TCPConnector` 实例并指定最大连接数量，然后将其传递给 `ClientSession`。

### Setting timeouts with aiohttp

之前介绍过如何使用 `asyncio.wait_for()` 设置超时，但更干净的一种做法是使用 aiohttp 提供的功能。
默认情况下，aiohttp 有 5 分钟超时，即没有任何一个操作应该超过这个时间。
如果要修改超时时间，可以对整个 session 的所有操作设置，或者对单独的每个请求设置超时。

使用 aiohttp 特有的 `ClientTimeout` 数据结构设置超时。
该结构不仅允许为整个请求指定以秒为单位的超时时间，还允许我们设置建立连接或读取数据的超时时间。

```Python
import asyncio
import aiohttp
from aiohttp import ClientSession


async def fetch_status(session: ClientSession, url: str) -> int:
    ten_millis = aiohttp.ClientTimeout(total=.01)  # GET 超时
    async with session.get(url, timeout=ten_millis) as result:
        return result.status

async def main():
    session_timeout = aiohttp.ClientTimeout(total=1, connect=1)  # 默认上限，握手限制
    async with aiohttp.ClientSession(timeout=session_timeout) as session:
        await fetch_status(session, 'https://www.example.com')

asyncio.run(main())
```

这里设置了总超时 1 秒，并明确将连接超时设置为 100 毫秒。
接着，在 `fetch_status` 函数中，我们针对 **get** 请求覆盖了这一设置，将总超时时间调整为 10 毫秒。
在这种情况下，如果向 `example.com` 发出的请求耗时超过 10 毫秒，在等待 `fetch_status` 时就会触发 `asyncio.TimeoutError`。
在这个例子下，10 毫秒应该足够让 `example.com` 的请求完成，因此不太可能看到异常。

## Running tasks concurrently, revisited

使用之前的函数举例说明如何一次进行多个任务

```Python
import asyncio
from util import async_timed, delay


@async_timed()
async def main() -> None:
    delay_times = [3, 3, 3]
    [await asyncio.create_task(delay(seconds)) for seconds in delay_times]

asyncio.run(main())
```

这里的问题很微秒，由于在创建 task 的时间就使用了 `await`。
这意味着，对于每一个我们创建的延迟任务，我们都会暂停列表推导式和主 coroutine 协程，直到该延迟任务完成。
这种情况下，任何给定时间只会运行一个任务，而不是多个。
要修复这个问题，可以在列表推导式中创建任务，并在第二个列表推导式中 `await`。

```Python
import asyncio
from util import async_timed, delay


@async_timed()
async def main() -> None:
    delay_times = [3, 3, 3]
    tasks = [asyncio.create_task(delay(seconds)) for seconds in delay_times]
    [await task for task in tasks]

asyncio.run(main())
```

这段代码正常运行是因为 `create_task` 会立刻返回。
并在所有的 task 创建成功之前没有 `await`，因此 `create_task` 能够立刻返回。
这确保了它最多只需要延迟时间中的最大暂停，运行时间约为 3 秒。

但这样做仍有一些缺陷：

1. 含有多行代码，必须显示分离 task 和 awaits
2. 灵活性差，如果某个 coroutine 比其他先完成，则必须要等待其他 coroutine 完成
3. 最大的问题是错误处理，如果某个 coroutine 出现了异常，则会抛出该异常，导致任务终止

## Running requests concurrently with gather

一个广泛使用的 asyncio API 函数是 `asyncio.gather`。
该函数接收一系列 `awaitable` 对象，然后将他们都打包到一个 task 里面并发运行。
这意味着我们无需将所有东西都分别包装到 `asyncio.create_task` 中去运行了。

`asyncio.gather` 返回一个 awaitable 对象，当我们 `await` 该对象会暂停直到所有的 task 执行完成。
一但所有的任务完成，`asyncio.gather` 会返回一个完成的列表。

下面举一个 1000 个请求的例子，并获取每个响应的响应码。
并使用 `@async_timed` 来获取每个请求消耗的时间。

```Python
import asyncio
import aiohttp
from aiohttp import ClientSession
from util import async_timed


@async_timed()
async def fetch_status(session: ClientSession, url: str) -> int:
    timeout = aiohttp.ClientSession(total=1)
    async with session.get(url, timeout=timeout, ssl=False) as result:
        print(f'status code: {result.status}')
        return result.status

@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        urls = ['https://example.com' for _ in range(1000)]
        requests = [fetch_status(session, url) for url in urls]
        status_code = await asyncio.gather(*requests)

asyncio.run(main())
```

首先创建大量的请求地址 urls，然后创建对每个地址的获取状态码请求 `fetch_status`，最后将他们都放到 `asyncio.gather()` 中。
这将每个 coroutine 包装成一个 task 并开始并发执行。
当执行该代码时，我们会看到 1000 条信息顺序输出，说 `fetch_status` 协程启动了。
结果会看到类似这样的输出 `Finished <function fetch_status at 0x1025fd3a0> in 0.5628 second(s)`。
一旦获取了所有请求的 URL 内容，就会看到状态码开始打印出来。

如果要和同步对比，修改 main 函数为阻塞的：

```Python
async def main():
    async def aiohttp.ClientSession() as session:
        urls = ['http://example.com' for _ in range(1000)]
        status_codes = [await fetch_status(session, url) for url in urls]
        print(status_codes)
```

同步的方式花了足足 221.2207 秒！

值得注意的一点是 `gather` 函数可能会按不确定的顺序完成，但好在它返回的结果是有顺序的。
下面通过 `delay` 函数来证明一下：

```Python
import asyncio
from util import delay

async def main():
    results = asyncio.gather(delay(3), delay(1))
    print(results)

asyncio.run(main())
```

上面两个 `delay()` 函数一个消耗 3 秒，一个消耗 1 秒。
按照时间顺序应该是得到结果 `[1, 3]`，但实际上是根据会得到我们输入顺序的结果 `[3, 1]`。
在幕后，`gather` 使用了一种特殊的 `future` 实现来完成这一操作。

上面介绍的代码都没有考虑失败的情况，但如果抛出异常了怎么办？

## Handling exceptions with gather

在发送网络请求的时候，有时候可能并不总是能够得到一个结果。
由于网络肯能是不可靠的，我们可能会得到一个异常，可能会有不同的故障情况。

`asyncio.gather` 提供了一个可选参数 `return_exceptions`，其允许指定异常处理方式。
`return_exceptions` 是一个布尔值，因此有两个可能的值：

- `return_exceptions=False` 这是默认值，这种情况下，如果产生了任何异常，`gather` 会在 `await` 的时候抛出一样的异常。
  然而，即使某个协程失效了，只要处理了异常，或者异常不会导致事件循环停止并取消任务，其他协程就不会取消，并继续运行。

- `return_exceptions=True` 在这种情况下，`gather` 会在 `await` 它时将任何异常作为返回结果列表的一部分返回。调用 `gather` 本身不会抛出任何异常，可以按照自己意愿处理所有异常。

这里使用一个错误的 url 地址举例说明：

```Python
@async_timed
async def main():
    async with aiohttp.ClientSession() as session:
        urls = ['https://example.com', 'python://example.com']
        tasks = [fetch_status(session, url) for url in urls]
        status_code = await asyncio.gather(*tasks)
        print(status_code)
```

由于 url 地址不合法，`fetch_status` 函数会产生 `NonHttpUrlClientError` 的报错。
该异常会在 `await` 的时候抛出，但请求继续执行了。

```text
Starting <function main at 0x10600f420> with args () {}
Starting <function fetch_status at 0x104659440> with args (<aiohttp.client.ClientSession object at 0x10617f230>, 'https://example.com') {}
Starting <function fetch_status at 0x104659440> with args (<aiohttp.client.ClientSession object at 0x10617f230>, 'python://example.com') {}
Finished <function fetch_status at 0x104659440> in 0.0000 second(s)
Finished <function fetch_status at 0x104659440> in 0.0027 second(s)
Finished <function main at 0x10600f420> in 0.0030 second(s)
Traceback (most recent call last):
...
aiohttp.client_exceptions.NonHttpUrlClientError: python://example.com
```

如果发生错误 `asyncio.gather` 并不会取消其他任务。
对于许多场景来说，这或许可以接受，但这也是 `gather` 的缺点之一。

还有一个潜在问题是，如果代码产生多个异常，将只能看到第一个异常结果。
可以使用 `return_exceptions=True` 修复这个问题，这会在运行协程的时候返回遇到的所有异常。
然后可以过滤出所有的异常并处理，下面使用之前的例子展示一下：

```Python
@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        urls = ['https//example.com', 'python://example.com']
        tasks = [fetch_status(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        exceptions = [res for res in results if isinstance(res, Exception)]
        successful_results = [res for res in results if not isinstance(res, Exception)]

        print()
        print(f'All results: {results}')
        print(f'Finished successfully: {successful_results}')
        print(f'Threw exceptions: {exceptions}')
        print()
```

运行结果如下：

```text
All results: [200, NonHttpUrlClientError(URL('python://www.example.com'))]
Finished successfully: [200]
Threw exceptions: [NonHttpUrlClientError(URL('python://www.example.com'))]
```

这样解决了只能看到一个错误的问题，并且也不用通过 try catch 捕获异常了。
但仍然需要从成功结果中筛选出异常，这显得有些笨拙。

`gather` 有一些缺陷，一个是不容易在出现报错的时候取消任务。
另一个是必须等待所有的协程处理完成才能得到结果。
asyncio 提供了解决这两个问题的 API，先来看看如何一有结果就立刻处理的问题。

## Processing requests as they complete

假如有 100 个请求，其中两个响应很慢，其他都迅速完成了，这时候使用 `gather` 则需要等所有的请求结束，并获取到结果。
为了应对这种情况，asyncio 有一个 `as_completed` 函数。
该方法接收一个 awaitable 对象列表，并返回一个未来对象的迭代器。
为了展示该方法如何工作，使用 `delay` 模型需要不同时间的请求。

```Python
async def fetch_status(session: ClientSession, url: str, delay: int = 0) -> int:
    await asyncio.sleep(delay)
    async with session.get(url, ssl=False) as result:
        return result.status
```

并使用 for 循环通过 `as_completed` 迭代：

```Python
import asyncio
import aiohttp
from aiohttp import ClientSession
from util import async_timed


@async_timed()
async def main():
    async with aiohttp.ClientSession as session:
        fetchers = [
            fetch_status(session, 'https://www.example.com', 1),
            fetch_status(session, 'https://www.example.com', 1),
            fetch_status(session, 'https://www.example.com', 10),
        ]
        for finished_task in asyncio.as_completed(fetchers):
            print(await finished_task)

asyncio.run(main())
```

在上面的例子中，我们创建了 3 个协程，两个耗时 1 秒，另一个耗时 10 秒。
然后将其传递给 `as_completed`，在底层，每个协程都被封装在一个 task 中，然后并发运行。
该函数立即返回一个开始循环遍历的迭代器。
当我们进入 for 循环，会遇到 `await finished_task`。
这里会暂停执行，直到第一个结果返回。
在这个例子中，我们的第一个结果在 1 秒后返回，另一个结果也马上会返回，然后过一会儿运行 10 秒的结果返回。

```text
Starting <function main at 0x106983420> with args () {}
Starting <function fetch_status at 0x105051440> with args (<aiohttp.client.ClientSession object at 0x105162270>, 'https://www.example.com', 1) {}
Starting <function fetch_status at 0x105051440> with args (<aiohttp.client.ClientSession object at 0x105162270>, 'https://www.example.com', 1) {}
Starting <function fetch_status at 0x105051440> with args (<aiohttp.client.ClientSession object at 0x105162270>, 'https://www.example.com', 10) {}
Finished <function fetch_status at 0x105051440> in 1.6038 second(s)
200
Finished <function fetch_status at 0x105051440> in 1.6136 second(s)
200
Finished <function fetch_status at 0x105051440> in 10.2043 second(s)
200
Finished <function main at 0x106983420> in 10.2080 second(s)
```

总的来说，完整耗时仍然是 10 秒，这和使用 `asyncio.gather()` 相同。
但能够在第一个请求完成后立即执行代码来打印其结果。
这让我们有额外时间来处理第一个成功完成的协程的结果，而其他协程仍在等待完成。
从而使的应用程序更加具有响应性。

该函数也提供了更好的异常处理机制。
当一个 task 抛出异常，可以在异常抛出后就立刻处理。

### Timeouts with as_completed

任何基于网络的请求都有耗时较长的风险。
服务器可能处于高资源负载状态，或者网络连接不佳。
之前介绍了如何给单个请求设置超时，但要是一堆请求呢？
`as_completed` 函数支持一个设置超时的位置参数，可以设置超时秒数。
这会去跟踪每个 `as_completed` 的调用时间，如果比超时时间更长，则会在 `await` 的时候抛出一个 `TimeoutException` 异常。

为了说明这种情况，下面使用之前例子，并为 `as_completed` 设置 2 秒的超时时间。
一但循环结束，就打印所有的任务结果。

```Python
import asyncio
import aiohttp
from aiohttp import ClientSession
from util import async_timed

@async_timed()
async def fetch_status(session: Session, url: str, delay: str) -> int:
    await asyncio.sleep(delay)
    async with session.get(url, ssl=False):
        return result.status

@async_timed()
async def main():
    async with aiohttp.ClientSession() as session:
        fetchers = [
            fetch_status(session, 'https://example.com', 1),
            fetch_status(session, 'https://example.com', 10),
            fetch_status(session, 'https://example.com', 10),
        ]
        for done_task in asyncio.as_completed(fetchers, timeout=2):
            try:
                result = await done_task
                print(result)
            except:
                print('Got an error!')
        for task in asyncio.tasks.all_tasks():
            print(task)

asyncio.run(main())
```

输出如下：

```text
Finished <function fetch_status at 0x1043a53a0> in 1.5893 second(s)
200
Got an error!
Got an error!
```

`as_completed` 函数在快速获取结果方面很好用，但也有缺点。
首先，虽然我们能够实时获取结果，但由于顺序完全不确定，无法轻松地看出当前正在等待的是哪个协程或任务。
如果不关心顺序，这可能没问题。
但如果要以某种方式将结果与请求关联起来，就会面临挑战。

其次是超时时间，虽然能够正确地抛出异常，但任何创建的 tasks 仍然会在后台运行。
由于无法知道是哪些 tasks 还在运行，于是就无法取消它们，这是另一个挑战。
如果这些是需要处理的问题，则需要更细粒度的知道哪些 awaitables 运行结束了，哪些没有。
为了应对这个问题，asyncio 提供了另一个 `wait` API 函数。

## Finer-grained control with wait

`gather` 和 `as_completed` 相同的问题是无法轻易地取消在运行中抛出异常的任务。
这种很多情况下或许可以接受，但设想这样一个场景：发起多个协程调用，如果第一个失败其他的也失败。
例如，网络请求传递无效参数或达到 API 速率限制。
这可能导致性能问题，因为我们会运行超出实际所需的任务，从而消耗更多资源。
`as_completed` 的另一个缺点是，由于迭代顺序不确定，很难去追踪哪个任务完成。

asyncio 中的 `wait` 和 `gather` 类似，其提供了这种情况下更加具体的控制方式。
改方法提供了多种选项，可根据期望结果的时间进行选择。
此外，该方法返回两组集合：一组 tasks 集合，要么已经完成，要么带有异常；另一组还在运行中的 tasks。
该函数还支持指定超时时间，和其他 API 行为不同的是，它不会抛出异常。
当需要的时候，该函数可以解决其他 asyncio 函数的一些问题。

`wait` 签名的本质是一系列 awaitable 对象的列表，伴随着可选的超时和 `return_when` 字符串。
该字符串有几个预定义的值：`ALL_COMPLETED`, `FIRST_EXCEPTION` 和 `FIRST_COMPLETED`，默认是 `ALL_COMPLETED`。
