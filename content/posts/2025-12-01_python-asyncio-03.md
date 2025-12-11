+++
date = '2025-12-01T8:00:00+08:00'
draft = false
title = 'Python asyncio 03: A first asyncio application'
tags = ['Python', 'Asyncio']
+++

## Working with blocking sockets

socket 是在网络中读取和写入数据的一种方式。
可以将 socket 看成一个邮件，将信封放到里面后运送到接收者的地址。

下面使用 Python 的内置 socket 模块来创建一个简单的 server

```Python
import socket

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
```

这里，给 socket 函数指定了两个参数，第一个是 `socket.AF_INET`，这个告诉我们要与什么类型的地址进行交互，在这个例子中是 hostname 和 phonenumber。
第二个是 `socket.SO_REUSEADDR`，这个参数是说我们使用 TCP 协议进行交互。

然后使用 `socket.setsockopt` 方法将 `socket.SOL_SOCKET` 标志设置为 1。这将允许在关闭和快速重启应用，避免 _address already in use_ 这类错误，如果不这样做将会消耗操作系统一段时间来解除与 port 的绑定。

使用 `socket.socket` 创建 socket 后，并不能开始沟通，因为还没有将其绑定到任何地址上面。
在本例中，将使用电脑本地地址 `127.0.0.1` 和任意 port 8000

```Python
server_address = ('127.0.0.1', 8000)
server_socket.bind(server_address)
```

这里将地址设置为 `127.0.0.1:8000`，这意味着 client 将能够使用该地址向服务器发送数据，如果要向 client 发送数据，也会看到该地址为来源地址。

接下来，在套接字上调用 `listen` 方法，主动监听来自客户端的连接请求。
随后，通过调用 `accept` 方法等待连接建立。
该方法会保持阻塞状态直至接收到连接请求，当连接成功时，将返回一个连接对象及客户端地址。
这个连接对象本质上是一个新的套接字，可以用于与客户端进行双向数据通信

```Python
server_socket.listen()
connection, client_address = server_socket.accept()
```

有了这些组件，我们便掌握了创建基于套接字的服务器应用所需的所有基础模块。
该应用将等待连接，并在建立连接后打印提示信息。

在收到请求后，将会打印一条信息并退出，下面使用 telnet cli 来发送请求。

## Connecting to a server with Telnet

Telnet 是 "teletype network" 的缩写，telnet 会与指定的服务器和主机建立一个 TCP 连接。
可以使用下面命令连接服务器

```shell
telnet localhost 8000
```

将会看到下面这样输出

```shell
Trying ::1...
Connection failed: Connection refused
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Connection closed by foreign host.
```

服务器会输出，表明已经建立了连接

```
I got a connection from ('127.0.0.1', 61137)!
```

### Reading and writing data to and from a socket

socket 有一个 `recv` 方法能够从中获取数据，该方法提供一个整数，代表在一段时间内想要读取的字节数。
这一点很重要，因为我们不能一次从 socket 中读取所有的数据，我们需要缓冲直到达到输入末尾。

当用户用 telnet 输入内容并按 Enter 时，telnet 会自动在你的输入后面追加两个字符

```Python
`\r\n` # 回车(carriage return) + 换行(line feed)
```

这里将 `socket.recv` 的缓冲区设得很小，这样更容易看到消息分片、拆包问题，从而观察 TCP 的行为。
实际开发中，recv 一般写成 1024，大的缓冲区更符合生产环境的性能要求。
使用较大的 buffer（比如 1024 bytes）可以减少应用程序反复调用 recv 的次数，TCP 数据会先在内核的 socket buffer 中缓存。

```Python
import socket


# AF_INET: IPv4
# SOCK_STREAM: 面向连接的字节流套接字，即 TCP
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # Create a TCP server socket

# SOL_SOCKET: 表示操作的是通用套接字级别的选项
# SO_REUSEADDR: 端口快速重用选项
# 1 表示启用
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

server_address = ('127.0.0.1', 8000)  # 将 socket 地址设置为 127.0.0.1:8000
server_socket.bind(server_address)  # 监听连接
server_socket.listen()

try:
    connection, client_address = server_socket.accept()  # 等待连接并为客户端分配一个信封邮箱
    print(f"I got a connection from {client_address}!")

    buffer = b''
    while buffer[-2:] != b'\r\n':
        data = connection.recv(2)
        # 连接已被对端关闭，返回空字节串 b''
        if not data:
            break
        else:
            print(f"I got data: {data}!")
            buffer += data
    print(f"All the data is: {buffer}")
finally:
    server_socket.close()
```

