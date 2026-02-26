+++
date = '2025-08-07T8:00:00+08:00'
draft = false
title = 'FastAPI Parameters and Validations'
categories = ['Note']
tags = ["FastAPI"]
+++

这篇文章介绍 FastAPI 中的参数验证功能

### Query Parameters and String Validations

FastAPI 允许为参数声明额外的信息和验证规则

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/")
async def read_items(q: str | None = None):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

`q` 是类型为 `str | None` 的查询参数, 这意味着它可以是字符串, 也可以是 `None`. 其默认值是 `None`, 因此 FastAPI 会识别它为“可选参数”

> FastAPI 通过 `= None` 的默认值知道该参数是非必填的

使用 `str | None` 还能帮助编辑器提供更好的类型提示和错误检测

#### Additional validation 额外验证

即使 `q` 是可选的, 但仍然可以设置条件: 如果提供了 `q`, 则长度不能超过50个字符

使用 `Query` 和 `Annotated` 来实现

```python
from typing import Annotated
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def read_items(q: Annotated[str | None, Query(max_length=50)] = None):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

使用 `Annotated` 包装后, 就可以传递额外的元数据(`Query(max_length=5)`), 用于校验或者文档

注意: 使用 `Annotated` 的时候，不能在 `Query()` 中再次使用 `default`

- ❌ 错误写法
  ```python
  q: Annotated[str, Query(default="rick")] = "morty"
  ```
- ✅ 正确写法
  ```python
  q: Annotated[str, Query()] = "rick"
  ```

使用 `Annotated` 有以下优点

- 默认值直接写在函数参数上，更符合 Python 风格
- 该函数在非 FastAPI 环境中调用时也能正常工作
- 类型检查器能更准确提示
- 可复用于如 Typer 等其它框架
- `Annotated` 可附加多个元数据

#### More Validations 更多验证

- 也可以添加参数 `min_length`
  ```python
  @app.get("/items/")
  async def read_items(
      q: Annotated[str | None, Query(min_length=3, max_length=50)] = None,
  ):
      ...
  ```
- regular expressions 正则表达式

  ```python
  @app.get("/items/")
  async def read_items(
      q: Annotated[
          str | None, Query(min_length=3, max_length=50, pattern="^fixedquery$")
      ] = None,
  ):
      ...
  ```

  `^`: 以后面字符串开始, 之前没有其他字符串  
   `fixedquery`: 完全匹配的单词  
   `$`: 在此结束, 之后没有更多字符

- default values 默认值  
   除了 `None`, 也可以设置其他默认值

  ```python
  q: Annotated[str, Query(min_length=3)] = "fixedquery"
  ```

- reuqired parameters 必填参数  
   如果想让参数 q 是必填的, 不设置默认值即可

  ```python
  q: Annotated[str, Query(min_length=3)]
  ```

  即使参数可以为 None, 但仍强制要求传值

  ```python
  q: Annotated[str | None, Query(min_length=3)]
  ```

- query parameter list / multiple values 参数列表/多个值  
   可以接收多个值的查询参数

  ```python
  @app.get("/items/")
  async def read_items(q: Annotated[list[str] | None, Query()] = None):
      query_items = {"q": q}
      return query_items
  ```

  访问如下 URL

  ```
  http://localhost:8000/items/?q=foo&q=bar
  ```

  将得到多个 q 查询参数值, URL response 将如下

  ```JSON
  { "q": ["foo", "bar"] }
  ```

  若不使用 `Query()`, FastAPI 会把 `list[str]` 当成 request body (请求体)

#### Declare more metadata 添加更多元信息

这些信息会出现在 OpenAPI 文档中

```python
q: Annotated[str | None, Query(
    title="查询字符串",
    description="用于数据库中模糊搜索匹配的查询字符串",
    min_length=3
)] = None
```

#### Alias parameters 参数别名

有时想使用一个在 Python 中非法的别名, 例如 `item-query`

```
http://127.0.0.1:8000/items/?item-query=foobaritems
```

最接近的变量名为 `item_query`, 但是 `item-query` 不能为变量名

此时, 可以使用别名 `alias`

```python
q: Annotated[str | None, Query(alias="item-query")] = None
```

#### Deprecating parameters 弃用参数

想标记某个参数已被弃用, 可以加上

```python
Query(..., deprecated=True)
```

#### Exclude parameters from OpenAPI 从OpenAPI中隐藏参数

可以设置参数不出现在自动生成的文章中

```python
hidden_query: Annotated[str | None, Query(include_in_schema=False)] = None
```

#### Custom validation 自定义校验

若内建参数不够用, 可以使用 Pydantic v2 的 `AfterValidator`

```python
from fastapi import FastAPI
from pydantic import AfterValidator
from typing import Annotated

def check_valid_id(id: str):
    if not id.startswith(("isbn-", "imdb-")):
        raise ValueError("Invalid ID format, 必须以 'isbn-' 或 'imdb-' 开头")
    return id

