+++
date = '2025-10-24T8:00:00+08:00'
draft = false
title = 'Fastapi Lifespan Events'
tags = ['Fastapi']
+++

## Lifespan Events 生命周期事件

通过生命周期事件可以定义在应用开启之前需要执行的代码，这意味着这些代码会在开始接收外部请求之前被执行一次。
同样地，也可以定义应用在关闭的时候定义需要执行的代码，在尽力处理完所有请求后，该代码会被执行一次。

这对于设置需要在整个 app 的请求间共享的资源时非常有用，或者是需要进行清理工作的时候。
例如，一个数据库连接池，或者加载一个共享的机器学习模型。

### Use Case 使用示例

下面通过一个例子说明如何使用。

假如你有一个机器学习模型，并且需要让其处理请求，由于请求都共享同一个模型，因此不是一个请求对应一个模型，或一个用户一个模型。
假设模型加载需要一定的时间，因为要从磁盘中读取大量的数据，因此不能每个请求都去加载一次。
你可以在顶层的模块文件中定义加载，但这意味着当进行简单的自动化测试的时候，也会加载该模型，这样就会很慢。

这就是需要解决的问题，需要在请求响应之前加载模型，也不是在代码被加载的时候加载模型。

### Lifespan 生命周期

可以通过在 `FastAPI` app 中使用 `lifespan` 参数来定义启动和关闭逻辑，以及一个 "context manager" (上下文管理器)。

通过下面这种方法创建一个含 `yield` 的 `function`

```Python
from contextlib import asynccontextmanager

from fastapi impor FastAPI

def fake_answer_to_everything_ml_model(x: float):
    return x * 42

ml_models = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Load the ML model
    ml_models["answer_to_everything"] = fake_answer_to_everything_ml_model
    yield
    # Clean up the ML models and release the resources
    ml_models.clear()

app = FastAPI(lifespan=lifespan)

@app.get("/predict")
async def predict(x: float):
    result = ml_models["answer_to_everything"](x)
    return {"result": result}
```

这里在生成器 `yield` 之前将模拟的昂贵函数放入机器学习字典中。
这段代码将在应用程序接收请求之前执行，即启动阶段。

然后，在 `yield` 后面，卸载模型。
这段改名将在完成请求之后执行，即关闭之前，这样会释放内存和 CPU 资源。

#### Lifespan function 生命周期函数

第一件注意到的事是，定义了一个带 `yield` 的 async function，这与带 `yield` 的 Dependencies 相同。

```Python
async def lifespan(app: FastAPI):
    ...
    yield
    ...
```

在 `yield` 之前的部分会在应用开启之前执行，`yield` 之后的部分会在应用结束之后执行。

#### Async Context Manager 异步上下文管理器

该函数使用 `@asynccontextmanager` 异步上下文管理器装饰，将函数转化成一个 "**async context manager**"。

```
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    ...
```

在 Python 中的 **context manager** 上下文管理器可以在 `with` 语法中使用，例如 `open()` 可以作为上下文管理器使用：

```Python
with open("file.txt") as file:
    file.read()
```

在最近的 Python 版本中，也有一个 **async context manager** 异步上下文管理器，可以通过 `async with` 使用：

```Python
async with lifespan(app):
    await do_stuff()
```

当使用上面那样创建一个（异步）上下文管理器时，实际发生的事是。
在进入 `with` 块之前，会去执行 `yield` 之前的代码，退出 `with` 块之后，再去执行 `yield` 之后的代码。

在之前的例子中，并没有直接这样写，而是将其传递给 FastAPI 使用。

`FastAPI` app 的 `lifespan` 参数接受一个 **async context manager**，因此可以直接将 `lifespan` 异步上下文管理器传递给它。

```Python

@asynccontextmanager
async def lifespan(app: FastAPI):
    ...
    yield
    ...

app = FataAPI(lifespan=lifespan)
```

### Alternative Events (deprecated)

旧的语法这里就不再详细介绍了，大概下面这样使用

```Python
@app.on_event("startup")
async def startup():
    print("启动xxx")

@app.on_event("shutdown")
async def shotdown():
    print("关闭xxx")
```

### Technical Details 技术细节

在 ASGI 协议规范下，这是 [Lifespan Protocol](https://asgi.readthedocs.io/en/latest/specs/lifespan.html) 的一部分，并且定义了 `startup` 和 `shutdown` 的事件。

要记住，这种 lifespan events 将只会在 mian application 执行，而不会在 [Sub Applications - Mounts](https://fastapi.tiangolo.com/advanced/sub-applications/) 中执行。
