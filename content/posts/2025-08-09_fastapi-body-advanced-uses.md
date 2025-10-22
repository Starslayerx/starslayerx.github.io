+++
date = '2025-08-09T8:00:00+08:00'
draft = false
title = 'FastAPI Body Advanced Uses'
tags = ["FastAPI"]
+++

本篇文章介绍 FastAPI Request Body 的进阶用法

### Body - Multiple Parameters

首先, 可以将`Path`, `Query` 和 request body 参数声明自由的写在一起

对于 request body 参数可以是可选的, 并且可设置为默认的 `None`

```Python
from typing import Annotated

from fastapi import FastAPI, Path
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.put("/items/{item_id}")
async def update_item(
    item_id: Annotated[int, Path(title="The ID of the item to get", ge=0, le=1000)], # Path
    q: str | None = None,     # Query
    item: Item | None = None, # body
):
    results = {"item_id": item_id}
    if q:
        results.update({"q": q})
    if item:
        results.update({"item": item})
    return results
```

#### Multiple body parameters 多参数请求体

在上面例子中, FastAPI 期望一个包含 `Item` 属性的 JSON body, 例如

```JSON
{
    "name": "Foo",
    "description": "The pretender",
    "price": 42.0,
    "tax": 3.2
}
```

但也可以声明多个body parameters, 例如 `item` 和 `user`

```Python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None


class User(BaseModel):
    username: str
    full_name: str | None = None

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item, user: User):
    results = {"item_id": item_id, "item": item, "user": user}
    return results
```

在这种情况下, FastAPI 会检测到函数有一个 body parameter, 这时会使用中的参数名作为请求体的 key(field names), 并期望如下结构:

```JSON
{
    "item": {
        "name": "Foo",
        "description": "The pretender",
        "price": 42.0,
        "tax": 3.2
    },
    "user": {
        "username": "dave",
        "full_name": "Dave Grohl"
    }
}
```

FastAPI 会自动进行请求解析、类型转换、验证, 并在 OpenAPI 文档中反映出这种结构

#### Singular values in body 请求体中的单个参数

和 `Query` 、`Path` 可以添加额外信息一样, FastAPI 也提供了 `Body` 来对请求参数添加额外信息

例如, 除了 `item` 和 `user` 外, 还想在请求体中添加一个 `importance` 字段, 如果直接写 `importance: int` 则会被当作查询参数

可以通过 `Body()` 明确告诉 FastAPI 把它当作一个 body parameter

```Python
@app.put("/items/{item_id}")
async def update_item(
    item_id: int, item: Item, user: User, importance: Annotated[int, Body()]
):
    ...
```

这种情况下, FastAPI 会期待如下的请求体:

```JSON
{
    "item": {
        "name": "Foo",
        "description": "The pretender",
        "price": 42.0,
        "tax": 3.2
    },
    "user": {
        "username": "dave",
        "full_name": "Dave Grohl"
    },
    "importance": 5
}
```

它同样会自动转换数据类型、校验并生成文档

#### Multiple body params and query 多个请求体参数和查询参数

也可以在多请求体参数的基础上, 添加查询参数

```Python
@app.put("/items/{item_id}")
async def update_item(
    *,                    # 强制 key=value
    item_id: int,
    item: Item,
    user: User,
    importance: Annotated[int, Body(gt=0)],
    q: str | None = None, # 查询参数
):
    ...
```

#### Embed a single body parameter 嵌入单个请求体参数

假设只有一个请求体参数 `item: Item`, 默认情况下 FastAPI 期望请求体就是一个 `Item` 对应的结构

```JSON
{
  "name": "Foo",
  "description": "The pretender",
  "price": 42.0,
  "tax": 3.2
}
```

但若希望如下带有 `item`key 的结构

```JSON
{
  "item": {
    "name": "Foo",
    "description": "The pretender",
    "price": 42.0,
    "tax": 3.2
  }
}
```

那么可以使用 `Body(embed=True)`

```Python
@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    item: Annotated[
            Item,
            Body(embed=True),  # embed a single param
        ]
    ):
    ...
```

这将使 FastAPI 将请求体视为嵌套结构, key 为 `item`

### Body - Fields

除了可以在 _path_ operation (路径操作)函数参数中使用 `Query`、`Path`和`Body`来声明额外的验证和数据, 还可以在 Pydantic 模型内部的 `Field` 的字段验证规则和元数据

#### Declare model attributes 声明模型字段属性

首先要导入 Filed

```Python
from pydantic import BaseModel, Field # import Filed
```

可以在模型字段上使用 `Filed` 来添加验证规则和信息

```Python
class Item(BaseModel):
    name: str
    description: str | None = Field(
        default=None, title="项目的描述", max_length=300
    )
    price: float = Field(gt=0, description="价格必须大于 0")
    tax: float | None = None
```

