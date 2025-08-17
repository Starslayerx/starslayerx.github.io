+++
date = '2025-08-17T8:00:00+08:00'
draft = false
title = 'Designing and Building REST APIs'
tags = ["REST"]
+++
这篇文章延续之前微服务的内容, 将介绍关于 REST API 的以下几个方面:
- REST API 的设计原则
- Richardson maturity model (RMM) 如何帮助理解 REST 的优势和设计原则
- REST API 中资源(resource)和端点(endpoints)设计的概念
- 使用 HTTP 请求方法(verb)和HTTP状态码(status code)来创建高解释性的REST API
- 为REST API 设计高质量的载荷(payload)和URL查询参数(query parameters)

表达性状态转移 representational state transfer (REST) 描述了一种通过网络进行通信的应用程序架构风格.
最初, REST 的概念包含了一组用于设计分布式、可扩展 Web 应用的约束条件.
随着时间推移, 出现了更为细致的协议和规范, 为 REST API 的设计提供了明确的指导方针.
如今, REST 已经成为构建 Web API 的最流行选择.

下面将继续在 CoffeeMesh 项目上, 设计相关订单 API.

## What is REST?
REST 由 Roy Fielding 在他的博士论文 "Architectural Styles and the Design of Network-based Software Atchitecture" (PhD diss. University of California,Irvine,2000,p. 109) 中创造.
> 定义: REST 是一个松耦合和高伸缩的 API 架构风格. REST APIs 以资源为核心来组织, 这些资源是可以通过 API 操作的实体.

资源 resource 是可以通过唯一的 URL 来标识的实体, 有两种类型: 集和 collections 和 单体 singletons.
单体标识一个单独的实体, 而集和标识一组实体.

例如, 在 CoffeeMesh 的订单服务负责管理订单, 通过 `/orders/{order_id}` 访问特定订单, 是一个单体端点(singleton endpoint); 而所有订单通过 `/orders` 获取, 是一个集和端点 (collection endpoint).

某些资源还可以嵌套进其他资源中, 例如一个订单的 payload 中, 可能包含一个嵌套数组列出该订单的多个商品
, 例如下面这样:
```JSON
{
    "id": "924721eb-a1a1-4f13-b384-37e89c0e0875",
    "status": "progress",
    "created": "2023-09-01",
    "order": [
        {
            "product": "cappuccino",
            "size": "small",
            "quantity": 1
        },
        {
            "product": "croissant",
            "size": "medium",
            "quantity": 2
        }
    ]
}
```

可以创建一个嵌套端点来表示嵌套资源, 例如通过 GET `/orders/{order_id}/status` 端点查询订单的状态和细节信息. 
当资源对应的负载 payload 较大时, 使用嵌套端点是一种常见的优化策略, 例如只想知道状态信息, 就不需要查询大量的详细数据了, status 端点返回信息如下:
```JSON
{
  "status": "processing"
}
```

### Architectural constraints of REST applications
这里解释 REST 应用的架构约束, 这些约束由 Fielding 列出, 用于规定服务器应如何处理并响应客户端请求.
下面是每个约束的简单描述:
- Client-server architecture 客户端-服务器架构: 用户界面必须与后端解耦 decoupled
- Statelessness 无状态性: 服务器不能在请求之间维护状态
- Cacheability 可缓存性: 返回相同内容的请求, 应支持缓存
- Layered system 分层系统: API 按层架构, 但要向用户隐藏复杂性
- Code on demand 按需代码: 服务器可以按需将代码注入到用户界面
- Uniform interferace 统一接口: API 必须提供一致的接口来访问和操作资源

#### Separation of concers: The client-server architecture principle
关注点分离: 客户端-服务器架构原则

REST 依赖于关注点分离原则, 因此要求用户界面(UI)必须于数据存储和服务器逻辑解耦.
这样一来, 服务器组件就可以独立于 UI 元素进行开发.
一种常见的实现方法是: 将 UI 构建为一个独立应用, 例如单页应用(SPA)

#### Make it scalable: The statelessness principle
可扩展性: 无状态原则

在 REST 中, 每一次对服务器的请求都必须包含处理该请求的全部信息.
特别是, 服务器不能在请求之间保持状态. 将状态管理从服务器组件中移除, 可以更容易地对后端进行水平扩展, 这使得我们能够部署多个服务器示例, 并且由于这些实例都不管理 API 客户端的状态, 客户端就可以于任意一个实例进行通信.

#### Optimize for performance: The cacheability principle
性能优化: 可缓存原则

