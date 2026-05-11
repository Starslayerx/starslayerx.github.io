+++
date = '2026-05-02T8:00:00+01:00'
draft = false
title = 'Learning concurrency - a deep dive into multithreading with Python'
categories = ['Blog']
tags = ['Python']
+++

这篇文章讲 Python 中的并发，例如多线程 _multithreading_, 多进程 _multiprocessing_, 竟态条件 _race conditions_ 以及同步状态 _synchronization_ 的机制，例如锁。
然后会探讨如何关闭 GIT 来实现 Python 中真正的多线程，并通过清晰的代码来突出差异、优点以及注意事项。

## Introduction

你可能会好奇为什么会需要并发。
大多数情况下，并不需要，但如果是以下情况可能需要：

- 数据处理 _data processing_ 和*[ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load)* - 解析大量文本文件、清理混乱的数据、或将复杂的正则表达式应用到数百万行上
- 加密 _cryptography_ 和哈希 _hashing_ - 读取文件并计算每一行的加密哈希 (例如 SHA-256)
- 数据科学 - 计算蒙特卡罗模拟 _Monte Carlo simulations_ 要求大量数学计算，例如 numpy, pandas 或 scikit-learn
- 网络操作 - 下载文件，爬取网站或调用 REST API

但首先，这里先把术语理清楚：

- **Sequential** 串行：单个进程一次只做一件事，在做下一件事情之前会等待直到完成后再开始下一个进程
- **Concurrency** 并发：单个进程同时做多件事，但不一定在同一时刻做
- **Event Loop** 事件循环: 一种控制接口，持续等待事件（例如 I/O 操作、定时触发或用户操作），分派对应的任务，然后重复这个过程
- **Parallelism/Multiprocessing** 并行/多进程: 在同一时刻做多件事
- **Multithreading** 多线程: 这种编程模型是单个进程生成多个独立的执行线程，从而达到并发。所有线程都共享内存空间和资源。

线程安全 _thread safety_ 是指程序或系统的一种特性，它允许多个线程同时访问和修改共享内存与资源，而不会造成数据损坏、内存泄漏或程序崩溃。

当多个线程操作同一个内存空间的时候，有可能他们会同一时刻读取并覆盖相同的值。
如果这种访问没有处理好，则会引发竞态条件 _race condition_。
也就是说最终的结果会无法预测，完全取决于每个线程执行任务的具体时间。

为了达成线程安全并防止竞态条件，开发者和编程语言依赖各种同步机制 _synchronization mechanisms_。
例如互斥量 _mutex_、锁 _lock_ 以及原子操作 _atomic operations_。

全局解释器锁 **The Global Interpreter Lock (GIL)** - 这是 Python 默认的设置，它会阻止多线程执行。

自由线程 **Free-threading** - 移除全局解释器锁 (GIL)，让 python 能支持多线程。

多核 CPU - 一个 CPU 核心是用户电脑中的一个独立处理单元，并会独立读取和执行指令。
现代 CPU 通常会有多个核心，并且允许并行。

## Example: Non-parallel Concurrency

现在让我们后退一步，找个更好的并发场景。
也就是函数代码应该被同时运行的情况，这里就是 `worker` 函数：

```python
import asyncio

async def worker(name, delay):
    print(f'{name} starting')
    await asyncio.sleep(delay)
    print(f'{name} finished after {delay} seconds')

async def main():
    task1 = asyncio.create_task(worker('task1', 1))
    task2 = asyncio.create_task(worker('task2', 2))
    task3 = asyncio.create_task(worker('task3', 3))

    await task1
    await task2
    await task3

asyncio.run(main())
```

`asyncio` 库会在单个线程上运行，它只会在单个 CPU 核心上运行。
它依赖的是这些任务涉及 I/O 操作（例如等待网络、数据库或者 `asyncio.sleep()` 这样定时器）。
这样之所以能够工作是因为 CPU 等待的时候什么都没有干，于是事件循环就能给它塞进另一个任务。
因此，在 `asyncio` 中，当遇到 I/O 阻塞操作时，代码会主动将控制权还给事件循环，让循环在后台高效运行其他任务，而不是闲置等待。

