+++
date = '2025-11-19T8:00:00+08:00'
draft = false
title = 'Python Asyncio 01: Getting to know asyncio'
tags = ['Python', 'Asyncio']
+++

## Python asyncio 基础篇

本篇包含

- asyncio 是什么以及如何使用它
- concurrency 并发、parallelism 并行、threads 线程和 processes 进程
- GIL (global interpreter lock) 全局解释器锁和其带来的并发跳转
- 非阻塞 sockets 如何只通过一个线程实现并发
- 基于事件循环 (event-loop-based) 并发的基本原理

asynchronous programming 异步编程意思是可以在主程序之外，额外运行一个特定的长时运行的任务。

一个 coroutine 协程是一种方法，协程是一种方法，当遇到可能长时间运行的任务时，它可以暂停执行，并在任务完成后恢复执行。

asyncio 这个库的名称可能让人人为其只适合编写 I/O 操作，但实际上该库可以和 multithreading 和 multiprocessing 库结合使用。
基于这种 interoperability 互操作性，可以使用 async/await 关键字让工作流更加容易理解。
这意味着，asyncio 不仅适合 I/O 的并发，也可以在 CPU 密集操作中使用。

所谓的 I/O-bound 和 CPU-bound 是指限制程序运行更快的主要因素，这意味着如果增加该方面的性能，程序就能够在更短的时间内完成。

下面是一些例子

- I/O 密集操作：网络请求、文件读取
- CPU 密集操作：循环遍历文件夹、计算 pi

```Python
import requests

response = requests.get('https://www.example.com')  # 1
items = response.headers.items()
headers = [f'{key}: {headers}' for key, header in items]  # 2
formatted_headers = '\n'.join(headers)  # 3
with open('headers.txt', 'w') as file:  # 4
    file.write(formatted_headers)
```

1. I/O-bound 网络请求
2. CPU-bound 响应处理
3. CPU-bound 字符串拼接
4. I/O-bound 写入磁盘

Concurrency 并发 和 Parallelism 并行的区别这里就不多说了。

Multitasking 分为 preemptive multitasking 抢占式多任务处理 和 cooperative multitasking 协作式多任务处理。

- Preemptive Multitasking

  在抢占式多任务处理中，通过时间片轮转 (time slicing) 进程，让操作系统来决定执行哪些任务切换。
  当操作系统在不同任务之间切换的时候，我们称其为抢占 (preempting)。
  该机制底层如何工作取决于操作系统，这主要通过多进程或多线程实现

- Coorperative Multitasking

  在协作式多任务处理中，不让操作系统决定何时切换任务，而是通过明确的编码来指明何时可以切换到其他任务上。
  应用任务运行一个协作模型，显示地说“我要暂停这个任务一会儿，执行别的任务去吧”。

  协作式的优势：首先协作式资源消耗更低，当操作系统需要在进程或线程间切换的时候，会有一个上下文切换的过程。
  第二个是粒度，操作系统切换任务的时间点可能并非最优的，当并发处理的时候就有了更多控制权，可以在正确的时间切换任务。

### Process 进程

一个进程有独立的内存空间，且其他应用不能访问。
一台机器可以运行多个进程，如果 CPU 是多核的，那么可以同时运行多个进程。

### Thread 线程

线程是操作系统的最小管理单元，可以看作是一个轻量的进程。
它们没有进程那样的独立内存空间，相反，线程共享进程创建的内存空间。

线程是和创建他们的进程相关联的，进程至少会有一个线程，一般称为主线程 main thread。
进程还可以创建其他线程，一般称为 worker 或 background threads。
这些线程可以和主线程并发运行，操作系统也可以在时间片(time slicing)轮转时切换他们。

当启动一个 Python 应用的时候，我们会创建一个进程和一个主线程来负责运行 Python 应用。

```Python
import os
import threading

print(f"Python process running with process id: {os.getpid()}")
total_threads = threading.active_count()
thread_name = threading.current_thread().name

print(f"Python is currently running: {total_threads} thread(s).")
print(f"The current thread is {thread_name}")
```

输出为

```text
Python process running with process id: 62973
Python is currently running: 1 thread(s).
The current thread is MainThread
```

- 多线程 Python 应用示例

```Python
import threading


def hello_from_thread():
    """Print current thread name."""
    print(f"Hello from thread {threading.current_thread()}!")

# 显示创建的 thread (not main thread)
hello_thread = threading.Thread(target=hello_from_thread)
hello_thread.start()  # start thread

total_threads = threading.active_count()
thread_name = threading.current_thread().name

print(f"Python is currently running {total_threads} thread(s)")
print(f"The current thread is {thread_name}")

hello_thread.join()  # pause untill started and completed
```

输出为

```text
Hello from thread <Thread(Thread-1 (hello_from_thread), started 6111047680)>!
Python is currently running 2 thread(s)
The current thread is MainThread
```

要注意，运行上面代码可能会看到线程中输出 hello 的内容，并且看到 "Python is currently runnig 2 thread(s)" 在统一行输出两次。
这里一种 race condition 竟态条件，多线程是许多编程语言的实现并发的一种方式，但由于 GIL 的限制，python 的多线程只对 I/O-bound 类型有效。

