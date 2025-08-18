+++
date = '2025-08-11T8:00:00+08:00'
draft = false
title = 'Fastapi Cookie and Header Parameters'
tags = ["FastAPI"]
+++
这篇文章介绍 Fastapi 的 Cookie 和 Header 参数

### Cookie Parameters
通过定义 `Query` 和 `Path` 参数一样定义 `Cookie` 参数
```Python
from typing Annotated
from fastapi import Cookie, FastAPI

app = FastAPI()

@app.get("/items/")
async def read_items(ads_id: Annotated[str | None, Cookie()] = None):
    return {"ads_id": ads_id}
```

#### Cookie Parameters Models
如果有一组相关的 cookies, 可以使用 Pydantic model 来声明.

这样可以在多个部分复用这个模型, 同时还能一次性为所有参数声明验证规则和元数据.

下面使用 Pydantic 模型定义 Cookies, 然后将参数声明为 `Cookie`
```Python
from typing import Annotated
from fastapi import FastAPI, Cookie
from pydantic import BaseModel

app = FastAPI()

class Cookie(BaseModel):
    session_id: str
    fatebook_tracker: str | None = None
    googall_tracker: str | None = None

@app.get("/items/")
async def read_items(cookies: Annotated[Cookies, Cookie()]):
    return cookies
```

#### Forbid Extra Cookies 禁止额外的Cookie
在某些场景下(虽然并不常见), 可能希望限制 API 只能接收特定的 Cookie.
这样, API 就可以"自己"管理 Cookie 同意策略了.
```Python
from typing import Annotated
from fastapi import FastAPI, Cookie
from pydantic import BaseModel

app = FastAPI()

class Cookies(BaseModel):
    model_config = {"extra": "forbid"} # forbid extra cookies

    session_id: str
    fatebook_tracker: str | None = None
    googall_tracker: str | None = None

@app.get("/items/")
async def read_items(cookies: Annotated[Cookies, Cookie()]):
    return cookies
```
这样, 如果客户端发送额外的 cookies, 则会收到一个错误响应. 例如, 客户端发送了 `santa_tracker` 这个额外 Cookie
```Python
santa_tracker = good-list-please
```
将会收到如下错误响应
```JSON
{
    "detail": [
        {
            "type": "extra_forbidden",
            "loc": ["cookie", "santa_tracker"],
            "msg": "Extra inputs are not permitted",
            "input": "good-list-please",
        }
    ]
}
```




### Header Parameters
同样的, 通过定义 `Query` 和 `Path` 参数一样定义 `Header` 参数
```Python
from typing import Annotated
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/items/")
async def read_items(user_agent: Annotated[str | None, Header()] = None):
    return {"User-Agent": user_agent}
```


#### Automatic conversoin 自动转换
`Header` 拥有一些在 `Path`, `Query` 和 `Cookie` 上的额外功能

大多数标准的 header 都通过一个连字符(hyphen character), 也称为减号(minus symbol)分开, 
但是变量 `user-agent` 这样在 Python 中是不合法的. 所以, 默认情况下 `Header` 会将参数名中的 hypen(-) 使用下划线 undersocre(\_) 替换.

同样的, HTTP headers 是不区分大小写的, 所以可以使用标准的 Python 风格 (snake\_case). 因此可以使用 `user_agent` 在 Python 代码中, 而不需要首字母大写成 `User_Agent`.

如果想要禁止这种自动转换, 需要将 `Header` 的参数 `convert_undersocres` 设置为 `False`

```Python
from typing import Typing
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/items/")
async def read_items(
    strange_header: Annotated[str | None, Header(convert_undersocres=False)] = None
):
    return {"strange_header": strange_header}
```

#### Duplicate headers 重复请求头
一个请求中可能会收到重复的 headers, 也就是同一个 header 有多个值.

可以在类型声明中使用 `list` 来处理这种情况, 这样会得到一个 Python 列表.

例如要声明一个可能多次出现的 `X-Token` 头部, 可以这样写:
```Python
from typing import Annotated
from fastapi import FastAPI, Header

app = FastAPI()

@app.get("/items/")
async def read_items(x_token: Annotated[list[str] | None, Header()] = None):
    return {"X-Token values": x_token}
```

如果向该接口发送两个这样的 HTTP headers
```
X-Token: foo
X-Token: bar
```

返回类似这样
```JSON
{
    "X-Token values": [
        "bar",
        "foo"
    ]
}
```


#### Header parameters models 请求头参数模型
同样可以使用 Pydantic model 定义 Header Parameters, 这样可以在多个地方复用模型, 还能一次性为所有参数声明规则和元数据
```Python
from typing import Annotated
from fastapi import FastAPI, Header
from pydantic import BaseModel

app = FastAPI()

class CommonHeaders(BaseModel):
    host: str
    save_data: str
    if_modified_since: str | None = None
    traceparent: str | None = None
    x_tag: list[str] = []

@app.get("/items")
async def read_items(headers: Annotated[CommonHeaders, Header()]):
    return headers
```

#### Forbid extra headers 禁止额外请求头
同样也可以禁止额外的 headers
```Python
class CommonHeaders(BaseModel):
    model_config = {"extra": "forbid"}  # 禁止额外字段
    ...
```

如果客户端尝试发送额外的 Header，将会收到错误响应. 例如, 客户端发送了 `tool` 这个额外 Header
```
tool: plumbus
```

将会收到如下错误响应
```json
{
    "detail": [
        {
            "type": "extra_forbidden",
            "loc": ["header", "tool"],
            "msg": "Extra inputs are not permitted",
            "input": "plumbus"
        }
    ]
}
```

#### Disable convert undersocres 禁止转换下划线
同样可以禁用自动下换线转换

与普通的 Header 参数一样, 如果参数名中包含下划线 undersocre (\_), FastAPI 会自动将其转换为连字符 hypens (-)
```Python
async def read_items(
    headers: Annotated[CommonHeaders, Header(convert_underscores=False)],
):
    ...
```

> 在将 `convert_underscores` 设置为 False 前, 注意有些 HTTP 代理和服务器不允许带下划线的头部字段