上面去循环检查 buffer 最后两个字节码是不是 `\r\n`，如果不是就读取两个字节码到缓存里面，直到读取最后的 `\r\n`。
socket 有一个 `sendall` 方法，该方法接受一条数据并将其写回客户端。

```Python
    buffer = b''
    while buffer[-2:] != b'\r\n':
        data = connection.recv(2)
        if not data:
            break
        else:
            print(f"I got data: {data}!")
            buffer += data
    print(f"All the data is: {buffer}")
    # 返回数据给客户端
    connection.sendall(b"Buffer: " + buffer)
```

现在应该在输入内容后，能够看到 `Buffer: <input>` 的内容，这样就完成了一个基础的 echo server。

现在这个应用每次能处理一个客户端，但是多个客户端可能会连接到单个 server socket。
下面来修改这个示例，从而允许多个客户端同时连接。

### Allowing multiple connections and the dangers of blocking

在监听模式下的 socket 同时运行多个 client 连接。
这意味着我们可以重复调用 `socket.accept`，并且每次客户端连接时，
都会获得一个新的连接套接字，用于与该客户端进行数据读写交互。
根据上面的知识，我们可以直接使用之前的例子来处理多个客户端。
一直循环调用 `socket.accept` 来监听新的连接。
每次得到一个新的连接，就将其插入到一个连接列表中，并遍历每个链接，接收输入的数据，并将数据写回客户端连接。

```Python
import socket


server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

server_address = ('127.0.0.1', 8000)
server_socket.bind(server_address)
server_socket.listen()

connections = []

try:
    while True:
        connection, client_address = server_socket.accept()
        print(f"I got a connection from {client_address}")
        connections.append(connection)
        for connection in connections:
            buffer = b''

            while buffer[-2:] != b'\r\n':
                data = connection.recv(2)
                if not data:
                    break
                else:
                    print(f"I got data: {data}!")
                    buffer += data
        print(f"All the data is: {buffer}")
        connection.send(buffer)
finally:
    server_socket.close()
```

尝试运行上面代码，并使用 telnet 建立两个连接，会立刻发现一个问题。
第一个客户端运行正常，消息按预期返回，但第二个客户端却接收不到任何回显。
这是由于 socket 的默认注释行为导致的，`accept` 和 `recv` 方法会一直阻塞，直到接收到数据为止。

这意味着，一但第一个客户端连接后，我们将会阻塞并等待第一条 echo message。
这导致其他客户端被阻塞，等待循环的下一次迭代，这种情况只有在第一个客户端发送数据后才会发生。

这明显不是用户希望的结果，用户超过一个人后很难扩展。
可以通过将 sockets 设置为 non-blocking 模式来解决这个问题。
将 socket 设置为非阻塞后，其方法将不会阻塞等待数据，而是继续执行后面的代码。

### Working with non-blocking sockets

在非阻塞模式下，如果套接字有数据准备就绪，那么我们将像使用阻塞套接字一样获得返回的数据。
如果没有，套接字会立即告诉我们它没有任何数据准备就绪，我们可以自由地继续执行其他代码。

```Python
import socket

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_socket.bind('127.0.0.1', 8000)
server_socket.listen()
server_socket.setblocking(False)
```

使用阻塞和非阻塞的 socket 本质上并没有什么不同，除了要设置 `setblocking(False)`

```Python
import socket


server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

server_address = ('127.0.0.1', 8000)
server_socket.bind(server_address)
server_socket.listen()
server_socket.setblocking(False) # non-blocking

connections = []

try:
    while True:
        connection, client_address = server_socket.accept()
        connection.setblocking(False) # non-blocking
        print(f"I got a connection from {client_address}")
        connections.append(connection)
        for connection in connections:
            buffer = b''

            while buffer[-2:] != b'\r\n':
                data = connection.recv(2)
                if not data:
                    break
                else:
                    print(f"I got data: {data}!")
                    buffer += data
        print(f"All the data is: {buffer}")
        connection.sendall(buffer)
finally:
    server_socket.close()
```

