+++
date = '2025-08-06T10:00:00+08:00'
draft = false
title = 'FastAPI Parameters'
+++
FastAPI 是一个现代、快速（高性能）的 Python Web 框架, 它自动处理参数的解析、验证和文档生成

本文将介绍 FastAPI 中三类最常用的参数: **路径参数 (Path Parameters)**、**查询参数 (Query Parameters)** 和 **请求体(Request Body)** 的用法与原理




### 1. Path Parameters 路径参数
路径参数是 URL 路径中的动态部分, 使用 `{}` 包裹表示
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: str):
    return {"item_id": item_id}
```
访问 `/items/foo` 返回:
```python
{"item_id": "foo"}
```

#### Data conversion & validation 类型声明与自动转换
可以为路径参数声明类型, FastAPI 会自动解析并验证:
```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```
访问 `/items/3`, `item_id` 会被转换为 int 类型

#### Routing orders 路由匹配顺序
路径匹配按声明顺序执行, 例如
```python
@app.get("/users/me")
async def read_user_me():
    return {"user_id": "current_user"}

@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```
必须先声明 `/users/me`, 否则会被 `/users/{user_id}` 捕获


#### Predefined enum values 预定义枚举值
使用 Python 的 `Enum` 定义一组可选的路径参数值
```python
from enum import Enum

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    return {"model_name": model_name}
```
Swagger 文档会自动显示可选值


#### Path parameters containing paths 路径型参数
默认路径参数不能包含斜杠 `/`, 但可以用 `:path` 声明允许匹配完整路径
```python
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    return {"file_path": file_path}
```
访问 `/files/home/user/file.txt`, `file_path` 会是 `"home/user/file.txt"`




### 2. Query Parameters 查询参数
查询参数是 URL `?` 后的键值对, 不属于路径部分
```python
fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}]

@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    return fake_items_db[skip : skip + limit]
```
访问 `/items/?skip=0&limit=10` 时, 会自动把查询参数 `skip` 和 `limit` 转成 `int`

#### Optional parameters 可选参数默认值
给查询参数赋默认值即为可选
```python
@app.get("/items/{item_id}")
async def read_item(item_id: str, q: str | None = None):
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}
```
`q` 是可选查询参数, 默认为 `None`

#### Query parameter type conversion 查询参数类型转换
```python
@app.get("/items/{item_id}")
async def read_item(item_id: str, q: str | None = None, short: bool = False):
    ...
```
支持自动把字符串转换成布尔值, 以下都会被识别为 `True`
```
http://127.0.0.1:8000/items/foo?short=1
```
或者
- `?short=true`
- `?short=on`
- `?short=yes`

#### Multiple path and query parameters 多路径查询参数组合
路径参数和查询参数可混合使用, 无需声明顺
```python
@app.get("/users/{user_id}/items/{item_id}")
async def read_user_item(user_id: int, item_id: str, q: str | None = None, short: bool = False):
    ...
```

#### Required query parameters 必填查询参数
未设置默认值的查询参数为必填参数
```python
@app.get("/items/{item_id}")
async def read_item(item_id: str, needy: str):
    ...
```
上面的 `needy` 就是一个必填的 `str` 类型

当然也可以定义一些必填参数, 以及有默认值的可选参数
```python
@app.get("/items/{item_id}")
async def read_user_item(
    item_id: str, needy: str, skip: int = 0, limit: int | None = None
):
    ...
```
- `needy` & `item_id`, 必填 `str` 类型
- `skip`, 默认值为 0 的类型
- `limit`, 一个可选的类型

[注]
- 路径参数永远是必填的, 因为它们来自 URL 本身
    ```python
    @app.get("/items/{item_id}")
    def read(item_id: str = "123"):  # 这里写默认值是无效的
        ...
    ```
- 类型为 `Optional[...]` 或 `type | None` 不等于可选参数, 仍然要配合默认值 `= None` 才是可选
    ```python
    def func(x: int | None):         # 必填
    def func(x: int | None = None):  # 可选
    ```




### 3. Request Body
当通过 API 传送数据的时候, 通常通过 request body 发送

request body 是 client 客户端发送给 API 的数据, 而 response body 是 API 发送给 client 的数据

#### Pydantic's BaseModel
使用 Pydantic 定义数据模型
```python
from fastapi import FastAPI
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

app = FastAPI()

@app.post("/items/")
async def create_item(item: Item):
    return item
```

#### Declare it as a parameter
在路由中声明请求体
```python
@app.post("/items/")
async def create_item(item: Item):
    return item
```
FastAPI 会:
- 读取 request body, 并转换为 JSON
- 校验字段和类型
    - 返回类型错误时给出详细反馈, 包括数据那里以及导致了什么错误
- 提供编辑器类型提示
- 生成模型的 JSON Schema 定义, 也可以在项目中任何位置使用
- 根据 schema 自动生成文档

#### Request body + path + query parameters 路径参数、查询参数与请求体同时使用
```python
@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item, q: str | None = None):
    result = {"item_id": item_id, **item.dict()}
    if q:
        result.update({"q": q})
    return result
```
这个函数参数会被以下方式识别:
- 如果参数同时在 path 中声明, 被当成 path parameter
- 如果参数为单一类型, 如 `int`, `float`, `str` 或 `bool` 等, 将会被解释为 query parameter
- 如果参数声明为一个 Pydantic Model, 将被解释为 request body