如果移除 `asyncio` 则每个任务会串行执行，这将会花费 `asyncio` 三倍的时间。

这里要指出的就是，GIL 会严重限制 CPU 密集代码，而对 I/O 密集的代码限制很少。
CPU 计算代码是受到处理器速度限制的代码，例如数学计算、图片处理或复杂数据处理。
由于 GIL 限制只有一个线程可以执行字节码，它完全阻止了多 CPU 核心多线程的并行计算。
如果尝试进行多线程 CPU 计算反而会损坏程序性能。
这是因为线程会不断争夺 GIL 的控制权，导致显著的上下文切换和资源争夺。

相反，I/O 密集任务耗费大量时间等待外部操作完成，例如从互联网下载数据、查询数据库或读取和写入文件到硬盘。
GIL 并不会阻止这些操作同时进行。
每当 python 线程发起一个阻塞 I/O 操作，它都会主动释放 GIL。
这让解释器能把锁交给另一个线程，这个线程可以主动执行 Python 代码，而第一个线程则在后台等着自己的数据送达。
由于协作式多任务处理 _cooperative multitasking_，多线程是加速 I/O 密集 Python 程序的有效策略。

## Free-Threaded Concurrency

自由线程并发是一种，移除全局解释器锁 GIL，多线程并发运行的方式，

默认情况下，GIL 会阻止多线程在多 CPU 多核心上运行，强制开发者使用多进程或编写 c 扩展。
移除 GIL 允许 Python 实现真正的多线程并行。

### The Global Interpreter Lock (GIL)

但首先，先讲讲 GIL 从何而来。

全局解释器锁是一个互斥锁 _mutex (mutual exclusion lock)_，它只允许一个 python 线程在单个处理器上执行字节码。
由于 GIL，Python 线程无法在多核 CPU 实现真正的并行；
实际上，他们是通过协作式 _cooperative_ 或抢占式 _preemptive_ 多任务处理，轮流使用处理器的。

GIL 起初是 1980 年代实现，它提供了一种简单的方法来确保 Python 内部内存管理（特别是引用计数）的线程安全。
也让整合哪些本身不是线程安全的 c 库变得容易的多。
由于不需要为每个数据结构都设置多个细粒度锁，GIL 也让单线程程序跑得飞快。

尽管全局计数器锁 GIL 早期有好处，但现在可以按需关掉它。
因为它已经称为了高性能计算、人工智能和机器学习的一大瓶颈。
这些主要的移除动机有：

- 无法重复利用现代硬件：现代计算机高度依赖多核处理器，但Python里的全局解释器锁却让那些需要大量计算的任务没法充分利用这些核心。当多个线程同时想要进行复杂运算时，它们就会争抢这个GIL，结果不仅导致频繁的上下文切换，使得开销很大，甚至可能比只用单个线程跑还要慢。

- 多进程的严重缺点：为了绕过 _work around_ GIL，开发者曾经依赖多进程并行，替换为系统进程而不是线程。
  然而，多线程处理会带来很大的代价；创建进程会消耗比线程多的多的内存，并且在进程中共享数据需要进行昂贵的数据序列化和进程间通信。

- 代码复杂度和 C++ 重新：因为没法放松地跑并行线程，开发者们只能被迫维护一些复杂又变通的方案。
  很多情况下，公司不得不把大量的 Python 代码转换成 C 或 C++ 代码，只为了提升性能。