但直接运行会产生一个 `BlockingIOError` 错误，因为服务器还未建立连接，因此还没有数据要处理

```shell
  File "/Users/starslayerx/GitHub/book_asyncio/socket_server.py", line 16, in <module>
    connection, client_address = server_socket.accept()
                                 ^^^^^^^^^^^^^^^^^^^^^^
  File "/Library/Frameworks/Python.framework/Versions/3.12/lib/python3.12/socket.py", line 295, in accept
    fd, addr = self._accept()
               ^^^^^^^^^^^^^^
BlockingIOError: [Errno 35] Resource temporarily unavailable
```

这是套接字一种有点反直觉的表达方式，意思是说：“我现在没有任何数据，你等会儿再来调用我。”
这里没有简单的方法来判断 socket 是否有数据，所以一种解决方案就是捕获这个异常，把它忽略掉，然后不断循环，直到拿到数据。
采用这种策略，将会以尽可能快的速度不断检查是否有新的连接和数据。

```Python
import socket


server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

server_address = ('127.0.0.1', 8000)
server_socket.bind(server_address)
server_socket.listen()
server_socket.setblocking(False)  # non-blocking

connections = []
try:
    while True:
        try:
            connection, client_address = server_socket.accept()
            connection.setblocking(False)  # non-blocking
            print(f"I got a connection from {client_address}")
            connections.append(connection)
        except BlockingIOError:
            pass

        for connection in connections:
            try:
                buffer = b''

                while buffer[-2:] != '\r\n':
                    data = connection.recv(2)
                    if not data:
                        break
                    else:
                        print(f"I got data: {data}")
                        buffer += data
                print(f"All the data is: {buffer}")
                connection.send(buffer)
            except BlockingIOError:
                pass
finally:
    server_socket.close()
```

这样 `accept` 和 `recv` 就不会阻塞了，并且每次要么忽略，要么处理对应的数据。
循环的每次迭代都迅速完成，且从不依赖任何外部数据来推进下一行代码的执行。
这解决了阻塞服务器的问题，使得多个客户端能够同时连接并发送数据。

这种方法能够工作，但是有代价的。

- 每当可能尚未获取数据时就捕获异常，不仅会使代码变得冗长，还可能容易出错。
- 第二个是资源问题，这个程序会使用几乎 100% 的 CPU 处理能力，因为不断的循环和获取报错导致 CPU 较高的工作负载。

在之前提到过操作系统特有的事件通知系统，它能在 socket 有数据可操作的时候通知我们。
这些系统依赖于硬件层面的通知机制，而非刚才使用了轮询循环。
Python 内置了一个库来调用这种事件循环通知系统，下面将用其来解决 CPU 占用率问题，并构建一个套节字事件的小型事件循环。

## Using the selector module to build a socket event loop

操作系统提供了高效的 API，允许我们监听 socket 以接收数据和其他事件。
虽然具体的 API 取决于系统（例如 kqeueu、epoll 和 IOCP），但这些 I/O 通知系统都基于相似的概念运行。
我们向系统提供一个需要监控事件的套节字列表，操作系统在套节字有数据时会明确通知我们，而不是去不断轮询检查每个套接字是否有数据。

由于这是硬件层面实现的，在监控中需要很少的 CPU 利用率，产生高效的资源利用。
这些通知系统是 asyncio 实现并发的核心，理解其工作原理能够帮助我们理解 asyncio 的工作方式。

这些 event notification system 在不同的系统中是不同的。
好在 Python 的 `selectors` 模块将底层抽象，无论在那个系统上运行，都能正确获取事件。

该库暴露一个抽象基类型 `BaseSelector`，该类型对不同的事件通知系统有着不同的实现。
同时，库中还包含 `DefaultSelector` 类，能够自动为当前系统选择最高效的实现方案。

`BaseSelector` 类有一些重要的概念：

1. _registration_: 注册，当我们有一个感兴趣的套接字，希望获取其通知时，我们会将其注册到选择器中，并告之需要关注哪些事件。
   这些事件有 read 和 write，相反，也可以取消不感兴趣的注册。
