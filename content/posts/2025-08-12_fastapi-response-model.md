+++
date = '2025-08-12T8:00:00+08:00'
draft = false
title = 'FastAPI Response Model'
tags = ["FastAPI"]
+++
本篇文章介绍 FastAPI 的返回类型 response model

可以在返回函数的类型注解中声明该接口的响应数据类型

类型注解的用法和输入数据参数一样, 可以使用:
- Pydantic 模型
- list 列表
- dict 字典
- scalar 标量值 (int, bool ...)

```Python
@app.post("/items/")
async def create_item(item: Item) -> Item:
    ...

@app.get("/items/")
async def read_items() -> list[Item]:
    ...
```

FastAPI 会使用返回类型完成一下事情:
- 验证返回类型
    如果返回的数据无效, 说明业务代码有问题, FastAPI 会返回服务器错误, 而不是把数据发给客户端

- 在 OpenAPI 中为响应添加 JSON Schema
    用于自动生成接口文档, 自动生成客户端代码

- 最重要的是
    它会限制并过滤出数据, 只保留返回类型中定义的字段


### `response_model` Parameter
有时候可能需要返回的数据和类型注解不完全一致, 例如:
- 可能想返回字典或数据库对象, 但声明的响应类型为 Pydantic 模型
- 这样 Pydantic 会做数据文档、验证等工作, 即使返回的是字典或 ORM 对象

如果直接用返回类型注解, 编辑器会提示类型不匹配的错误

这种情况下, 可以用路径装饰器的 `response_model` 参数来声明响应类型, 而不是用返回类型注解

```Python
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
    tags: list[str] = []

@app.post("/items/", response_model=Item)
async def create_item(item: Item) -> Any:
    return item

@app.get("/items/", response_model=list[Item])
async def read_items() -> Any:
    return [
        {"name": "Portal Gun", "price": 42.0},
        {"name": "Plumbus", "price": 32.0},
    ]
```

注意:
- `response_model` 是装饰器(`get`、`post` 等方法)的参数, 不是函数的参数
- 接收的类型和 Pydantic 字段定义一样, 可以是单个模型, 也可以是模型列表等
- FastAPI 用其做数据库验证、文档生成、以及过滤输出数据

> 如果使用 mypy 之类做 static type check, 可以声明函数返回类型为 `Any`

- `response_model` 优先级
    如果同时声明了 `response_model` 和返回类型, 则 `response_model` 会优先生效  
    如果想要禁用响应模型, 可以设置 `response_model=None` (用于一些非 Pydantic 类型的返回值)


### Return the Same Input Data 返回相同数据数据
很多情况下, 希望模型返回与输入模型相同的数据

这式, 可以在路径函数中直接声明 `response_model=YourModel`, FastAPI 会自动处理

```Python
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr

app = FastAPI()

class UserIn(BaseModel):
    username: str
    password: str
    email: EmailStr
    full_name: str | None = None

# Don't do this in production!
@app.post("/user/")
async def create_user(user: UserIn) -> UserIn:
    return user
```
> 不要在生产环境中以明文形式存储用户密码, 也不要像这样直接返回密码

### Add an Output Model 添加输出模型
我们可以改成: 输入模型包含明文密码, 输出模型不含
```Python
from typing import Any

from fastapi import FastAPI
from pydantic import BaseModel, EmailStr

app = FastAPI()

# Input model
class UserIn(BaseModel):
    username: str
    password: str
    email: EmailStr
    full_name: str | None = None

# Output model
class UserOut(BaseModel):
    username: str
    email: EmailStr
    full_name: str | None = None


@app.post("/user/", response_model=UserOut) # output
async def create_user(user: UserIn) -> Any:
    return user # like input
```
这样, 即使路径操作函数返回的对象中包含该字段, FastAPI 也会按照 `response_model=UserOut` 来过滤密码


### Return Type and Data Filtering 返回类型与数据过滤
延续上面的例子, 希望函数的类型注解和实际返回值不同:

函数返回的对象可能包含更多数据, 但响应中只保留输出模型声明的字段

之前由于类不同, 只能用 `response_model`, 这样就失去了编辑器和类型检查对返回值的检查

