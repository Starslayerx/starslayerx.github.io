+++
date = '2025-08-02T10:00:00+08:00'
draft = true
title = 'Microservice APIs'
+++

### What are microservices ?
什么是 microservices? 微服务可以有多种不同的定义方式, 具体取决于希望强调微服务架构的哪个方面, 不同作者会给出略有不同但相关的定义

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



### A basic API implementation
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