在适用的情况下, 服务器必须是可缓存的. 缓存提升 API 的性能, 这意味着不必一次又一次地执行生成响应所需要的计算. GET 请求适合缓存, 因为他们返回的是服务器中已保存的数据. 通过缓存 GET 请求, 可以避免在用户每次请求相同信息时都从数据源重新获取数据, 生成 GET 请求响应所需要的时间越长, 缓存所带来的收益就越大.

#### Make it simple for the client: The layered system principle
让客户端更简单: 分层系统原则

在 REST 架构中, 客户端必须通过一个唯一的入口访问 API, 而且不应该知道自己是直接连接到最终服务器, 还是连接到某个中间层(例如负载均衡器). 可以把服务端的应用的不同组件部署在不同的服务器上, 或者把相同的组件部署在多个服务器上, 以实现冗余和壳扩展性. 但这些复杂性必须对用户隐藏, 只暴露一个统一的入口来封装服务访问.

#### Extendable interferaces: The code-on-demand principle
可扩展接口: 按需代码原则

服务器可以通过直接从后端发送可执行代码, 来扩展客户端应用的功能. 这个约束是可选的, 只适用于后端提供客户端界面的应用.

#### Keep it consistent: The uniform interface principle
保持统一性: 统一接口原则

REST 应用必须向其使用者提供统一且一直的接口, 接口必须有文档说明, 服务器和客户端必须严格遵循 API 规范. 每个资源通过统一资源标识符(URI)来标识, 每个 URI 必须唯一, 并且始终返回相同的资源. 资源必须使用某种序列化方法表示, 并且这种方法在整个 API 中应保持一致. 如今, REST API 通常使用 JSON 作为序列化格式, 但也可以使用其他格式, 例如 XML.


## Hypermedia as the engine of application state
超媒介作为应用状态引擎 (HATEOAS)

在 2008 年发表的一篇题为[REST APIs Must Be Hypertext-Driven](https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)的文章中, Fielding 提出:
> REST API 的响应必须包含相关链接, 以便客户端可以通过这些链接导航 API

HATEOAS 是 REST API 设计中的一种范式(paradigm), 它强调可发现性. 每当客户端向服务器请求某个资源时, 响应必须包含指向该资源的相关链接列表. 例如, 客户端请求订单详情时, 响应必须包含取消订单和支付订单的相关链接.

例如下面这样
```JSON
{
    "id": 8,
    "status": "progress",
    "created": "2025-8-16",
    "order": [
        {
            "product": "cappuccino",
            "size": "small",
            "quantity": 1
        },
        {
            "product": "croissant",
            "size": "medium",
            "quantity": 2
        }
    ],
    "links": [
        {
            "href": "/orders/8/cancel",
            "description": "Cancels the order",
            "type": "POST"
 
        },
        {
            "href": "/orders/8/pay",
            "description": "Pays for the order",
            "type": "POST"
        }
    ]
}
```

提供关联链接可以使 API 具有可导航性, 更易于使用, 因为每个资源都会附带与之交互所需的所有 URL.
然而在实际中, 许多 API 并没有这样实现, 原因包括:
1. 超链接提供的信息在 API 文档中已经可用
    - 实际上, OpenAPI 规范中包含的信息比单独为特定资源提供的相关链接列表要丰富和结构化得多
2. 不总是清楚应该返回哪些链接
    - 不同用户拥有不同的权限和角色, 可以执行不同操作和访问不同资源
    - 例如，CoffeeMesh API 的外部用户可以使用 `POST /orders` 下单, 也可以使用 `GET /orders/{order_id}` 查询订单详情, 但不能使用 `DELETE /orders/{order_id}` 删除订单, 因为该接口仅限内部用户
    - 如果 HATEOAS 的目标是让 API 可以从单一入口导航, 那么向外部用户返回他们无法使用的 DELETE 链接显然没有意义
    - 因此, 需要根据用户权限返回不同的相关链接列表, 但这会增加 API 设计和实现的复杂性, 并将授权层与 API 层耦合
3. 资源状态可能限制某些操作
    - 例如, `POST /orders/1234/cancel` 只能在活跃订单上调用, 而无法对已取消订单调用
    - 这种不确定性会增加遵循 HATEOAS 原则的接口设计和实现难度
4. 响应负载可能过大
    - 在一些 API 中, 相关链接列表可能非常庞大, 使响应体变大, 从而影响 API 性能, 以及对网络连接较差的小设备的可靠性

在设计自己的 API 时, 可以根据实际情况决定是否遵循 HATEOAS 原则, 在某些情况下是有用的, 例如:

在 Wiki 应用中, 响应中的 "linked resources" 部分可以列出
- 与某篇文章相关的内容
- 该文章的多语言版本链接
- 可以对文章执行的操作链接