根据 [PEP 703](https://peps.python.org/pep-0703/)，Python 从 3.13 版本开始正式推出了一种实现性的 “自由线程” 模式，开发者可以关掉全局计数器锁来运行 Python。
为了保证可以关掉解释器而不因此崩溃，Python 内部正在全面翻新 _overhauled_，加入新的线程安全机制。
这些机制包括一个叫 "mimalloc" 的新型内存分配器，把某些对象 “永久化” 让他们不再需要
引用计数，还用上了偏置引用计数 _biased and deffered reference counting_ 这类高级计数，防止线程之间互相锁死。

### Installing the free-threading Python (i.e Python without GIL)

```shell
# check that uv has been installed
uv --version

# install your free threaded Python in a virtual environment
uv venv --python 3.14t

# activate the virtual environment
source .venv/bin/activate
```

检查一下安装是否成功

```shell
python -VV
```

输出如下：

```text
Python 3.14.0rc1 free-threading build (main, Aug  6 2025, 22:54:03) [Clang 20.1.4 ]
```

然后检查 GIL 是否开启：

```shell
python -c 'import sys; print(sys._is_gil_enabled())'
```

可以使用 `-X` 参数切换回 GIL：

```shell
python -X gil=1 -c 'import sys; print(sys._is_gil_enabled())'
```

可以看看安装的 Python 有没有一个叫做 `Py_GIL_DISABLED` 的配置变量，有的话就说明能开关 GIL

```shell
python -c 'import sysconfig; print(sysconfig.get_config_var("Py_GIL_DISABLED"))'
```

系统级安装参考[这里](https://py-free-threading.github.io/installing-cpython/)

### Liability Waiver

在使用 GIL-free Python 多线程之前，需要留意一下几点：

- 竞态条件 (Race Conditions): 如果不仔细同步好各线程读写共享的顺序，最终结果就完全取决于线程那不可预测的时序。这些竞态条件会导致输出出错，而且特别难以复现和调试。

- 内存泄露和解释器崩溃 (Memory leask and interpreter crashed): Python 内部依赖引用计数管理内存。
  如果多线程通过非线程安全的方式，并发增加 _increment_ 和减少 _decrement_ 对象的引用计数，该计数还能会被损坏，导致内存泄露或程序突然崩溃。

- 死锁 (Deadlocks): 为了避免竞态条件，开发者使用锁 _locks_，然而，这会带来顺序死锁 _ordering deadlocks_ 的风险。当多个线程以不同顺序获取同一组锁时，就会发生这种情况，导致他们互相等待对付释放锁，从而永远卡死。

- 不安全的迭代器 (Unsafe iterators): 具体来说，在 Python 新的自由线程版本中，多个线程同时访问一个迭代器是不安全的，可能会导致程序重复处理或遗漏某些元素。

- C 扩展漏洞 (C-extension vulnerabilities): 对于使用 C 扩展的开发者来说，依赖“借来的引用” _borrowed references_ （也就是临时用一下列表或字典里的对象，但并没有真正拥有它）是非常危险的。在没有 GIL 保护的情况下，另一个线程可能会在读取某个对象时，偷偷把它从集和里删掉或改掉。另外，如果线程之间乱共享 C 语言的原始指针，很容易引发段错误 _segmentation faults_ 和数据损坏。需要小心在自由线程模式下运行的 C 扩展包括 NumPy、Pandas 和 Scikit-Learn。

- CPU 核心数量：下面示例是针对 8 核 CPU 的电脑设计的。要想准确测出脚本性能，记得修改 THREADS 为实际电脑核心数。

### Example: Embarrassingly Parallel Multithreading

Python 转向 GIL-free 架构的一个好处是，纯 Python 代码（不依赖 C 库）无需修改或重写，就能直接在自由线程上运行。
完全相同的多线程代码在两个环境中都能运行，唯一区别就在于使用哪个 Python 解释器，还有执行脚本的时候设置了哪些环境标志。

这是一个使用 `threading.Thread` 多线程 Python 示例，在四个线程上执行一个 CPU 密集任务的任务（计算斐波那契数列） ：

```python
import threading
import time

THREADS = 8

def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fin(n - 2)

def main():
    start_item = time.time()
    threads = []

    # Create and start 8 threads for CPU-bound task
    for i in range(TRHEADS):
        thread = threading.Thread(target=fib, args=(35,))
        thread.start()
        threads.append(thread)

    # Wait for all threads to finish
    for thread in threads:
        thread.join()

    print(f'Completed: {time.time() - start_time:.2f} seconds.')

if __name__ == '__main__':
    main()
```

下面运行：

```shell
$ uv run python fib.py
Completed: 0.81 seconds.
```

对比 GIL 版本：

```shell
$ uv run python -X gil=1 fib.py
Completed: 4.13 seconds.
```

由于 GIL 性能几乎慢了 4 倍。
这是因为 GIL 阻止了多线程同时执行 Python 字节码，所以线程之间会不停地相互打断、争抢锁。
该程序会串行执行，只能用到 8 和 CPU 大约 12.5% 的性能，变现像单线程程序一样。

话虽然这么说，但还是要提现一下线程安全的问题。
虽然语法没变，但现有代码的安全性可能会有变化。
这是因为很多人有个误解，以为全局解释器锁让所有 Python 代码天生就是线程安全的。
实际上，GIL 主要保护的是 Python 内部的内存管理和状态，并不能保证高层应用程序的操作是原子性的，也不能防止出现竞争条件。

现在来重新开一下斐波那契数列的代码示例。
递归的 `fib(n)` 函数纯粹是一个计算量很大的数学运算，用来测试 CPU 密集型的性能。
每个线程完全独立地执行这个函数，只依赖于自己独立的调用栈和局部输入，因此它是线程安全的。
因为线程不会修改任何共享数据或状态，所以他不需要任何同步 _synchronization_ 操作。

同步原语 _synchronization primitives_ 例如锁，是专门用来协调对共享资源的访问的。
防止多个线程同时写入一块内存空间时出现竞态条件。

因为上面这个例子里的那些线程既不交换数据，也不往共享的全局变量里写东西，更不合并各自算出来的部分结果，所以它们不会互相干扰。

当任务之间可以完全独立地进行，不需要交换任何数据时，它们通常被称为 “尴尬并行” _embarrassingly parallel_。
因为这种情况下没有需要保护的共享状态，所以加锁完全是多余的。

### Exmaple: Parallel Multithreading with Shared Resources - Buggy

现在来看一种事情可能出岔子 _go awry_ 的情况，使用简单的并行数字汇总器来查看：

```python
import threading
import time


COUNT = 1_000_000
THREADS = 8

counter = 0

def worker():
    global counter

    for _ in range(COUNT):
        counter += 1

threads = [
    threading.Thread(target=worker)
    for _ in range(THREADS)
]

start_time = time.time()
for t in threads:
    t.start()

for t in threads:
    t.join()

print('Time:', time.time() - start_time)
print('Counter:\t', counter)
print('Expected:\t', COUNT * THREADS)
```

分别运行

```shell
$ uv run python counter_buggy.py
Time: 0.4012320041656494
Counter:         1003668
Expected:        8000000

$ uv run python -X gil=1 counter_buggy.py
Time: 0.25642895698547363
Counter:         8000000
Expected:        8000000
```

发现多线程情况下是结果不符合预期，这是由于 `counter += 1` 不是一个原子操作。
它分为三步：

```
1. 读取 counter
2. 加 1
3. 将 counter 写回
```

交错 _interleaving_ 执行是导致 bug 的原因，多个线程都在对方写入的时候读取了值，因为没有同步机制。
这正式竞态条件的定义：结果取决于线程之间的“竞速”（时序）。

那么，GIL 能保护免受竞态条件的影响吗？

GIL 有时会掩盖这个问题，因为在 CPython 中线程并非真正并行运行，上下文切换也不那么频繁。
所以上述交错执行出现的频率较低，但**仍然可能发生**。

需要记住 `counter += 1` 的心智模型是：

```
Read → Compute → Write
```

如果两个线程同时执行这个操作，他们就可能读取到同一个旧值，导致其中一个更新被覆盖。
所以要牢记这条黄金法则：当多个线程访问共享的可变数据，并且至少有一个线程在写入时，就必须使用同步。

### Exmaple: Parallel Multithreading with Shared Resources - Simple Fix

对于一个多线程程序更新共享计数器的情况，需要使用像 `threading.Lock` 这样的同步原语，来保护应用程序的共享数据和逻辑，就像全局解释器锁存在时一样。
如果代码之前就需要锁或队列来保证现在安全，那么在自由线程版本中，他们仍然必不可少。
下面来看一个例子

```python
import threading
import time

COUNT = 1_000_000
THREADS = 8

counter = 0
lock = threading.Lock() # FIX A: create a lock object

def worker():
    global counter

    for _ in range(COUNT):
        # FIX B: lock this block so only 1 thread can run it
        with lock:
            counter += 1

threads = [
    threading.Thread(target=worker)
    for _ in range(THREADS)
]

start_time = time.time()
for t in threads:
    t.start()

for t in threads:
    t.join()

print("Time:", time.time() - start_time)
print("Counter:", counter)
print("Expected:", COUNT * THREADS)
```

现在再运行就发现 Counter 正确了：

```
$ uv run python -X gil=1 counter_fixed.py
Time: 0.5505349636077881
Counter: 8000000
Expected: 8000000

$ uv run python counter_fixed.py
Time: 0.8671579360961914
Counter: 8000000
Expected: 8000000
```

但是等等！
没有全局解释器锁的版本执行时间反而长了将近一倍？
这是怎么回事？说好的超快速度呢？这完全不合常理啊！

按理说，关掉GIL应该能让多线程代码跑得更快，这个想法很自然。
但是，这段代码恰恰暴露了并行编程中最大的一个悖论：
去掉GIL并不会神奇地把串行代码变成并行，反而可能让情况更糟

没有GIL的版本慢了将近三倍，原因主要出在两个方面：极端的锁竞争 *lock contention* 和 CPU 缓存抖动 *cache bouncing*。

1. 代码本质是顺序执行的

```python
for _ in range(COUNT):
    with lock:
        counter += 1
```
因为用了 `with lock:` 语句，一次只有一个线程能改计数器。
哪怕有 100 个 CPU 核心，其中 99 个也得停下来，排着队等着那个拿着锁的核心。
说到底，在这个任务里根本就没有并行运算。

2. 在 GIL 下的执行（非并行，但快）

当 GIL 开启时，Python 就像一个交通警察一样在中间指挥。
GIL 保证在任何时候只有一个线程能执行 Python 的字节码。
因为 GIL 已经阻止了多个线程同时在多个核心上运行，所以锁的竞争其实很小。
Python 内部的线程切换处理交接得还算顺畅。
线程们不会在操作系统层面激烈地争夺锁，因为 GIL 让它们保持了一定的秩序。

3. 没有 GIL 下的执行（并行，但是慢）

当你禁用全局解释器锁（GIL）后，Python 就像卸掉了辅助轮，直接把控制权交给了操作系统。
这时，操作系统把你的 8 个线程分配到 8 个独立的物理 CPU 核心上，然后大喊一声：“冲啊！”
所有 4 个核心在同一毫秒的瞬间都冲上去抢锁。因为锁的竞争非常激烈（总共要抢 800 万次），操作系统不得不频繁插手，让线程休眠又唤醒。
这种操作系统级别的上下文切换，跟 Python 内部自己管 GIL 比起来，代价大得多，也慢得多。

另一个原因是硬件层面发生的“缓存抖动”。
也就是说，CPU 核心花在互相同步和无效化对方内存缓存上的时间，比真正花在计算上的时间还要多。

### Example: Parallel Multithreading with Shared Resources - Proper Fix

禁用 GIL 并非提升性能的万灵药。
“无 GIL” 版本在无状态共享的 CPU 密集型任务中表现出色。
如果你的线程在对独立变量进行大量数学运算，无 GIL 版本会显著更快。
但如果你的线程不断争夺单个共享且被锁定的变量，无 GIL 版本则会遭遇严重的硬件级“交通堵塞”。
因此，让多线程代码飞速运行的秘诀，就是消除共享状态。

针对这个问题，我们来重写代码，使其真正利用多核，并在没有 GIL 的情况下大幅提速。
具体做法是：不再让四个线程争夺单个被锁定的计数器，而是为每个线程分配自己的本地计数器。
它们将完全独立地完成各自的工作，最后我们只需将它们的最终计数汇总即可。

下面使用 Python 的 `concurrent.futures` 模块来实现——这是处理该问题最简洁的方式：

```
import time
from concurrent.futures import ThreadPoolExecuter

COUNT = 1_000_000
THREADS = 8

def worker():
    local_counter = 0
    for _ in range(COUNT):
        local_counter += 1
    return local_counter

start_time = time.time()

with ThreadPoolExecuter(max_workers=THREADS) as executor:
    results = list(executor.map(
        lambda _: worker(), range(THREADS)
    ))

counter sum(results)

print('Time:', time.time() - start_time)
print("Counter:", counter)
print("Expected:", COUNT * THREADS)
```

它不仅正确，而且比那些有bug的代码快上大约五倍！

```
$ uv run python counter_fixed_fast.py
Time: 0.05626487731933594
Counter: 8000000
Expected: 8000000

$ uv run python -X gil=1 counter_fixed_fast.py
Time: 0.10117506980895996
Counter: 8000000
Expected: 8000000
```

首先，因为完全没有锁竞争。
没有用到 `threading.Lock()`，操作系统就不用为了等另一个线程而暂停当前线程。
比起有问题的老代码，这里也没有缓存跳跃的问题。
每个线程都在更新自己的局部变量 `local_counter` ，这些变量存在于各自独立的 CPU 缓存里。
CPU 核心不用浪费时间去和全局变量同步内存。
把这段代码放在“自由线程”的 Python 环境里运行，就能实现真正的并行，扩展性会非常好。
各个核心会同时全速跑各自的循环，执行时间也会大幅下降。

## The Golden Rule of Parallelism

每当需要通过多核心加速计算的时候，问自己：这些线程需要和对方沟通吗？
如果是，那么速度会很慢。
最好的并行代码会把一个大任务拆分成完全独立的小块，分别处理它们，最后在终点把结果合并起来。

大部分纯 Python 代码都不用重写或改动。
自由线程架构本来就是设计成跟现有代码高度兼容的。
以前在全局解释器锁下被认为是原子操作的那些代码，在自由线程版本里也依然是原子的。
所以，只要一个纯 Python 库本身已经是线程安全的（也就是该用 `threading.Lock` 之类标准同步工具的地方都用了），那就算没有 GIL，它也照样是线程安全的。

但那些依赖底层 C 库的 Python 代码或库呢？
难道不应该重写它们，让它们变成线程安全的，才能在禁用了 GIL 的 Python 上安全使用吗？

历史上，Python 的 C 语言接口让开发者可以直接控制全局解释器锁（GIL），很多原生扩展在 C 代码里都悄悄依赖 GIL 来保护全局数据结构和对象状态。
那些依赖原生 C 扩展的库，比如大多数数据科学、人工智能以及性能关键的库，像 NumPy、Pandas 和 scikit-learn 目前还没能完全做到线程安全。

如果某个库还没重写过会怎么样？
Python 自带了一个安全网机制。
当你导入一个第三方 C 语言 API 扩展包，而它没有明确标记支持自由线程时，Python 解释器会在运行时自动重新启用 GIL（全局解释器锁），并弹出一条警告信息。
这样能防止那些旧版包导致你的程序崩溃，不过代价就是暂时让你享受不到多核性能带来的好处了。

好消息是，这一转型已经在进行中。
像 Meta 和 Quansight 这样的组织正在积极为 Python 生态中最流行的软件包添加自由线程兼容性，而像 Python 自由线程指南这样的网站甚至已经建立起来，用来追踪常用库的兼容状态。
总之，无 GIL 的 Python 确实能成倍提升性能，但它并非万能药；
移除 GIL 并不能神奇地将串行代码变成并行代码，反而可能让情况更糟。
你得搞清楚如何设计代码才能充分利用它。

## Further Exploration

如果您希望充分发挥新一代自由线程系统的能力，Python 自由线程指南中提供了一些出色的示例，展示了在没有 GIL 的情况下所能实现的惊人效果。
其中有一个进行 [Web Scraping](https://py-free-threading.github.io/examples/asyncio/) 的例子，将多线程与 asyncio 结合在一起，而这两个概念最初看起来完全是正交的 *orthogonal*！

网页抓取是从网站中提取有用数据的过程，当需要处理成百上千个页面时，这一过程变得尤为具有挑战性且耗时。
传统的同步方式一次只能抓取一个页面，速度很慢。
借助 asyncio，我们可以利用异步 I/O 同时抓取多个页面，从而显著加快抓取速度；
然而，asyncio 只能使用单个 CPU 核心。
现代计算机通常配备多个 CPU 核心，但 asyncio 仅能利用其中一个核心。
而在自由线程 Python 中，我们可以通过线程运行多个 asyncio 工作程序，从而充分利用所有可用的核心。