@app.get("/items/")
async def read_items(
    id: Annotated[str | None, AfterValidator(check_valid_id)] = None,
):
    if id:
        team = data.get(id)
    else:
        id, item = random.choice(list(data.items()))
    return {"id": id, "name": item}
```

- `value.startswith(("isbn-", "imdb-"))` 可以一次检查多个前缀
- `random.choice(list(data.items()))` 取出随机的键值对

### Path Parameters and Numberic Validations

和使用 `Query` 查询参数声明更多验证规则和元数据一样, 也可以使用 `Path` 为路径参数声明相同类型的规则验证和元数据

#### Import Path 导入路径

首先, 从 `fastapi` 中导入 `Path`, 并导入 `Annotated`

```python
from typing import Annotated
from fastapi import FastAPI, Path, Query

app = FastAPI()

@app.get("/items/{item_id}")
async def read_items(
    item_id: Annotated[int, Path(title="要获取的物品 ID")],
    q: Annotated[str | None, Query(alias="item-query")] = None,
):
    results = {"item_id": item_id}
    if q:
        results.update({"q": q})
    return results
```

#### Declare metadata 声明元数据

可以像在 `Query` 中一样声明所有的参数

```python
item_id: Annotated[int, Path(title="要获取的物品 ID")]
```

> ⚠️ 路径参数总是必填的, 它必须作为路径的一部分存在. 即使将它设为 `None` 或指定默认值, 也不会生效, 它仍然是必须的

#### Order the parameters 自由排序参数

如果希望 query parameter 声明为必填的 `str`, 并且不需要声明任何其他事情, 那么不需要用 `Query()` 包裹

但是对于 path parameter `item_id` 仍然需要使用 `Path`, 并且出于一些原因并不像使用 `Annotated`

如果将有 `defalult` 默认值的参数, 放到没有默认值参数前面, 那么 Python 会报错, 所以要这样声明函数

```Python
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/items/{item_id}")
async def read_items(q: str, item_id: int = Path(title="要获取的物品 ID")):
    results = {"item_id": item_id}
    if q:
        results.update({"q": q})
    return results
```

但是, 如果使用 `Annotated` 就不会有这个顺序的问题, 因为默认值并不写在函数参数中

```Python
@app.get("/items/{item_id}")
async def read_items(
    q: str, item_id: Annotated[int, Path(title="要获取的物品 ID")]
):
    ...
```

#### Order the parameters tricks 参数顺序技巧

如果不想使用 `Annotated`, 但是又想:

- 为查询参数 `q` 不使用 `Query`, 也不设置默认值
- 为路径参数 `item_id` 使用 `Path`
- 两个参数顺序任意
- 不想用 `Annotated`

那可以使用一个小技巧: 在函数参数前面加一个星号 `*`

作用是: 告诉 Python, 后面所有参数必须作为关键字参数传入 (即使用`key=value`的方法, 不能省略参数名)

```Python
async def read_items(*, item_id: int = Path(title="The ID of the item to get"), q: str):
    ...
```

#### Better with `Annotated` 推荐使用`Annotated`

如果使用 `Annotated`, 由于不是用参数默认值来传递 `Path()`、`Query()`, 就不需要使用`*`这种语法

```Python
# Python 3.9+
async def read_items(
    item_id: Annotated[int, Path(title="The ID of the item to get")], q: str
):
    ...
```

#### Number Validations 数字验证

在 FastAPI 中, 可以通过 `Path()`、`Query()` (以及其他参数类) 为数值类型参数添加约数条件, 有以下四种:

- `gt`: greater than (大于)
- `ge`: greater than or equal (大于等于)
- `lt`: less than (小于)
- `le`: less than or equal (小于等于)

这些验证适用于路径参数(path parameter)和查询参数(query parameter), 并且支持 int 和 float 类型

- 整数验证示例 (Path 参数)  
   使用 `ge=1` 表示 `item_id` 必须是一个大于等于1的整数

  ```Python
  from typing import Annotated
  from fastapi import FastAPI, Path

  app = FastAPI()

  @app.get("/items/{item_id}")
  async def read_items(
      item_id: Annotated[int, Path(title="要获取的项目 ID", ge=1)],
      q: str
  ):
      return {"item_id": item_id, "q": q}
  ```

  也可以通过 `ge` 和 `le` 同时限制一个整数的区间范围

  ```Python
  @app.get("/items/{item_id}")
  async def read_items(
      item_id: Annotated[int, Path(title="要获取的项目 ID", gt=0, le=1000)],
      q: str
  ):
      return {"item_id": item_id, "q": q}
  ```

- 浮点数验证示例 (Query 参数)  
   浮点类型的校验同样适用. 例如, 使用 `gt` 可以确保值 严格大于 0

  ```Python
  @app.get("/items/{item_id}")
  async def read_items(
      *,
      item_id: Annotated[int, Path(title="项目 ID", ge=0, le=1000)],
      q: str,
      size: Annotated[float, Query(gt=0, lt=10.5)],
  ):
      return {"item_id": item_id, "q": q, "size": size}
  ```

  - `item_id` 必须在 [0, 1000] 区间内
  - `size` 必须在 (0, 10.5) 区间内