此外，还可以使用多进程，即 multiprocessing，一个父进程会创建子进程并管理这些进程，然后将工作内存分发给这些子进程。
multiprocessing 的 API 类似 multithreading，首先创建一个 target function，然后调用 start 方法执行，最后 join 等待运行完成。

```Python
import os
import multiprocessing


def hello_from_process():
    print(f"Hello from child process {os.getpid()}!")

if __name__ == "__main__":
    hello_process = multiprocessing.Process(target=hello_from_process)
    hello_process.start()

    print(f"Hello from parent process {os.getpid()}")
    hello_process.join()
```

输出为

```text
Hello from parent process 35718
Hello from child process 35720!
```

多进程尤其适合 CPU 密集任务。

### Global interpreter lock

全局解释器锁会使得 python 进程只能同时有一个线程运行 python 代码。

全局解释器存在的原因是由于 CPython 的内存管理方式，在 CPython 中内存主要通过引用计数 (reference counting) 的方式管理。
引用计数的工作原理是跟踪当前哪些程序需要访问特定的 Python 对象，如整数、字典或列表。

这里的冲突是 CPython 多线程不是线程安全的，如果有多余两个线程同时修改同一个变量，则该变量的状态将变得未知。
当两个线程访问同一个 Python 对象的时候会产生竟态条件，最终可能导致应用崩溃。

这是否意味着在 Python 中无法利用多线程的性能优势了呢？
实际上，对于 I/O 任务仍然可以通过多线程加速（在一些特殊情况下有一些 cpu 密集的例外可以多线程并发），来看下面的示例

```Python
import time
import requests

def read_example() -> None:
    response = resuests.get("https://www.example.com")
    print(response.status_code)

sync_start = time.time()
read_example()
read_example()
sync_end = time.time()

print(f"Running synchronously took {sync_end - sync_start:.4f} seconds.")
```

应该会看到下面这样输出

```text
200
200
Running synchronously took 0.2066 seconds.
```

下面再编写一个多线程版本对比一下

```Python
import time
import requests
import threading

def read_example() -> None:
    response = requests.get("https://www.example.com")
    print(response.status_code)

thread_1 = threading.Thread(target=read_example)
thread_2 = threading.Thread(target=read_example)

thread_start = time.time()
thread_1.start()
thread_2.start()
print("All threads running!")
thread_end = time.time()

thread_1.join()
thread_2.join()

thread_end = time.time()
print(f"Running with threads took {thread_end - thread_start:.4f} seconds.")
```

输出为

```text
All threads running!
200
200
Running with threads took 0.0917 seconds.
```

这几乎有两倍的差距！
答案藏在系统后台调用中，在这种情况下 I/O 会释放锁，因为网络请求是在操作系统层处理的。
这时，只有当收到数据返回为 Python 对象的时候才会重新获取锁。
换句话说，在 Java 或 C++ 这样语言中，这种情况下会并行执行，然而 Python 由于 GIL 的限制只能进行 I/O 类型的并发，同一时刻只能有一份 Python 代码在执行。

### asyncio and the GIL

asyncio 利用 I/O 操作会释放 GIL 的特性，使得即使只有单个线程也能够实现并发。
当使用 asyncio 的时候会创建叫 coroutines 协程的对象，一个协程可以被看成是一个轻量级线程。

这意味着 asyncio 并非绕过了 GIl，而是仍然受其限制。
如果我们有一个 cpu-bound 任务，我们仍然需要多进程来并发执行。

### How single-threaded concurrency works

之所以可以实现单线程并发是因为在系统层面，I/O 操作可以被并发完成。
为了更好理解，先需要弄懂 sockets 如何工作，有其实非阻塞套接字 non-blocking sockets。

socket 本质上网络发送和接收数据的是底层抽象，这是数据在服务器之前传输的基础。
sockets 支持两个简本的操作：发送字节和接收字节。
可以将 sockets 类比为邮箱，你可以向邮箱中放入信封，收件人打开邮箱和你的邮件，根据右键里面内容，收件人可能会给你回信。

sockets 默认是阻塞的，简单来说，这意味着当我们在等待服务器回复数据的时候，会暂停当前应用或阻塞直到收到数据。
但在操作系统层面，sockets 可以在非阻塞模式下运行，并可以执行即发即弃的读写操作，程序能够继续执行其他任务。
然后，会在操作系统层收到并处理字节。不是阻塞和等待数据返回，而是通过更加响应式的方式，让操作系统数据准备好后通知我们。

在后台，这时通过不同的通知系统实现的，根据操作系统会有不同：

- kqueue - FreeBSD and MacOS
- epool - Linux
- IOCP (I/O complection port) - Windows

这些通知系统是 asyncio 实现的基础，Python 中一个最基础的事件循环 event loop 类似下面这样：

```Python
from collections import deque

messages = deque()
while True:
    if messages:
        message = message.pop()
        process_message(message)
```

在 asyncio 中，事件循环里面报错的是 tasks 任务，而非消息。
当创建事件循环的时候，会创建一个空的任务队列。
事件循环的每一次迭代都会检查需要执行的任务，并逐个运行它们，直到某个任务遇到 I/O 操作。
这时候任务会被暂停，并且我们指示操作系统监控套接字，等待 I/O 完成，然后去运行下一个任务。
在每一次事件循环中，检查是否有任何 I/O 完成了。