总体来说, 需要在 API 文档已经清晰详细提供信息 与 通过响应辅助客户端交互 之间找到平衡
- 面向公众的 API: 客户端会从关联链接中受益
- 小型内部 API: 通常不需要提供关联链接


## Analyzing the matruity of an API with the Richardson maturity model
使用 Richardson 成熟度模型分析 API 的成熟度

这是由 Leonard Richardson 提出的一种思维模型, 用于帮助评估一个 API 在多大程度上遵循了 REST 原则, Richardson 成熟度模型将 API 的"成熟度"划分为四个等级: 
0. Level 0: RPC over HTTP
1. Level 1: Resources
2. Level 2: HTTP methods and status codes
3. Level 3: Hypermedia controls (HATEOAS)
- Glory of REST!

### Level 0: Web APIs à la RPC
类似 PRC 的 Web API

在0级中, HTTP 本质上只作为一种传输系统, 用于承载与服务器的交互. 在这种情况下, API 的概念更接近 远程过程调用(remote procedure call, RPC). 所有服务器的请求都在同一个端点发起, 并使用相同的 HTTP 方法, 客户端请求的具体细节通过 HTTP 的 payload 传递.

例如, 在 CoffeeMesh 网站下单时, 客户端可能会向通用 `/api` 端点发送如下 POST 请求:
```JSON
{
  "action": "placeOrder",
  "order": [
      {
          "product": "mocha",
          "size": "medium",
          "quantity": 2
      }
  ]
}
```
服务器通常会返回200状态码, 并附带一个 payload, 告诉请求处理结果.

类似地, 要获取某个订单的详情, 客户端也可能向通用 `/api` 端点发送如下 POST 请求:
```JSON
{
  "action": "getOrder",
  "order": [
      {
          "id": 8
      }
  ]
}
```


### Level 1: Intorducing the concept of resource
第1级引入了资源 URL 的概念, 服务器不再使用通用的 `/api` 端点, 而是暴露表示资源的 URL. 例如:

- `/orders` 表示订单集和
- `/order/{order_id}` 表示单个订单

要下单时, 客户端向 `/orders` 端点发送 POST 请求, payload 与 0 级类似
```JSON
{
  "action": "placeOrder",
  "order": [
      {
          "product": "mocha",
          "size": "medium",
          "quantity": 2
      }
  ]
}
```
在这一层, API 还没有使用不同的 HTTP 方法来区分不同操作


### Level 2: Using HTTP methods and status codes
第2级引入了 HTTP 请求方法verbs 和 状态码status 的概念, 这一层, HTTP verbs 用于表示具体操作.
例如, 要下订单, 客户端向 `/orders` 端点发送一个 POST 请求, 内容如下:
```Python
{
  "order": [
      {
          "product": "mocha",
          "size": "medium",
          "quantity": 2
      }
  ]
}
```

在这个例子中, HTTP 方法 POST 表示要执行的操作, 而请求体仅包含想要下的订单的具体信息

类似地, 如果要获取某个订单的详细信息, 我们会向该订单的 URI 发送 GET 请求: `/orders/{order_id}`. 这里使用 GET 告诉服务器, 希望获取 URI 指定资源的详细信息

前几个级别的响应通常都使用相同的状态码(通常为 200), 而第二级引入了 HTTP 状态码的语义化使用, 用来报告客户端请求处理的结果. 例如:
- 使用 POST 创建资源时, 服务器会返回 201 Created 状态码
- 请求不存在的资源时, 会返回 404 Not Found 状态码


### Level 3: API discoverability
第3级引入了可发现性的概念, 通过 HATEOAS 原则, 并在响应中添加表示可对资源执行操作的链接来实现.

例如, 对 `/orders/{order_id}` 端点发送 GET 请求, 会返回该订单的表示(representation), 并包含一系列相关链接
```Python
{
  "id": 8,
  "status": "progress",
  "created": "2023-09-01",
  "order": [
      {
          "product": "cappuccino",
          "size": "small",
          "quantity": 1
      },
      {
          "product": "croissant",
          "size": "medium",
          "quantity": 2
      }
  ],
  "links": [
      {
          "href": "/orders/8/cancel",
          "description": "Cancels the order",
          "type": "POST"
      },
      {
          "href": "/orders/8/pay",
          "description": "Pays for the order",
          "type": "GET"
      }
  ]
}
```
在 Richardson 成熟度模型中, 第三级代表了他所称的 "REST 的荣耀(Glory of REST)" 的最后一步

该模型为我们提供了一个框架, 用来思考 API 设计在 REST 原则体系中的位置. 它的目的不是衡量 API 在多大程度上"符合"REST 原则, 也不是评估 API 设计的质量; 而是帮助我们思考如何充分利用 HTTP 协议, 创建表达力强、易理解、易使用的 API.
