+++
date = '2025-08-15T8:00:00+08:00'
draft = false
title = 'Microservice with FastAPI'
+++

## What are microservices ?
什么是微服务? 微服务可以有多种不同的定义方式, 具体取决于希望强调微服务架构的哪个方面, 不同作者会给出略有不同但相关的定义

Sam Newman, 微服务领域最有影响力的作者之一, 给出了一个极简的定义:

> “Microservices are small, autonomous services that work together.”

这个定义强调了这样一个事实: 微服务是彼此独立运行的应用程序, 但它们可以协作完成任务. 该定义还强调微服务是 “small (小的)”, 这里的 small 并不是指微服务代码量的大小, 而是指微服务具有狭窄且定义清晰的职责范围, 符合单一职责原则(**Single Responsibility Principle**) —— 即“只做一件事，并把它做好”.

James Lewis 和 Martin Fowler 撰写的一篇开创性文章提供了一个更详细的定义, 他们将微服务定义为一种架构风格(architectural style)

> “an approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms, often an HTTP resource API”

这个定义强调了服务的自主性(autonomy), 指出它们运行在各自独立的进程中. Lewis 和 Fowler 同样强调了微服务职责的狭窄性(narrow scope of responsibilities), 称其为“small”, 并明确指出微服务之间通过轻量协议(如HTTP)进行通信

- **定义**  
    微服务是一种架构风格，其中系统的各个组件被设计为可独立部署的服务(independently deployable services). 微服务围绕明确的业务子领域(business subdomains)进行设计，并通过如 HTTP 等轻量协议(lightweight protocols)相互通信

从以上定义中我们可以看到, 微服务可以被定义为一种架构风格, 其中服务作为组件执行一组小而明确的相关功能. 这意味着微服务是围绕特定的业务子领域来设计和构建的, 例如处理支付、发送邮件或处理客户订单等.

微服务作为独立的进程进行部署, 通常运行在独立的环境中, 并通过定义清晰的接口暴露其能力



## A basic API implementation
这里通过一个 CoffeeMesh 项目的 orders service (订单服务) api 介绍微服务