2. _select_: select 选择器会阻塞直到有事件发生，一但事件触发，该调用将返回一个包含带处理的列表以及它触发的事件。
   它还支持超时设置，在指定时间后若无事件则返回空的事件集和。

通过上面的方法，我们能够创建一个不会压垮 CPU 的非阻塞 echo server。
一旦我们创建了服务器套接字，就会将其注册到默认的选择器 selector 中，该选择器将监听来自客户端的任何连接。
之后在任何时间某人连接到我们的 server socket，我们将会注册客户端 connection socket，并使用 selector 观察任何数据发送。

如果获取到任何不是来自 server socket 的数据，我们就知道是 client 发送了数据，然后我们收到数据并将其写入客户端。

```Python
import selectors
import socket
from selectors import SelectorKey


selector = selectors.DefaultSelector()

server_socket = socket.socket()
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

server_address = ("127.0.0.1", 8000)
server_socket.setblocking(False)
server_socket.bind(server_address)
server_socket.listen()

selector.register(server_socket, selectors.EVENT_READ)

while True:
    # 创建 1 秒过期的选择器
    events: list[tuple[SelectorKey, int]] = selector.select(timeout=1)

    # 没有事件
    if len(events) == 0:
        print("No events, waiting a bit more!")

    for event, _ in events:
        # 获取 socket event
        event_socket = event.fileobj

        if event_socket == server_socket:
            connection, address = server_socket.accept()
            connection.setblocking(False)
            print(f"I got a connection from {address}")
            selector.register(connection, selectors.EVENT_READ)
        else:
            data = event_socket.recv(1024)
            print(f"I got some data: {data}")
            event_socket.send(data)

```

这样实现的 echo server 的 CPU 利用率就少了很多，虽然仍然是死循环，但是循环内部的 `selector()` 会让线程进入内核的阻塞睡眠状态，直到超时才会 print() 一条语句，是典型的事件驱动。

而 try except 轮询会不间断的轮询，且内部会不断抛出异常、处理异常、重试等，CPU 占用很高。

---

上面构建的部分和 asyncio 底层的大部分工作方式类似。
在这个例子中，events 是 sockets 接收数据。
无论是我们的事件循环还是 asyncio 的事件循环，它们的每一次迭代都是由两种情况触发的：要么有某个 socket 事件发生，要么是超时导致事件循环继续运行。

在 asyncio 的事件循环里，只要发生了这两种情况中的任意一种，所有正在等待调度的协程都会运行，直到它们结束，或者它们遇到下一个 await 语句为止。

当协程执行到一个基于非阻塞 socket 的 await 时，该 socket 会被注册到系统的 selector 中，同时事件循环会记录该协程已暂停并正在等待这个 socket 的结果。

我们可以把这个概念翻译成伪代码来展示。

```Python
paused = []
ready = []

while True:
    paused, new_sockets = run_ready_tasks(ready)
    selector.register(new_sockets)
    timeout = calculate_timeout()
    events = selector.select(timeout)
    ready = process_events(events)
```

我们会运行所有“已经准备好的协程”，直到他们在某个 await 语句上暂停，并把这些协程放到 paused 列表中。
同时，还会记录这些协程运行过程中产生的所有新 socket，并将他们注册到 selector 中。
之后，我们需要计算下一次调用 select 时的超时时间。
这个 timeout 的计算方式比较复杂，但通常取决于在未来某个时间点或者等待某个持续时间后要执行的任务。
例如 asyncio.sleep() 就会影响这个 timeout。

接着，调用 select 并等待 socket 事件或超时。
当其中一个发生时，会处理这些事件，并将其转化为一个可立即继续执行的协程列表。

---

虽然上面的 event loop 只是用于 socket 的，但其展示了使用 selectors 注册 sockets 的主要思想，
即只在我们关心的事件发生后才启动。

然而，如果我们仅使用 selectors 来构建应用程序，就需要自行实现事件循环才能达到与 asyncio 相同的功能。
下面介绍如何使用 async/await 来实现上面功能。

## An echo server on the asyncio event loop

