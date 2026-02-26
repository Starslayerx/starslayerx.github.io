+++
date = '2025-08-20T8:00:00+08:00'
draft = false
title = 'HTTP Methods, Status Codes and Payloads'
categories = ['Note']
tags = ["HTTP"]
+++

本篇文章基于 REST api 介绍HTTP请求方法、HTTP响应码和API数据载荷, 是之前介绍 REST 那篇文章的延伸

## HTTP Status Codes

- 1xx group: Signals that an operation is in progress
- 2xx group: Signals that a request was successfully processed
- 3xx group: Signals that a resource has been moved to a new location
- 4xx group: Signals that someting was wrong with the request
- 5xx group: Signals that there was an error while processing the request

在之前文章中, 定义的 HTTP status code 如下:

- POST /orders: 201 (Created) - 资源成功创建
- GET /orders: 200 (OK) - 请求成功处理
- GET /orders/{order_id}: 200 (OK) - 请求成功处理
- PUT /orders/{order_id}: 200 (OK) - 资源成功更新
- DELETE /orders/{order_id}: 204 (No Content) - 请求被成功处理, 但是没有响应内容, 对比其他方法, DELETE 请求不需要 payload 来删除资源
- POST /orders/{order_id}/chanel: 200 (OK) - 取消成功, 由于并不创建任何资源, 故返回200
- POST /orders/{orders\id}/pay: 200 (OK) - 支付成功, 同样由于未创建资源, 返回200

上面全是成功的响应, 下面介绍错误响应

- 由于用户传入畸形的数据(malformed payload)或者请求一个不存在的 endpoint, 返回4xx响应码
- 服务器内部产生的错误, 这类错误使用5xx响应码

### Client errors in the request

这里将 malformed payload 分为两类:

1. payload with invalid syntax: 服务器无法解析或理解的数据, 例如json格式不对, 少了个反括号"}"之类的
2. unprocessable entities: 指却少要求属性的数据. 例如json里面要求name属性, 但是payload没有传这个属性; 又比如传入了一个不存在的资源, 这时返回一个404, 表示找不到相关资源

还有一种常见错误是, 发送了一个不支持的 HTTP 请求, 有两种 status code:

- 可以返回一个 501 (Not Implemented) 表示该方法目前还未支持, 但是未来会添加的功能
- 如果未来也不打算实现该方法, 则可以返回一个 405 (Method Not Allowed)

关于 身份验证(authentication) 和 授权(authorization) 相关的请求错误有以下两个:

- 对于未验证的请求, 返回 401 Unauthorized
- 对于已验证, 但是未授权的访问, 返回 403 Forbidden

### Server errors in the report

第2种错误是由于服务器代码 bug 或者基础设施限制导致的, 这时返回一个 500 (Internal Server Error)

另一种相关的错误是, 程序无法处理请求的问题, 通常使用 proxy server 或者 API gateway 来解决这个问题.
由于服务器过载或者下线维护的时候, 我们需要将当前情况告知用户.

- 当服务器无法处理新请求的时候, 必须返回 503 (Service Unavailable) 状态码, 表明服务器过载或者下线维护
- 当服务消耗太长时间返回响应, 应返回一个 504 (Gateway Timeout) 状态码

## Designing API Payloads

这部分介绍设计用户友好的 HTTP request / response payloads 的最佳实践.  
payloads 是指 client 和 server 之间传输的数据部分. API 的可用性往往依赖好的 payload 设计, 糟糕的设计会使得 API 的用户使用体验变差.

一个 HTTP request 包含了 URL, HTTP method 和 一系列的 headers 以及一个可选的 body(payload). HTTP headers 包含了请求的元数据, 例如 encoding format.  
类似的, HTTP response 包含一个 status code, 一协力的 headers 以及一个可选的 payload.  
可以使用不同的序列化方法来表示 payloads, 例如 XML 和 JSON. 在 REST APIs, 数据通常使用 JSON document.

HTTP 请求规范在 DELETE 和 GET 请求是否可以包含 payload 这一点上故意保持模糊, 其并未禁止使用 payload.
这使得一些 API 可以在 GET 请求中包含负载, 一个著名的例子是 Elasticsearch, 它允许客户端在 GET 请求的请求体中发送查询文档.

