+++
date = '2025-08-03T8:30:00+08:00'
draft = false
title = 'How FastAPI Works'
categories = ['Note']
tags = ["FastAPI"]
+++

FastAPI 的工作原理: 从 routing 到 lifecycle 以及在现实中的使用

### FastAPI

FastAPI 是一个现代的 Python Web 框架, 注重**高性能**和**开发效率**. 旨在帮助开发者编写结构清晰、可靠的API, 同时尽量减少样板代码 (boilerplate)

其由以下两个库驱动:

- **Starlette**: 负责 Web 服务器逻辑、路由、中间件和异步能力
- **Pydantic**: 基于 Python 类型提示, 处理数据验证、解析和序列化

此外, Fastapi 还有输入验证、基于 Swagger UI 的自动文档生成和代码清晰化的基础

### API 请求周期

Fastapi 的请求生命周期如下

```
客户端请求 (Client Request)
    ↓
FastAPI App
    ↓
中间件（Middleware）
    ↓
路由匹配 (Route Matching)
    ↓
依赖注入（Dependency Injection）
    ↓
输入验证 (Input Validation)
    ↓
端点函数 (Endpoint)
    ↓
响应序列化 (Response Serialization)
    ↓
客户端响应 (Client Response)
```

1. 请求首先进入 FastAPI 应用 (本质就是一个 Starlette 应用)
2. 所有中间件优先执行 (如: 日志、错误处理、CORS等)
3. 路由器检查路径和方法, 找到对应的处理函数
4. FastAPI 使用`Depends`解析依赖
5. 使用 Pydantic 自动解析并验证输入数据
6. 执行端点函数, 参数验证完毕
7. 返回结果被序列化为合适的响应格式 (JSON)
8. 响应返回给客户端

### 路由 Router

1. 在应用对象上定义  
   适合小项目或原型验证

   ```python
   from fastapi import FastAPI

   app = FastAPI()

   @app.get("/items/{item_id}")
   def read_item():
       return {"item_id": item_id}
   ```

2. 使用 APIRouter 模块化  
   适合大项目

   ```python
   from fastapi import FastAPI

   router = APIRouter(prefix="/users", tags=["users"])

   @router.get("/{user_id}")
   def get_user(user_id: int):
       return {"user_id": user_id}
   ```

   使用`APIRouter`可以将相关的端点分组, 添加前缀和标签, 保持代码结构清晰模块化

当某个请求与端点匹配时, FastAPI 内部执行一下步骤:

1. Starlette 找到对应路由, 并创建一个`APIRouter`实例
2. FastAPI 使用`get_router_header()`包装端点函数并解析依赖
3. 使用 Pydantic 或基本类型对请求数据解析与验证
4. 装饰函数被调用, 传入验证后的参数
5. 返回值被序列化为响应对象

### 依赖注入: 干净、可复用的逻辑

FastAPI 有一个轻量且强大的依赖注入系统, 可以进行数据库链接、身份验证信息或配置信息等

```python
from fastapi import Depends

def get_db():
    db = create_db_session()
    try:
        yield db
    finally:
        db.close()

@app.get("/items/")
def read_items(db=Depends(get_db)):
    return db.query(item).all()
```

使用`Depends`, FastAPI 会负责调用`get_db`, 处理生成器生命周期, 并将结果注入到函数中

### 原生支持异步 (Async)

不同于一些后加入 async 的框架, FastAPI 一开始就设计为支持 async/await

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

@app.get("/hi")
async def greet():
    await asyncio.sleep(1)
    return "Hello? World?"
```

当 fastapi 收到 `/hi` 这个 URL 的 GET 请求时，会自动调用 async greet(), 无需在任何地方添加 await

但是, 对于其他的 async def 函数, 调用的时候必须在前面加上 await

> FastAPI 会运行一个异步事件循环，用于执行异步路径函数(async path functions)，同时也会使用一个线程池来处理同步函数(synchronous path functions), 这样就不需要手动调用 `asyncio.gather()` 和 `asyncio.run()` 之类的方法

### 示例: CURD API

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str = None
    price: float
    tax: float = None

@app.post("/items/")
async def create_item(item: Item):
    total = item.price + (item.tax or 0)
    return {"name": item.name, "total_price": total}

@app.get("/")
def read_root():
    return {"message": "FastAPI is working!"}
```

运行

```
uvicorn main:app --reload
```

- 还可以使用 Gunicorn 部署4个 Uvicorn 异步服务
  ```
  gunicorn main:app --workers 4 --worker-class \
  uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
  ```
  实际上也可以直接诶使用 uvicorn 运行多个进程, 但是这样无法进行进程管理，因此使用 gunicorn 的方法一般更多被使用

### 性能提升

如果 API 返回大量数据, 使用 ORJSON 加快序列化速度

```python
from fastapi import FastAPI
from fastapi.responses import ORJSONResponse

app = FastAPI(default_response_class=ORJSONResponse)
```