使用 `select` 对于很多应用来说有些太底层了。
我们可能希望在等待 socket 的时候，让代码在后台运行，或者我们可能希望按计划执行后台任务。
如果只使用 selectors 来实现这个，我们将需要构建自己的事件循环，与此同时，asyncio 有一个完整的实现可以使用。
此外，coroutines 和 tasks 在 selectors 之上提供了抽象层，这使得我们代码更易于实现和维护，无需考虑 selectors 细节。

下面通过 asyncio 的 coroutines 和 tasks 再次实现前面的 echo server。
这里仍然会通过底层 API 来实现，这些 API 会返回 coroutines 和 tasks。

### Event loop coroutines for sockets

考虑到 sockets 的一个相对底层的概念，处理他们的方法是通过 asyncio 的事件循环。
下面会使用三种主要的协程处理：

- `sock_accept`
- `sock_recv`
- `sock_sendall`

这些方法和之前的很类似，不同在于这些方法会将 socket 作为一个参数输入，这样我们可以 `await` 返回的协程，直到我们得到可以作用于其上的数据。

下面先从 `sock_accept` 开始，这个协程类似之前的 `socket.accept` 方法。

该方法返回一个 tuple: `(socket_connection, client_address)`，传入感兴趣的 socket，然后 `await` 等待连接返回。
一但接受该协程就能获取到连接与地址，这个 socket 必须是非阻塞的，并和一个端口绑定起来：

```Python
connection, addresss = await loop.socke_accept(socket)
```

`sock_recv` 和 `sock_sendall` 也是类似的，输入一个 socket，然后 `await` 等待结果。

- `sock_recv` 会等待直到有可以处理的字节
- `sock_sendall` 同时接受一个 socket 和要发送的 data，它会等待直到所有数据成功发送至 socket，并在成功后返回 `None`

```Python
data = await loop.sock_recv(socket)
success = await loop.sock_sendall(socket, data)
```

### Designing an asyncio echo server

之前介绍了 coroutines 和 tasks，那么什么时候使用 coroutine，什么时候使用 task 呢？
让我们来审视一下，我们希望应用程序如何表现以做出这一判断。

我们从如何监听应用连接开始。
当监听应用连接的时候，一次将只能处理一个连接，因为 `socket.accept` 只会给客户端一个连接。
如果有多个连接到达，后续的连接会被存储到一个被称作 `backlog` 的队列里面。

由于不需要并发处理多个连接，单个协程循环就足够了。
这样能够让其他代码在等待连接的时候并发执行。
这里定义一个一直循环的协程 `listen_for_connections`

```Python
async def listen_for_connections(server_socket: socket, loop: AbstractEventLoop):
    while True:
        connection, address = await loop.sock_accept(server_socket)
        connection.setblocking(False)
        print(f"Got a connection from {address}")
```

这样就有了一个监听连接的协程，由于要并发处理多个 connection，因此这里要为每个 connection 创建一个 task 来读写数据。

这里将创建负责处理数据的协程 `echo`，这个协程会一直循环来接收来自 client 的数据，一但其收到数据，就写会到 client 中去。
然后在 `listen_for_connections` 里，创建一个 task 来包装 `echo` 协程。

```Python
import asyncio
import socket
from asyncio import AbstractEventLoop


async def echo(connection: socket, loop: AbstractEventLoop) -> None:
    while data := await loop.sock_recv(connection, 1024):
        await loop.sock_sendall(connection, data)

async def listen_for_connections(server_socket: socket, loop: AbstractEventLoop):
    while True:
        connection, address = await loop.sock_accept(server_socket)
        connection.setblocking(False)
        print(f"Got a connection from {address}")

        asyncio.create_task(echo(connection, loop))

async def main():
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    server_address = ("127.0.0.1", 8000)
    server_socket.setblocking(False)
    server_socket.bind(server_address)
    server_socket.listen()

    await listen_for_connections(server_socket, asyncio.get_event_loop())

asyncio.run(main())
```

架构如下：

- 协程 `listen_for_connections` 持续监听连接，收到连接后该协程就会切换到 `echo` task 去处理每个连接
  - Client 1 echo task <--Read/Write--> Client 1
  - Client 2 echo task <--Read/Write--> Client 2
  - Client 3 echo task <--Read/Write--> Client 3

这样设计的 echo server 实际上有一个问题，下面来解决

### Handling errors in tasks

