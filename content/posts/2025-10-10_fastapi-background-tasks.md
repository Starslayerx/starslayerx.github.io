+++
date = '2025-10-10T8:00:00+08:00'
draft = false
title = 'Fastapi Background Tasks'
categories = ['Blog']
tags = ['Fastapi']
+++

## Background Tasks 后台任务

你可以定义一个在返回响应之后运行的后台任务。
这对请求之后执行一些操作十分有用，客户端无需一直等待操作任务完成再接收响应。

这包含一些例子：

- 执行操作后发送电子邮件
  - 由于连接邮件服务器并发送邮件一般会比较“慢”（几秒钟），你可以立刻返回响应并在后台发送邮件请求。

- 处理数据
  - 例如，你收到了一个文件需要缓慢处理，你可以返回一个 "Accepted" 响应 (HTTP 202) 并在后台处理文件。

### Using `Background Tasks` 使用后台任务

首先要导入 `BackgroundTasks` 并在执行函数中定义一个路径参数，使用 `BackgroundTasks` 类型声明。

```Python
from fastapi import BackgroundTasks, FastAPI

app = FastAPI()

def write_notification(email: str, message=""):
    with open("log.txt", mode="w") as email_file:
        content = f"notification for {email}: {message}"
        email_file.write(content)

@app.post("/send-notification/{email}")
async def send_notification(email: str, backgroud_tasks: Background(Tasks): # Add parameter here
    background_tasks.add_task(write_notification, email, message="some notification") # add backgroud task here
    return {"message": "Notification sent in the background"}
```

### Create a task function 创建任务函数

创建一个函数放到后台运行，只是一个接收参数的基本函数，可以是 `async def` 或者就普通的 `def` 函数，FastAPI 会正确的处理它。

在这个例子中的任务函数将会编写文件，并且写入操作不使用 `async` 或 `await` 故使用 `def` 定义了一个基本的函数。

```Python
# 任务函数
def write_notification(email: str, message=""):
    with open("log.txt", mode="w") as email_file:
        content = f'notification for {email}: {message}'
        email_file.write(content)
```

### Add the background task 添加后台任务

在路径操作函数内部，使用 `BackgroundTasks` 对象方法 `.add_task()` 将任务函数添加到后台任务中。

```Python
@app.post("/send-notification/{email}")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(write_notification, email, message="some notification")
    return {"message": "Notification sent in the background"}
```

`.add_task()` 接受下面参数：

- 一个后台运行的任务函数 (`write_notification`)
- 一系列参数，并将按顺序传入任务函数中 (`email`)
- 任何通过关键字参数传入任务函数中 (`message=some notification`)

### Dependency Injection 依赖注入

使用 `BackgroundTasks` 也同样适用于依赖注入系统，你可以在多层级声明一系列的 `BackgroundTasks`：如路径操作函数，依赖，子依赖等。

FastAPI 知道每种情况下应该怎么做，以及如何重用一个对象，以便所有的后台对象被合并到一起，并之后在后台运行。

```Python
from typing import Annotated
from fastapi import BackgroundTasks, Depends, FastAPI

app = FastAPI()

# 日志写入函数
def write_log(message: str):
    with open("log.txt", mode="a") as log:
        log.write(message)

# 依赖函数
def get_query(background_tasks: BackgroundTasks, q: str | None = None):
    if q:
        message = f"found query: {q}\n"
        background_tasks.add_task(write_log, message)
    return q

@app.post("/send-notification/{email}")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks,
    q: Annotated[str, Depends(get_query)]
):
    message = f"message to {email}\n"
    background_tasks.add_task(write_log, message)
    return {"message": "Message sent"}
```

上面代码中 `get_query()` 的参数 `q` 为一个 query 查询参数，从 url 中 `?q=xxx` 获取，然后再通过后台任务将其写入日志。

### Technical Details 技术细节

类型 `BackgroundTasks` 直接源于 [starlette.background](https://www.starlette.dev/background/)。
该类可以直接从 FastAPI 导入，这样避免了从 `starlette.background` 导入 `BackgroundTask` (不含s)。

### Caveat 警告

如果需要大量的后台运算，且并不一定需要让他们在相同的进程中运行（例如无需共享内存），那么可以考虑使用 [Celery](https://docs.celeryq.dev/) 这样更大的工具。

这往往需要更加复杂的配置，例如一个任务队列管理器，想 RabbitMQ 或 Redis，但他们允许你在多个进程甚至多个服务器中运行后台任务。