首先给出 OpenAPI 格式的 API 定义文档 [oas.yaml](https://github.com/abunuwas/microservice-apis/blob/master/ch02/oas.yaml), 可以通过 [Swagger UI](https://editor.swagger.io/) 来查看该文档内容 (OAS 代表 OpenAPI specification/规范, 是一种标准的 REST API 文档)

具体 API 如下
- `/orders`: 检索订单(GET) 和 创建订单(POST)
- `/orders/{order_id}`: 检索某个订单的细节(GET), 更新订单(PUT) 和 删除订单(DELETE)
- `/orders/{order_id}/cancel`: 删除某个订单
- `/orders/{order_id}/pay`: 支付订单

除了 API endpoints, 还有 data models (在 OpenAPI 中被称为 *schemas*). Schemas 告诉客户端需要什么样的数据载荷(payload)以及什么是类型.

例如,**OrderItemSchema** 指定了 **product** 和 **size** 是必填的, 而 **quantity** 属性是可选的, 当这个属性消失的时候, 默认值为 1
```yaml
# file: oas.yaml
 
OrderItemSchema:
  type: object
  required:
    - product
    - size
  properties:
    product:
      type: string
    size:
      type: string
      enum:
        - small
        - medium
        - big
    quantity:
      type: integer
      default: 1
      minimum: 1
```

请求处理流大概下面这样:
HTTP request -> Uvicorn -> FastAPI(Starlette routing -> data -> api endpoints) -> Pydantic

下面是一个 orders API 的最小实现
```Python
from datetime import datetime
from uuid import UUID
from starlette.responses import Response
from starlette import status
from orders.app import app

order = {
    "id": "ff0f1355-e821-4178-9567-550dec27a373",
    "status": "delivered",
    "created": datetime.utcnow(),
    "order": [
        {
            "product": "cappuccino",
            "size": "medium",
            "quantity": 1,
        }
    ]
}

@app.get("/orders")
def get_orders():
    return {"orders": [orders]}

@app.post("/orders", status_code=status.HTTP_201_CREATED)
def create_order():
    return order

@app.get("/orders/{order_id}")
def get_order(order_id: UUID):
    return order

@app.get("/orders/{order_id}")
def update_order(order_id: UUID):
    return order

@app.delete("/orders/{order_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_order(order: UUID):
    return Response(status_code=HTTPStatus.NO_CONTENT.value)

@app.post("/orders/{order_id}/cancel")
def cancel_order(order_id: UUID):
    return order

@app.post("/orders/{order_id}/pay")
def pay_order(order_id: UUID):
    return order
```
现在有了 API 的基本骨架, 后面将继续实现 incoming payload 和 outgoing response 的验证


### Implementing data validation models with pydantic
这里介绍 data validation 和 marshalling

> "Marshalling" 指的是将一个内存中的数据结构转换成一种适合存储或通过网络传输的格式. 
> 在 Web API 的上下文中, Marshalling 特指将一个对象转换为一个数据结构(比如 JSON 或 XML). 
> 以便将其序列化为所选的内容类型, 同时明确指定对象属性的映射关系

点单系统包含了3个shcemas: `CreateOrderSchema`, `GetOrderSchema` 和 `OrderItemSchema`, 可以在[oas.yaml](https://github.com/abunuwas/microservice-apis/blob/master/ch02/oas.yaml#L211)查看

下面使用 Pydantic 实现对应 schema, 可以在 [schema.py](https://github.com/abunuwas/microservice-apis/blob/master/ch02/orders/api/schemas.py)找到

```Python
from enum import Enum

class Size(Enum):
    small = "small"
    medium = "medium"
    big = "big"

class StatusEnum(Enum):
    created = "created"
    paid = "paid"
    progress = "progress"
    cancelled = "cancelled"
    dispatched = "dispatched"
    delivered = "delivered"
```
对于只能从特定值中选择的类型, 定义枚举类型 `Size` 和 `StatusEnum`


```Python
class OrderItemSchema(BaseModel):
    product: str
    size: Size
    quantity: conint(ge=1, strict=True) | None = 1
```
将 `OrderItemSchema` 的属性设置为 `conint`, 这将强制使用整数值, 并且规定数值要大于等于1, 以及默认值1


```Python
class CreateOrderSchema(BaseModel):
    order: conlist(OrderItemSchema, min_items=1)

class GetOrderSchema(CreateOrderSchema):
    id: UUID
    created: datetime
    status: StatusEnum

class GetOrdersSchema(BaseModel):
    orders: List[GetOrderSchema]
```
使用 pydantic 的 `conlist` 类型定义了 `CreateOrderSchema` 的 `order` 属性, 要求列表至少有一个元素

### Validating request payloads with pydantic
上面实现了模型定义, 现在通过将其声明为视图函数的一个参数来拦截请求负载, 并通过将其类型设置为相关的 Pydantic 模型进行验证

代码可以在[api.py](https://github.com/abunuwas/microservice-apis/blob/master/ch02/orders/api/api.py)里找到
```Python
from uuid import UUID

from starlette.response import Response
from starlette import status

from orders.app import app
from orders.api.schemas import CreateOrderSchema # 导入数据模型

@app.post("/orders", status_code=status.HTTP_201_CREATED)
def create_order(order_details: CreateOrderSchema):
    return order

@app.get("/orders/{order_id}")
def get_order(order_id: UUID):
    return order

@app.put("/orders/{order_id}")
def update_order(order_id: UUID, order_details: CreateOrderSchema):
    return order
```

如果发送一个有问题的数据(例如移除 product 字段), FastAPI 将会生成一份错误消息.
```JSON
{
  "detail": [
    {
      "loc": [
        "body",
        "order",
        0,
        "product"
      ],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
```
该错误消息使用 JSON Pointer 来指示问题所在, JSON Pointer 是一种语法, 用来表示 JSON 文档中特定值的路径

例如, `loc: /body/order/0/product` 大概等同于 Pytohn 中的以下表示法 `loc['body']['order'][0]['product']`
- body 指的是请求的主体部分
- order 指的是主体中的 order 键
- 0 指的是 order 列表中的第一个元素
- product 指的是这个元素中的 product 键

有时候参数可能是可选的, 但是并不能为 null. 这里使用 Pydantic 的 `validator()` 装饰器来添加额外的规则

```Python
from pydantic import BaseModel, conint, validator

...

class OrderItemSchema(BaseModel):
    product: int
    size: Size
    quantity: conint(ge=1, strict=True) | None = 1

    @validator('quantity')
    def quantity_non_nullable():
        assert value is not None, "quantity may not be None"
        return value
```

### Marshalling and validating response payloads with pydantic
这里定义一下返回类型 [api.py](https://github.com/abunuwas/microservice-apis/blob/master/ch02/orders/api/api.py)
```Python
from starlette.responses import Response
from starlette import status
from orders.api.schemas import (
    GetOrderSchema,
    CreateOrderSchema,
    GetOrdersSchema,
)

@app.get("/orders", response_model=GetOrdersSchema)
def get_orders():
    return [
        order
    ]

@app.post(
    "/orders",
    status_code=status.HTTP_201_CREATED,
    response_model=GetOrderSchema
)
def create_order(order_details: CreateOrderSchema):
    return order
```
现在, 如果 response payload 中缺少了返回类型需要的属性, FastAPI 则会报错, 如果有多的属性, 则会被去除


### Adding an in-memory list of orders to the API
现在通过一个简单的内存列表来管理订单状态
```Python
import time
import uuid
from datetime import datetime
from uuid import UUID

from fastapi import HTTPException
from starlette.responses import Response
from starlette import status

from orders.app import app
from orders.api.schemas import GetOrderSchema, CreateOrderSchema

ORDERS = [] # in memory list

# 获取订单列表
@app.get("/orders", respones_model=GetOrderSchema)
def get_orders():
    return ORDERS # return order list

# 创建订单
@app.post(
    "/orders",
    status_code=status.HTTP_201_CREATED,
    response_model=GetOrderSchema,
)
def create_order(order_details: CreateOrderSchema):
    # convert Pydantic model -> dict: v1 use .dict(); v2 use .model_dump()
    order = order_details.model_dump()
    order["id"] = uuid.uuid4()

# 获取订单
@app.get("/orders/{order_id}", response_model=GetOrderSchema)
def get_order(order_id: UUID):
    for order in ORDERS:
        if order["id"] == order_id:
            return order
    raise HTTPException(
        status_code=404,
        detail=f"Order with ID {order_id} not found",
    )

# 更新订单
@app.put("/orders/{order_id}", response_model=GetOrderSchema)
def update_order(order_id: UUID, order_details: CreateOrderSchema):
    for order in ORDERS:
        if order["id"] == order_id:
            order.update(order_details.model_dump())
            return order
    raise HTTPException(
        status_code=404,
        detail=f"Order with ID {order_id} not found",
    )

# 删除订单
@app.delete(
    "/orders/{order_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    response_class=Response
)
def delete_order(order_id: UUID):
    for index, order in enumerate(ORDERS):
        if order["id"] == order_id:
            ORDERS.pop(index)
            return Response(status_code=HTTPStatus.NO_CONTENT.value)
    raise HTTPException(
        status_code=404,
        detail=f"Order with ID {order_id} not found",
    )

# 取消订单
@app.post("/orders/{order_id}/cancel", response_model=GetOrderSchema)
def cancel_order(order_id: UUID):
    for order in ORDERS:
        if order["id"] == order_id:
            order["status"] = "cancelled"
            return order
    raise HTTPException(
        status_code=404,
        detail=f"Order with ID {order_id} not found",
    )

# 支付订单
@app.get("/orders/{order_id}/pay", response_model=GetOrderSchema)
def pay_order(order_id: UUID):
    for order in ORDERS:
        if order["id"] == order_id:
            order["status"] = "progress"
            return order
    raise HTTPException(
        status_code=404,
        detail=f"Order with ID {order_id} not found"<>
    )
```


### Microservice Principles
微服务设计原则: 如何将系统拆分为微服务 *service decomposition*, 以及如何估计其质量
下面是三个设计原则:
- Database-per-service principle 服务独立数据库原则
- Loose coupling principle  松耦合原则
- Single Responsibility Principle (SRP) 单一职责原则

遵循这些原则将帮助你避免构建一个"分布式单体应用"(distributed monolith)的风险

#### Data-per-service principle
服务独立数据库原则是指, 每个微服务拥有一系列具体的数据, 并且其他微服务只能通过 API 访问.

这并不意味着每个微服务都要连接到不同的数据库中, 可以是关系数据库中的不同 tables, 或者非关系数据库中的 collections, 关键是数据被某个服务拥有, 不能被其他服务直接访问.

例如, 为了计算价格, orders service 从 Production database 中获取每个物品的价格, 它也需要知道用户是否有折扣优惠, 这个需要从 User database 获取. 然而, 不能直接诶访问这两个数据库, order service 需要从 products service 和 users service 获取数据.

#### Loose coupling principle
松耦合原则要求在设计服务的时候, 必须清晰的关注分离点, 松耦合的服务不依赖另一个服务的实现细节, 这项原则有两个实际的应用:
- 每个服务都可以独立于其他服务工作: 如果一个服务在不调用另一个服务的情况下无法完成一个简单的请求, 那么这两个服务之间没有清晰的关注点分离, 他们应被视为一个整体
- 每个服务都可以在不影响其他服务工作的情况下进行更新: 如果一个服务的更新需要其他服务, 那么这些服务之间存在紧密耦合, 需要重新设计

例如, 一个基于历史数据计算销售预测的服务(Sales Forecast Service), 以及一个拥有历史销售数据的服务(Historical Data Service), 为了计算预测, 销售服务会调用历史数据服务的API来获取历史数据. 在这种情况下, 销售预测服务在不调用历史数据服务的情况下无法响应任何请求, 因此两个服务之间存在紧密耦合.

解决方案是重新设计这两个服务, 使它们不相互依赖, 或者将它们合并成一个单一的服务.


#### Single responsibility principle
单一职责原则(SRP)指出, 我们要设计职责少、理想情况下只有一个职责的组件.
当应用于微服务设计架构时, 这意味着我们应努力围绕单一的业务能力或子域来设计服务.


### Decomposing micorservices by business capabilities
下面将 CoffeeMesh 系统根据业务内容分成以下部分

- 产品团队对应产品服务
- 原料团队对应原料服务
- 销售团队对应销售服务
- 金融团队对应金融服务
- 厨房团队对应厨房服务
- 配送团队对应配送服务

在上面的微服务架构中, 将不同的业务定义为一个微服务, 这样是为了方便, 但并不一定要这样实现

如上的设计规则, 满足了 SRP 原则, 每个模块都处理自己的数据, 但是这种设计并不满足松耦合原则(loose couping principle), 产品服务需要确定每款产品的库存,由于库存数据在原料服务中, 这就需要依赖原料服务, 而为每个产品都设计一个面向原料服务的 API 显然不太合理. 

因此, 这两个服务应该耦合在一起, 最终的服务结构如下:
- Products service: Products and Ingredients team
- Sales service: Sales team
- Kitchen service: Kitchen team
- Finance service: Finance team
- Delivery service: Delivery service


#### Service decomposition by subdomains
通过子领域分解是一种从 领域驱动设计(domain-driven desgin, DDD) 中汲取灵感的方法.
领域驱动设计是一种软件开发方法, 它专注于使用业务用户相同的语言来对业务流程和流向进行建模, 当应用于微服务设计时, DDD 能够帮助定义每个服务的核心职责和边界

对于 CoffeeMesh 项目, 我们希望根据下单的过程, 以及配送给客户的过程来建模, 将其分解为以下8步:
1. 当用户登陆网站后, 像用户展示产品列表. 每个产品都表示是否有库存. 用户可以根据是否有库存和价格来排序
2. 用户选择产品后下单
3. 用户为订单付费
4. 一但用户付费, 就将订单细节传递给 kitchen 服务
5. kitchen 服务根据订单制作咖啡
6. 用户可以查询订单进度
7. 一但订单制作完成, 就安排配送
8. 用户可以追踪无人机的配送进度, 直到配送到用户手中

根据上面步骤, 将模块划分为以下几个子领域 (subdomains)
- Production Subdomain 产品子领域  
    第一个服务用于 CoffeeMesh 产品目录的子域, 这个子域告诉用户哪些产品可用, 哪些不可用. 为此, 产品子域会追踪每种产品和原料的库存

- Orders Subdomain 订单子域  
    第二步代表一个允许用户选择产品的子域, 这个子域用于管理订单的声明周期. 该子域拥有用户订单的数据, 并提供一个接口来管理订单和检查其状态. 订单领域还负责第四步的第二部分: 在成功处理付款后, 将订单传递给厨房. 同时也满足了第六步的要求: 允许用户检查其订单状态. 作为订单管理者, 订单子域还会与配送子域协作来安排配送.

- Payments Subdomain 支付子域  
    第三步代表一个处理用户支付的子域. 该子域包括用户支付处理的专门逻辑, 包括银行卡验证, 与第三方支付提供商集成, 处理不同支付方式等. 支付子领域拥有与用户支付相关的数据.

- Kitchen Subdomain 厨房子域  
    第五步代表一个与厨房协作来管理客户订单生产的子域. 厨房的生产系统是全自动的, 厨房子域与厨房系统进行接口交互, 以安排客户订单的生产并追踪其进度.一旦订单生产完成, 厨房子域会通知订单子域, 后者随后安排配送. 厨房子域拥有与客户订单生产相关的数据, 并公开一个接口, 允许我们向厨房发送订单并跟踪其进度. 订单子域通过与厨房子域的接口交互, 来更新订单状态, 以满足第六步的需求.

- Delivery Subdomain 配送子域  
    第七步代表一个与自动化配送系统进行接口交互的子域. 该子域包含专门的逻辑, 用于解析客户的地理位置并计算到达他们的最佳路线. 它管理着配送无人机机队并优化配送, 同时拥有与所有配送相关的数据. 订单子域通过与配送子域的接口交互, 来更新客户订单的行程, 以满足第八步的需求.

通过以上分析, 将 CoffeeMesh 分解为5个子领域, 这些子领域可以被映射为微服务, 每个子领域都封装了定义明确, 职责清晰且拥有自己的逻辑区域. 领域驱动设计的微服务也满足了之前的微服务设计原则: 所有这些子域都可以在不依赖其他微服务的情况下执行其核心任务, 因此是松耦合的; 每个服务都拥有自己的数据, 因此符合服务独立数据库原则; 最后, 每个服务都在一个定义狭窄的子域内执行任务, 这符合单一职责原则.


### Wrapping Up
上面介绍了微服务的概念, 并通过一个 CoffeeMesh 的项目解释了如何设计一个微服务, 分别通过业务分解和通过子领域分解, 以及设计微服务的3条原则:
- Database-per-service principle 数据库独享原则
- Loose coupling principle 松耦合原则
- Single responsibility principle 单一责任原则