网络连接通常都是不可靠的，我们可能得到非预期的报错。
下面修改 echo 的实现，添加一个错误处理的代码：

```Python
async def echo(connection: socket, loop: AbstractEventLoop) -> None:
    while data := await loop.recv(connection, 1024):
        if data == b"boom\r\n":
            raise Exception("Unexcepted network error")
        await loop.sock_sendall(connection, data)
```

现在只要发送 boom 就会导致下面这样的报错：

```text
Got a connection from ('127.0.0.1', 49470)
Task exception was never retrieved
future: <Task finished name='Task-2' coro=<echo() done, defined at /Users/starslayerx/GitHub/book_asyncio/asyncio_echo_server.py:4> exception=Exception('Unexcepted network error')>
Traceback (most recent call last):
  File "/Users/starslayerx/GitHub/book_asyncio/asyncio_echo_server.py", line 7, in echo
    raise Exception("Unexcepted network error")
```

这里的重点在于 `Task exception was never retrieved`。
当一个异常在 task 内部被抛出时，这个任务会被视为已完成，并且它的“结果”就是这个异常。
这意味着异常不会沿着调用栈向上传递。
此外，这里没有任何清理逻辑。
如果该异常被抛出，则无法对任务失败做出反应，因为从未获取 retrieve 这个异常。

要让异常真正传递，必须在 await 表达式中使用 task。
当 await 一个失败的 task 时，异常会在 await 的地方重新抛出，其 traceback 也会在该位置体现。
如果在程序中从未 await 一个 task，就有可能永远看不到这个 task 抛出的异常。

下面演示，与其忽略在 `listen_for_connection` 里面创建的 echo tasks，我们通过列表来跟踪他们

```Python
tasks = []
async def listen_for_connection(server_socket: socket, loop: AbstractEventLoop):
    while True:
        connection, address = await loop.socket_accept(server_socket)
        connection.setblocking(False)
        print(f"Got a connection from {address}")
        tasks.append(
            asyncio.create_task(echo(connection, loop))
        )
```

看上去可能会和之前一样，如果输入 boom 应该会看到警告被打印出来，同时伴随着从未获取任务异常的警告。
然而，实际上并不是这样，除非强制终止程序，否则什么都看不到。

这样因为我们保留了 task 的引用，asyncio 只有在任务被垃圾回收的之后才能打印出 traceback 或 failed task 相关信息。
这是因为无法判断该任务是否会在应用程序的其他某个时刻被等待，从而可能引发异常。
鉴于这种复杂性，要么 await tasks，要么处理所有可能的异常。

在 echo server 中，首先可以做的是使用 try/catch 语法，记录下异常并关闭连接：

```Python
import logging

async def echo(connection: socket, loop: AbstractEventLoop) -> None:
    try:
        while data := await loop.sock_recv(connection, 1024):
            print("got data!")
            if data == b"boom\r\n":
                raise Exception("Unexcepted network error")
            await loop.sock_sendall(connection, data)
    except Exception as ex:
        logging.exception(ex)
    finally:
        connection.close()
```

这将解决因异常导致的直接问题，即服务器会报错任务异常未被捕获，因为我们在协程内部已自行处理。
这也将在 finally 块中正确关闭 socket，因此不会留下一个未关闭的悬空异常。

该实现会在应用程序关闭时，正确关闭所有已建立的客户端连接。
在之前章节说过，asyncio.run 会取消未完成的 tasks。
如果 await 这个 task 则会产生一个 CancelledError。

这里的重点是要注意异常是在何处产生的。
如果你的任务是在等待一个类似 `await loop.sock_recv` 的语句，并且取消那个 task，这行就会产生一个 `CancelledError`。
这意味着对于上述代码，在任务被取消时 finally 块依然会执行，因为取消任务时我们就在 await 语句上抛出了一个异常。
如果我们修改 except 代码，使其捕获并记录 CancelledError，你将会看到每个任务都会产生一个 CancelledError。

上面已经解决了 echo tasks 失败的情况，那如果应用程序关闭后要如何处理这些失败的 tasks 呢？
下面介绍 asyncio signal handlers

## Shutting down gracefully

如果我们想要关闭我们的应用程序怎么办？
我们可以向应用添加自定义关闭逻辑，任何还在运行中的 tasks 都能在几秒内发送需要的信息。

