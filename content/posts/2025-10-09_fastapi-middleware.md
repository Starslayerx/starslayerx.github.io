+++
date = '2025-10-09T8:00:00+08:00'
draft = false
title = 'Fastapi Middleware'
tags = ['Fastapi']
+++

## Middleware

你可以添加中间件到 FastAPI 应用中。

“中间件” 是一个函数，它在每个请求被特定路径操作之前对其进行处理，同时在每个响应返回之前也对其进行处理。

- 在到达应用程序之前处理请求
- 可以在请求中做一些事情，或运行任何需要的代码
- 将处理后的请求传递给应用程序
- 之后处理应用程序返回的响应
- 可以对响应做一些事情，或运行任何需要的代码
- 然后返回响应

### Create a Middleware 创建一个中间件

想要创建一个中间件，你可以在函数上面使用装饰器 `@app.middleware("http")`，该函数接受：

- `request` 请求
- 一个函数 `call_next` 并将会接收 `request` 作为一个参数
  - 该函数会将 `request` 传递给对应的路径操作
  - 然后返回对应路由操作生成的 `response`
- 你可以修改或者直接返回 `response`

```Python
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    ...
```

> TIP

自定义专属 headers 可以使用 [X-prefix](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers) 来添加。

但如果你有一个自定义的 header 并想要客户端能够看到这些信息，你需要使用 [Starlette's CORS docs](https://www.starlette.dev/middleware/#corsmiddleware) 中的参数参数 `expose_headers` 将其加入你的 CORS 设置里 ([CORS (Corss-Origin Resource Sharing)](https://fastapi.tiangolo.com/tutorial/cors/))。

#### Before and after the `response` 在响应前后

你也可以在 `request` 前后运行代码，也可以在 `response` 前后运行代码。

例如，你可以添加一个自定义标头 `X-Process-Time` 包含处理请求和返回响应的时间。

```Python
import time
from fastapi import Request, FastAPI

app = FastAPI

async def add_process_time_header(request: Request, call_next):
    start_time = time.perf_counter()
    response = await call_next(request)
    process_time = time.perf_counter() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

这里使用 [time.perf_counter()](https://docs.python.org/3/library/time.html#time.perf_counter) 而不是 `time.time()` 因为它可以完成更加精细的控制。

### Multiple middleware execution order 多中间件执行顺序

无论当你使用 `@app.middleware()` 或 `@app.middleware()` 装饰器方法的时候，每个中间件都会将应用包装起来，形成一个栈。
最后添加的中间件是 _outermost_ 最外层，第一个添加的是 _innermost_ 最内层。

在请求路径上，最外层的中间件首先运行，在响应路径上，最后运行。

例如：

```Python
app.add_middleware(MiddlewareA)
app.add_middleware(MiddlewareB)
```

这会导致下面的执行顺序：

- **Request**: MiddlewareB -> MiddlewareA -> route
- **Response**: route -> MiddlewareA -> MiddlewareB

这种栈的行为保证了中间件执行是一个可预测与可控制的顺序。

### Other middlewares 其他中间件

在 [Advanced User Guide: Advanced Middleware](https://fastapi.tiangolo.com/advanced/middleware/) 中有其他中间件的描述。