实际上, `Query`、`Path` 和其他类, 都继承自一个公共的 `Param` 类, 而 `Param` 是 `Pydantic` 的 `FieldInfo` 类的子类, `pydantic.Field()` 返回的就是一个 `FieldInfo` 实例

### Body - Nested Models

在 FastAPI 中, 可以定义、校验、文档化并使用任意深度嵌套的模型

#### List fields 列表字段

可以将字段定义为某种子类型, 例如 Python 的 `list`

```Python
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
    tags: list = [] # list
```

- List fields with type parameter 带类型参数的列表字段

Python 提供一种"类型参数"的方法, 来指定列表类型

```Python
# Python 3.10+
tags: list[str] = []
```

对于py3.10之前的版本, 需要使用 `typing` 模块

```Python
tags: List[str] = []
```

#### Set types 集和类型

如果不希望 tages 重复, 则使用 `set` 更加合适

```Python
class Item(BaseModel):
    ...
    tags: set[str] = set()
```

这样即使客户端传来重复元素, FastAPI 也会自动去重并返回一个唯一元素集合

#### Nested Models 嵌套模型

Pydantic 的每个字段都可以是另一模型, 从而形成嵌套结构

```Python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Image(BaseModel):
    url: str
    name: str

class Item(BaseModel):
    ...
    image: Image | None = None

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item):
    return {"item_id": item_id, "item": item}
```

此时的 FastAPI 会期望请求体为如下结构:

```JSON
{
  "name": "Foo",
  "description": "The pretender",
  "price": 42.0,
  "tax": 3.2,
  "tags": ["rock", "metal", "bar"],
  "image": {
    "url": "http://example.com/baz.jpg",
    "name": "The Foo live"
  }
}
```

这样使用 FastAPI 会获得:

- 编辑器自动补全
- 类型转换
- 数据校验
- 自动生成文档

#### Special types and validation 特殊类型与验证

除了像 `str`, `int`, `float` 这类 singular types, 还可以使用更加负责的继承于 `str` 的 singular types, 全部类型可以在 [Pydantic's Type Overview](https://docs.pydantic.dev/latest/concepts/types/) 查看

下面是 `HttpUrl` 的例子

```Python
from pydantic import HttpUrl

class Image(BaseModel):
    url: HttpUrl
    name: str
```

这样会检查 JSON schema 中的 url 是否合法, 并在 OpenAPI 文档中显示

#### Attributes with lists of submodels 带有子模型属性的列表

```Python
class Image(BaseModel):
    url: HttpUrl
    name: str


class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
    tags: set[str] = set()
    images: list[Image] | None = None # lists of submodels
```

此时 FastAPI 会期望请求体有一个 `images` 字段, 为 `Image` 对象的列表

```JSON
{
    "name": "Foo",
    "description": "The pretender",
    "price": 42.0,
    "tax": 3.2,
    "tags": [
        "rock",
        "metal",
        "bar"
    ],
    "images": [
        {
            "url": "http://example.com/baz.jpg",
            "name": "The Foo live"
        },
        {
            "url": "http://example.com/dave.jpg",
            "name": "The Baz"
        }
    ]
}
```

#### Deeply nested models 深度嵌套模型

可以定义任意深度的嵌套模型

```Python
from fastapi import FastAPI
from pydantic import BaseModel, HttpUrl

app = FastAPI()


class Image(BaseModel):
    url: HttpUrl
    name: str


class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
    tags: set[str] = set()
    images: list[Image] | None = None


class Offer(BaseModel):
    name: str
    description: str | None = None
    price: float
    items: list[Item]

@app.post("/offers/")
async def create_offer(offer: Offer):
    return offer
```

#### Bodies of pure lists 纯列表请求体

如果请求体的顶层是一个数组(例如上传多个图片), 可以直接将参数类型声明为列表:

```Python
from fastapi import FastAPI
from pydantic import BaseModel, HttpUrl

app = FastAPI()

class Image(BaseModel):
    url: HttpUrl
    name: str

@app.post("/images/multiple/")
async def create_multiple_images(images: list[Image]):
    return images
```

#### Bodies of arbitrary `dict`S 任意字典作为请求体

可以声明请求体为一个字典 (键和值都可指定类型)

```Python
@app.post("/index-weights/")
async def create_index_weights(weights: dict[int, float]):
    return weights
```

- 虽然 JSON 标准只支持字符串作为 key, 但 Pydantic 会自动将字符串形式的数字转换为 int
- 因此, 如果客户端发送 `{ "1": 0.1, "2": 0.2 }`, 接收到的将是 `{1: 0.1, 2: 0.2}`
