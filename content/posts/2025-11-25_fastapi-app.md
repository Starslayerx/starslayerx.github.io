+++
date = '2025-11-25T8:00:00+08:00'
draft = true
title = 'FastAPI app and request'
tags = ['FastAPI']
+++

在 FastAPI 中，`Request` 对象和 `FastAPI` 应用实例 (`app`) 是核心概念，它们在应用状态管理和依赖注入中扮演着关键角色。
本文将介绍它们的关系、设计理念，以及如何利用 `app.state` 实现单例模式。

---

## FastAPI 对象

`FastAPI` 对象是整个应用的核心实例：

```python
from fastapi import FastAPI

app = FastAPI(title="示例应用")
```

### 核心职责

- **路由管理**：通过 `@app.get()`、`@app.post()` 等装饰器定义 URL 到视图函数的映射。
- **中间件和事件管理**：可注册中间件处理请求/响应，支持 `startup` 与 `shutdown` 事件。
- **应用状态管理**：提供 `app.state`，可存放全局单例对象、数据库连接池、配置等。
- **异常处理与依赖注入**：管理异常处理器，并协助依赖注入机制。

### 单例模式存储

这里要使用 app 的 State 对象存储单例，app 中定义如下

```python
# app
self.state: Annotated[
    State,
    Doc(
        """
        A state object for the application. This is the same object for the
        entire application, it doesn't change from request to request.

        You normally wouldn't use this in FastAPI, for most of the cases you
        would instead use FastAPI dependencies.

        This is simply inherited from Starlette.

        Read more about it in the
        [Starlette docs for Applications](https://www.starlette.dev/applications/#storing-state-on-the-app-instance).
        """
    ),
] = State()

```

State 源码如下，简单看就是一个字典

```Python
# State
class State:
    """
    An object that can be used to store arbitrary state.

    Used for `request.state` and `app.state`.
    """

    _state: dict[str, Any]

    def __init__(self, state: dict[str, Any] | None = None):
        if state is None:
            state = {}
        super().__setattr__("_state", state)

    def __setattr__(self, key: Any, value: Any) -> None:
        self._state[key] = value

    def __getattr__(self, key: Any) -> Any:
        try:
            return self._state[key]
        except KeyError:
            message = "'{}' object has no attribute '{}'"
            raise AttributeError(message.format(self.__class__.__name__, key))

    def __delattr__(self, key: Any) -> None:
        del self._state[key]

```

使用示例

```Python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """
    FastAPI 应用生命周期管理

    启动时初始化资源,关闭时清理资源
    """
    # 初始化单例
    print("[应用启动] 创建 Admin Agent 服务实例...")
    admin_agent_service = AdminService(mcp_server_configuration)
    app.state.admin_agent_service = admin_agent_service
    await admin_agent_service.initialize()  # 立即初始化 Agent

    yield  # 应用运行其间

    # 关闭数据库连接池
    await close_database()

    # 关闭 Redis 连接
    await close_redis()
```

当需要获取该单例的时候，通过依赖注入得到

```Python
def get_admin_service(request: Request) -> AdminService:
    """获取 Admin Agent 服务实例 (app.state)"""
    return request.app.state.admin_agent_service

@app.post()
async def send_admin_message(request: Request) -> StreamingResponse:
    ...

    # 获取 Admin Agent 服务实例
    admin_service = get_admin_service(request)

    ...

    # SSE
    return StreamingResponse(
        media_type="application/x-ndjson",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        }
    )
```

- **作用**：确保整个应用中只有一个 `AdminService` 实例被创建和共享。
- **优势**：
  - 避免重复初始化资源
  - 统一管理全局服务
  - 可在请求中安全访问

## Request 对象

`Request` 对象封装了一次 HTTP 请求的上下文：

```python
def get_admin_service(request: Request) -> AdminService:
    """获取 Admin Agent 服务实例 (app.state)"""
    return request.app.state.admin_agent_service
```

### 核心职责

- **封装请求信息**：包含方法、路径、headers、query 参数、body 等。
- **提供应用访问入口**：`request.app` 指向该请求所属的 `FastAPI` 实例。
- **支持依赖注入**：可以作为依赖函数参数，让函数获取应用状态或服务实例。

`Request` 与 `app` 的关联

1. **解耦请求与全局变量**

   请求处理函数无需直接引用全局 `app`，通过 `request.app` 即可访问应用实例和状态。

2. **支持多实例应用**

   同一个 Python 进程可以启动多个 FastAPI 实例，每个实例的 `state` 独立。请求绑定到具体实例，`request.app` 指向对应实例。

3. **依赖注入友好**

   在依赖函数中，通过 `Request` 获取 `app.state` 中的单例对象：

```python
from fastapi import Depends

def get_admin_service(request: Request) -> AdminService:
    return request.app.state.admin_service

@app.get("/admin")
async def admin_endpoint(service: AdminService = Depends(get_admin_service)):
    return {"message": service.hello()}
```

- **工作流程**：
  1. FastAPI 收到请求，创建 `Request` 对象。
  2. DI 系统调用 `get_admin_service`，将 `Request` 注入。
  3. 函数通过 `request.app.state.admin_service` 获取全局单例。
  4. 依赖对象传入 endpoint。

## 总结

- `app`：FastAPI 实例，负责路由、事件、中间件、依赖和状态管理。
- `app.state`：存储单例对象和全局资源，实现单例模式。
- `Request`：每次请求的封装，携带请求信息和对 `app` 的引用。
- `request.app.state`：结合依赖注入，让每个请求安全访问全局单例对象，无需使用全局变量。

这种设计模式既保证了应用单例对象的统一管理，又支持依赖注入和多实例应用，非常适合现代 Web 应用开发。
