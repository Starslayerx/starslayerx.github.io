+++
date = '2025-08-10T10:00:00+08:00'
draft = true
title = 'Python Asyncio Introduce'
+++

### Fast Introduce
下面代码中, 定义了一个 `main()` 协程函数, 然后通过 `asyncio.run()` 来执行它
```Python
import asyncio
import time

async def main():
    print(f"{time.ctime()} Hello!")
    await asyncio.sleep(1.0)
    print(f"{time.ctime()} Goodbye!")

asyncio.run(main())
```
在实践中, 大多数基于 asyncio 的代码都会使用 `run()` 函数, 但是要了解实际发生了什么, 参考下面这段代码 (实现了同样的功能)
```Python
import asyncio
import time

async def main():
    print(f"{time.ctime()} Hello!")
    await asyncio.sleep(1.0)
    print(f"{time.ctime()} Goodbye!")

loop = asyncio.get_event_loop() # 1
task = loop.create_task(main()) # 2
loop.run_until_complete(task)   # 3
pending = asyncio.all_tasks(loop=loop) # 4
for task in pending: # 5
    task.cancel()
group = asyncio.gather(*pending, return_exceptions=True) # 6

loop.run_until_complete(group) # 7
loop.close() # 8
```
代码解释:
1. 获取事件循环: 使用 `get_event_loop()` 创建一个**事件循环实例**(如果当前线程没有 loop 就会创建一个), 一个线程中, 多次调用 `get_running_loop()` 会返回同一个 loop 对象  
    - 如果是在异步的 async def 函数中, 则应该调用 `get_running_loop()`

2. 把协程调度到循环中: 使用 `loop.create_task()` 将协程对象 `main()` 封装成 Task 对象, 并注册到事件循环中. Task 会让协程自动开始执行, 而不必手动 `await`  
    - 返回的 `task` 可以用来: 查询任务状态(如 `.done()`、`.result()`), 取消任务(`task.cancel()`)

3. 启动事件循环并阻塞主线程: 阻塞当前进程, 直到 `task` 执行完成

4. 找出并取消剩余任务: 使用 `asyncio.all_tasks()` 获取这个 loop 里所有未完成的任务, 一般在程序收尾时做一次清理, 防止有任务卡住 loop

5. 给这些任务发取消信号

6. 收集所有任务并处理异常: `asyncio.gather()` 会返回一个**聚合任务**, 把所有 `pending` 任务捆绑成一个任务对象. `return_exceptions=True` 表示即使某个任务抛异常，也不会直接中断，会把异常作为结果返回

7. 再次运行事件循环, 直到所有 pending 任务完成(确保程序退出前 loop 是干净的)

8. 关闭事件循环