大多数情况下, 我们只是想去掉或过滤掉部分数据, 这时可以用 类继承(classes and inheritance) 来兼顾类型注解和数据过滤
```Python
class BaseUser(BaseModel):
    username: str
    email: EmailStr
    full_name: str | None = None

class UserIn(BaseUser):
    password: str

@app.post("/user/")
async def create_user(user: UserIn) -> BaseUser:
    return user
```
通过这种方式:
1. Type Annotations and Testing 编辑器和类型检查工具支持: `UserIn` 是 `BaseUser` 的子类, 返回 `UserIn` 实例完全符合 `BaseUser` 类型要求
2. FastAPI Data Filtering 数据过滤: 响应中会自动去掉 `password` 字段, 只保留 `BaseUser` 中声明的字段


### Other Return Type Annotations 其他类型注解
有些时候, 返回的内容不是有效的 Pydantic 字段, 但在函数中添加了注解, 为了获取工具支持

#### Return a response directly 直接返回响应
最常见的就是直接返回一个 Resposne
```Python
from fastapi import FastAPI, Response
from fastapi.resposnes import JSONResponse, RedirectResponse

app = FastAPI()

@app.get("/portal")
async def get_protal(teleport: bool = False) -> Response:
    if teleport:
        return RedirectResponse(url="https://www.youtube.com/watch?v=dQw4w9WgXcQ")
    return JSONResponse(content={"message": "Here's your interdimensional portal."})
```
这种简单情况由 FastAPI 自动处理, 因为返回类型注解是 Response 类

开发工具也能正常工作, 因为 `RedirectResponse` 和 `JSONResponse` 都是 `Response` 的子类, 所以类型注解是正确的


#### Invalid return type annotations 无效的类型注解
但是, 当返回一些其他任意对象(不是有效的 Pydantic 类型, 例如数据库对象)并在函数中这样注解时, FastAPI 会尝试从该类型注解创建一个 Pydantic 响应模型, 然后会失败

如果使用了联合模型, 其中有一个或多个不是有效的 Pydantic 类型, 同样会失败
```Python
from fastapi import FastAPI, Response
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.get("/portal")
async def get_portal(teleport: bool = False) -> Response | dict:
    if teleport:
        return RedirectResponse(url="https://www.youtube.com/watch?v=dQw4w9WgXcQ")
    return {"message": "Here's your interdimensional portal."}
```
这会失败是因为类型注解不是单一的 Pydantic 类型, 也不是单一的 Response 类或子类, 而是 Response 和 dict 之间的联合类型

#### Disable response Model 禁用响应类型
如果不希望 FastAPI 执行默认的数据验证、文档生成、过滤等操作, 但是又想在函数中保留返回类型注解, 以获得编辑器和类型检查工具的支持, 这种情况下设置 `response_model=None` 来禁用响应生成
```Python
from fastapi import FastAPI, Response
from fastapi.response import RedirectResponse

app = FastAPI()

@app.get("/portal", response_model=None)
async def get_protal(teleport: bool = False) -> Response | dict:
    if teleport:
        return RedirectResponse(url="https://www.youtube.com/watch?v=dQw4w9WgXcQ")
    return JSONResponse(content={"message": "Here's your interdimensional portal."})
```


### Response Model Encoding Parameters 响应模型编码参数
响应模型可能有默认值
```Python
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5
    tags: list[str] = []
```
例如, 在 NoSQL 数据库中哟许多可选属性的模型, 但不想发送默认值的很长的 JSON 响应

可以使用 path operation operator 的 `response_model_exclude_mode` 参数来去除默认值
```Python
@app.get("/items/{item_id}", response_model=Item, response_model_exclude_unset=True)
async def read_item(item_id: str):
    return items[item_id]
```
- `description: str | None = None` 默认值为 `None`
- `tax: float = 10.5` 的默认值为 10.5
- `tags: List[str] = []` 的默认值是空列表 `[]`

此时, 如果向该路径发送 ID 为 `foo` 的项目请求
```yaml
"foo": {"name": "Foo", "price": 50.2}
```

响应将是
```yaml
{
    "name": "foo",
    "price": 50.2,
}
```

- `response_model_include` 和 `response_model_exclude`