> 本篇内容不适用 Windows 系统:
> [https:// stackoverflow.com/questions/35772001](https:// stackoverflow.com/questions/35772001)

### Listening for signals

Singals 信号是一个 Unix 系统的概念。
例如使用 Ctrl-C 来关闭一个命令行工具就是发送了信号 SIGINT (singal interrupt)，这和在 Python 中捕获 KeyboardInterput 异常一样。
另一个常见的信号是 SIGTERM，这是使用 kill 命令杀死某个进程的信号。

为了实现自定义关闭逻辑，将会监听 SIGINT 和 SIGTERM 信号。
并在监听器中实现允许任何 echo tasks 能够在几秒内完成。

asyncio 事件循环允许我们通过 `add_signal_handler` 方法直接监听任何事件。
这与 signal 模块中的 `signal.signal` 函数设置的信号处理器不同。
`add_signal_handler` 能够安全地与事件循环交互。
该函数接收一个信号和一个函数，当监听到相应信号后会调用传入的函数。

下面编写一个 signal handler 来取消所有运行中的 tasks。
asyncio 有个方便的 tasks 叫做 `asyncio.all_tasks`。

```Python
# Adding a signal handler to cancel all tasks
import asyncio
import signal
from asyncio import AbstractEventLoop


from util.delay_functions import delay


def cancel_tasks():
    print("Got a SIGNAL!")
    tasks: set[asyncio.Task] = asyncio.all_tasks()
    print(f"Cancelling {len(tasks)} task(s).")
    [task.cancel() for task in tasks]

async def main():
    loop: AbstractEventLoop = asyncio.get_running_loop()
    loop.add_signal_handler(signal.SIGINT, cancel_tasks)
    await delay(10)

asyncio.run(main())
```

从 Python 3.11 版本开始，`asyncio.run()` 的设计趋于严格，不再允许 `asyncio.run()` 管理的事件训练里添加信号处理器。

因此不同版本行为会不一致，上面代码在 Python 3.12 需要这样写

```Python
import asyncio
import signal

from util.delay_functions import delay


def cancel_tasks():
    print("Got a Signal!")
    for task in asyncio.all_tasks():
        task.cancel()

async def main():
    try:
        await delay(10)
    except asyncio.CancelledError:
        print("Main task cancelled.")

loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)
loop.add_signal_handler(signal.SIGINT, cancel_tasks)
loop.run_until_complete(main())
loop.close()
```

### Waiting for pending tasks ot finish

在原始的问题里，希望 echo server 的 tasks 能够在关闭前再运行几秒钟。
一种实现方式就是使用 `wait_for` 包装并 await 所有的 tasks。
如果任务超时，则会产生一个 TimeoutError，之后就可以终止程序。

在 shutdown handler 里的一个问题是，这是一个普通的函数，无法在其内部 await。
一种解决方法就是创建一个协程来处理关闭逻辑，并将其包装成一个 task：

```
async def await_all_tasks():
    tasks = asyncio.all_tasks()
    [await task for task in tasks]

async def main():
    loop = asyncio.get_event_loop()
    # Warp into a task
    loop.add_signal_handler(singal.SIGINT, lambda: asyncio.create_task(await_all_tasks()))
```

这样虽然能工作，但问题是如果 await_all_tasks 内触发了一个异常，将会产生一个孤儿 task，并抛出 "exception was never retrieved" （异常未捕获）的警告，可能隐藏潜在的错误。

我们可以通过抛出一个自定义异常来停止主协程的运行，从而处理这个问题。
这样，当运行主协程时，我们可以捕获这个异常并执行任何关闭逻辑。
要实现这个功能，需要我们自定义事件循环，而不是使用 asyncio.run。
这是因为在 asyncio.run 中会取消所有在运行中的 tasks，这意味着我们不能将 echo tasks 给 wrap 包装进 `wait_for` 中：

```Python
class GracefulExit(SystemExit):
    pass

def shutdown():
    raise GracefulExit

loop = asyncio.get_event_loop()
loop.add_signal_handler(signal.SIGINT, shutdown)

try:
    loop.run_until_complete(main())
except GracefulExit:
    loop.run_until_complete(close_echo_tasks(echo_tasks))
finally:
    loop.close()
```

顺着上面的思路来编写关闭逻辑

```Python
async def close_echo_tasks(echo_tasks: list[asyncio.Task]):
    waiters = [asyncio.wait_for(task, 2) for task in echo_tasks]
    for task in waiters:
        try:
            await task
        except asyncio.exceptions.TimeoutError:
            # Except a timeout error here
            pass
```

在 `close_echo_tasks` 中，我们使用一个 echo tasks 的列表，并将其包装到 `wait_for` task 里，设置 2 秒的超时时间。
这意味着，在我们取消这些任务前，他们将会有 2 秒时间结束任务。
我们捕获在这两秒任务中任何的 TimeoutErrors。

结合上面说的所有逻辑，echo server 的关闭逻辑总体如下：

```Python
import asyncio
from asyncio import AbstractEventLoop
import socket
import logging
import signal


async def echo(connection: socket.socket, loop: AbstractEventLoop) -> None:
    try:
        while data := await loop.sock_recv(connection, 1024):
            print("got data!")
            if data == b"boom\r\n":
                raise Exception("Unexpected network error")
            await loop.sock_sendall(connection, data)
    except Exception as ex:
        logging.exception(ex)
    finally:
        connection.close()

echo_tasks = []

async def connection_listener(server_socket, loop):
    while True:
        connection, address  = await loop.sock_accept(server_socket)
        connection.setblocking(False)
        print(f"Got a connection from {address}")
        echo_task = asyncio.create_task(echo(connection, loop))
        echo_tasks.append(echo_task)

class GracefulExit(SystemExit):
    pass

def shutdown():
    raise GracefulExit()

async def close_echo_tasks(echo_tasks: list[asyncio.Task]):
    waiters = [asyncio.wait_for(task, 2) for task in echo_tasks]
    # 这里实际上会串行等待
    for task in waiters:
        try:
            await task
        except asyncio.exceptions.TimeoutError:
            pass
    # 并行等待
    # results = await asyncio.gather(*waiters, return_exceptions=True)

async def main():
    server_socket = socket.socket()
    try:
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

        server_address = ("127.0.0.1", 8002)
        server_socket.setblocking(False)
        server_socket.bind(server_address)
        server_socket.listen()

        for signame in {"SIGINT", "SIGTERM"}:
            loop.add_signal_handler(getattr(signal, signame), shutdown)
        await connection_listener(server_socket, loop)
    finally:
        server_socket.close()

loop = asyncio.new_event_loop()

try:
    loop.run_until_complete(main())
except GracefulExit:
    loop.run_until_complete(close_echo_tasks(echo_tasks))
finally:
    loop.close()
```

这段代码是一个支持优雅关闭 Graceful Shutdown 的异步 TCP Echo Server。

核心思想：当收到关闭信号时，不是立刻终止，而是给正在处理的连接一些时间来完成工作。

- `echo`: Echo 处理函数，接收数据原样返回，当接收到 "boom\r\n" 时模拟网络错误
- `cancel_listener`: 连接管理器，异步等待新连接，为每个连接创建一个 task
- `shutdown`: 优雅关闭机制，通过自定义异常实现优雅关闭
- `close_echo_tasks`: 任务清理函数，给每个任务最多 2 秒时间完成，超时后捕获 TimeoutError 并忽略
- `main`: 主函数，允许端口重用，并注册关闭信号

现在通过 `telnet localhost 8002` 发送连接后，按 Ctrl-C 会等待两秒才关闭。

但这段代码并不适合生产环境：

首先是在等待 echo tasks 完成的时候，我们不会关闭 connection_listener。
即在关闭的时候，可能会有一个新的请求过来，这个新请求将无法被加入 2 秒的优雅关闭逻辑中。

另一个原因是，每个 echo task 的关闭逻辑只会捕获 TimeoutExceptions。
这意味着，如果我们的某个任务抛出了其他类型的异常，我们将捕获该异常，而其他后续任务中可能出现的任何异常都将被忽略。

## Summary

在本章，展示了如何使用 blocking 和 non-blocking sockets，并更加深入了 asyncio 事件循环。
并使用 asyncio 创建了一个高并发的 echo server。
并展示了如何在 tasks 中处理错误，并添加自定义的关闭逻辑。
