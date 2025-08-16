+++
date = '2025-08-17T8:00:00+08:00'
draft = true
title = 'Designing and Building REST APIs'
+++
这篇文章延续之前微服务的内容, 将介绍关于 REST API 的以下几个方面:
- REST API 的设计原则
- Richardson maturity model (RMM) 如何帮助理解 REST 的优势和设计原则
- REST API 中资源(resource)和端点(endpoints)设计的概念
- 使用 HTTP 请求方法(verb)和HTTP状态码(status code)来创建高解释性的REST API
- 为REST API 设计高质量的载荷(payload)和URL查询参数(query parameters)

表达性状态转移 Representational state transfer (REST) 描述了一种通过网络进行通信的应用程序架构风格.
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























