也可以使用 path operation parameter 中的 `response_model_include` 和 `response_model_exclude`, 他们接受一个包含属性名称字符串的 `set`, 用于包含(省略其余部分)或排除(包含其余部分)

```Python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float = 10.5


items = {
    "foo": {"name": "Foo", "price": 50.2},
    "bar": {"name": "Bar", "description": "The Bar fighters", "price": 62, "tax": 20.2},
    "baz": {
        "name": "Baz",
        "description": "There goes my baz",
        "price": 50.2,
        "tax": 10.5,
    },
}


@app.get(
    "/items/{item_id}/name",
    response_model=Item,
    response_model_include={"name", "description"}, # 包含
)
async def read_item_name(item_id: str):
    return items[item_id]


@app.get(
    "/items/{item_id}/public",
    response_model=Item,
    response_model_exclude={"tax"}, # 排除
)
async def read_item_public_data(item_id: str):
    return items[item_id]
```

虽然可以通过上述方法自定义返回参数包含哪些, 但还是建议使用多个类来实现该功能, 而不这些参数

这是因为, 即使使用 `response_model_include` 或 `response_model_exclude` 来省略某些属性, 在应用程序的 OpenAPI 中生成的 JSON Schema 仍将是完整模型的 Schema

这对于 `response_model_by_alias` 也是一样的

#### Using `list` instead of `set`s
如果忘记使用集和而使用元组或列表, FastAPI 仍会将其转换为集和, 确保正常工作
```Python
@app.get(
    "/items/{item_id}/name",
    response_model=Item,
    response_model_include=["name", "description"],  # 使用列表
)
async def read_item_name(item_id: str):
    return items[item_id]

@app.get(
    "/items/{item_id}/public",
    response_model=Item,
    response_model_exclude=["tax"],  # 使用列表
)
async def read_item_public_data(item_id: str):
    return items[item_id]
```

### Response Status Code 响应状态码
就像可以指定响应模型一样, 也可以在任何路径操作中使用 `status_code` 参数声明用于响应:
- `@app.get()`
- `@app.post()`
- `@app.put()`
- `@app.delete()`

```Python
from fastapi import FastAPI

app = FastAPI()


@app.post("/items/", status_code=201)
async def create_item(name: str):
    return {"name": name}
```

`status_code` 是"装饰器"方法的一个参数, 而不是你的路径操作函数 *path operation function* 的参数

`status_code` 参数接收一个表示 HTTP 状态码的数字, 也可以接收一个 `IntEnum`, 比如 Python 中的 `http.HTTPStatus`

- 将在响应中返回该状态码
- 并在 OpenAPI 模式中也如此记录

#### About HTTP status codes
在 HTTP 协议中, 会在响应中发送一个3位数的数字状态码

这些状态码又一个相关联的名称便于识别, 但重要的是数字本身

- 100~199: 用于"信息", 很少会直接使用它们, 这些状态码的响应不能有响应体
- 200~299: 用于"成功"的响应, 这些是最常用的
    - 200 的默认的"成功"响应, 表示一切 OK
    - 201 表示已创建, 通常在数据库中创建新记录后使用
    - 204 表示无内容, 当没有内容返回给客户端时使用此响应, 因此不能有响应体
- 300~399: 用于"重定向", 这些状态码的响应可能有也可能没有响应体. 但 304 (未修改) 除外, 它必须没有响应体
- 400~499: 用于"客户端错误"响应,
    - 404 用于"未找到"的响应
    - 400 客户端通用错误
- 500~599: 用于服务器错误, 几乎从不直接使用它们. 当的应用代码或服务器的某个部分出错时, 它会自动返回这些状态码之一

要了解更多关于每个状态码的信息以及哪个代码用于什么目的，请查阅 [MDN 关于 HTTP 状态码的文档](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)

#### Shortcut to remember the names
除了直接使用数字外, 还可以使用 `fastapi.status` 中的便捷变量
```Python
from fastapi import FastAPI, status

app = FastAPI()

@app.post("/items/", status_code=status.HTTP_201_CREATED)
async def create_item(name: str):
    return {"name": name}
```
这只是一直便利, 都是一样的树枝, 但这样可以使用编辑器的自动补全功能

> 也可以使用 `from starlette import status`