对于 HTTP Response 而言, 根据 status code 的不同, 可能会包含 payload. 根据 HTTP 规范(specification), 1xx, 204(No Content) 和 304(Not Modified) 这些状态码**不能**包含payload, 而其他的 response 都有.  
在 REST APIs 中, 最重要的就是 4xx 和 5xx 的错误响应, 以及 2xx 的成功响应和204的异常.

### HTTP payload designing patterns

错误的响应应该包含"error"关键字, 以及具体的细节信息, 并解释错误原因.  
例如, 一个 404 Response 返回的信息应该包含下面这些

```JSON
{
    "error": "Resource not found"
}
```

error 是比较常用的关键字, 当然你也可以使用 "detail" 和 "message" 这类关键字. 大多数的 Web 框架都有默认的错误模板, 例如 FastAPI 使用"detail".

对于成功响应的 HTTP Response 而言, 区分为3种类型: 创建资源, 更新资源 和 获取资源.

- Response Payloads for **POST** requests  
   使用 POST 请求来创建资源. 在 CoffeeMesh 的订单 API 中, 通过 POST /orders 端点来下单.
  为了创建一个订单, 需要将购买的商品列表发送给服务器, 服务器负责为该订单分配唯一的 ID, 因此订单的 ID 必须包含在响应数据中返回.
  服务器还会设置订单创建的时间以及初始状态. 这里将由服务器设置的属性称为 sever _sever-side_ 或 _read-only_, 这些属性也必须包含在响应数据中.
  最佳实践返回的响应是对 POST 方法的全面表示, 这个 payload 用于验证资源是否被正确创建.

- Response payloads for **PUT** and **PATCH** requests  
   要更新资源, 这里使用一个 PUT 或者 PATCH 请求. 对单个资源发送 PUT / PATCH 请求, 例如 CoffeeMesh 订单 API 中的 PUT /orders{order_id} 端点. 在这种情况下, 返回资源的完整表示也是一种良好实践, 客户端可以利用它验证更新是否已被正确处理.

- Response payloads for **GET** requests  
   使用 GET 方法检索资源. 例如, 在 CoffeeMesh 里面的订单 AIP 一样, 有两个 GET endpoints: GET /orders 和 GET /orders/{orders_id}.  
   GET /orders 返回一个列表, 有两种设计策略:  
   包含每个订单的完整信息或者包含每个订单的部分信息. 第一种方法在比较大的响应体中, 往往会导致 API 性能下降.第二种方法是包含所有订单的部分信息, 这种实践比较常见, 例如只返回每个订单的 ID 信息, 客户端需要使用 GET /orders/{orders_id} 来获取每个订单的具体信息.一般倾向于返回完整的信息, 尤其是公开发布的 APIs. 然而，如果是在开发一个内部的 API, 并且不需要详细的信息. 那么可以只提供客户端需要的信息. 更小的 payloads 处理起来更快, 能带来更好的用户体验. 但是对于单例 endpoint(GET /orders/{orders_id})应该总是返回完整的信息.

## Designing URL query parameters

一些 API 接口会返回一个资源列表, 当一个接口返回资源列表时, 最佳实践是允许用户对结果进行**筛选**和**分页**.
例如 GET /orders 接口, 可能希望结果为最近的5个订单, 或者只列出已取消的订单.
URL 查询参数能让我们实现这些目标, 他应当始终是**可选的**, 并且在适当的情况下, 服务器可以为其分配默认值.

定义: URL 查询参数是 URL 中的键值对参数. 查询参数位于问号(?)自后, 通常用于筛选接口的返回结果. 可以用与号(&)来分隔组合多个查询参数.

调用 GET /orders 接口并按"已取消"来筛选订单结果, 可以这样写

```
GET /orders?cancelled=true
```

### 链接多个参数

向 GET /orders 端点添加一个名为 limit 的查询参数以限制返回结果的数量. 如果要筛选"已取消"订单并将返回结果限制为 5 条, 可以这样请求 API

```
GET /orders?cancelled=true&limit=5
```

### 分页

允许 API 客户端对结果进行分页也是一种常见的做法. **分页**(**Pagination**)是指将结果切分成不同的集和, 并一次提供一个集和.
可以使用多种策略进行分页, 最常见的方法是使用 page 和 per_page 则两个参数的组合. page 代表数据的某个集和(页码), 而 per_page 则告诉每个集和中想要包含多少个项目. 服务器根据 per_page 的指来确定每一页返回多少条数据.  
在 API 中组合这两个参数, 如下所示:

```
GET /orders?page=1&per_page=10
```
