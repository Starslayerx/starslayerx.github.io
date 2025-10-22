+++
date = '2025-08-25T8:00:00+08:00'
draft = false
title = 'Documenting REST APIs with OpenAPI'
tags = ['REST', 'OpenAPI']
+++

本章介绍如何使用 OpenAPI 来为 API 编写文档. OpenAPI 是描述 RESTful API 最流行的标准, 拥有丰富的生态系统, 可以用于测试、验证和可视化 API. 大多数编程语言都支持 OpenAPI 规范的库.

OpenAPI 使用 JSON Schema 来描述 API 的结构和模型, 因此首先介绍 JSON Schema 的工作原理. JSON Schema 是一种用于定义 JSON 文档结构的规范, 包括文档中值的类型和格式.

### Using JSON Schema to model data

**使用 JSON Schema 对数据建模**

JSON Schema 是一种规范标准, 用于定义 JSON 文档的结构及其属性的类型和格式. JSON Schema 规范通常定义一个具有特定属性或特性的对象, 由键值对的关联数组表示, 如下面这样:

```JSON
{
    "status": {
        "type": "string"
    }
}
```

1. 在 JSON Schema 规范中, 每个属性都以键值对的形式出现, 其中值是该属性的描述符
2. 一个属性最基本的描述符就是 `type`, 上面例子中, 指定类型为字符串

JSON Schema 支持以下基本数据类型:

- `string`: 字符串
- `number`: 整数和十进制数
- `object`: 关联数组 (类似py中的字典)
- `array`: 其他数据类型的集合
- `boolean`: 真或假
- `null`: 未初始化的数据

定义一个 object 的例子

```JSON
{
    "order": {
        "type": "object",
        "properties": {
            "product": {
                "type": "string"
            },
            "size": {
                "type": "string"
            },
            "quantity": {
                "type": "intger"
            }
        }
    }
}
```

由于 order 是一个 object, 故 order 属性有 properties. 每个 property 都有自己的类型.  
一个符合规则的 JSON 文档例子如下

```JSON
{
    "order": {
        "product": "coffee",
        "size": "big",
        "quantity": 1,
    }
}
```

属性 property 也可以代表一个项目的数组.  
order 对象代表一个对象的数组, 使用 items 关键字来定义数组中的元素.

```JSON
{
    "order": {
        "type": "array",
        "items": {
            "type": "object",
            "properties": {
                "product": {
                    "type": "string"
                },
                "size": {
                    "type": "string"
                },
                "quantity": {
                    "type": "integer"
                }
            }
        }
    }
}
```

在上面例子中, order 属性是一个数组 array. 数组类型需要在模式 schema 中有一个额外的属性, 那就是 items 属性, 其定义了数组中包含的每个元素的类型. 这种情况下, 数组中的每个元素都是一个对象, 代表订单中的一个项目.

一个对象可以包含任意数量的嵌套对象, 但是, 嵌套太多时, 缩进会变得很大, 导致规范难以阅读.  
为了避免这个问题, JSON Schema 允许单独定义每个对象, 并使用 JSON 指针 (JSON pointers) 来引用它们.  
JSON pointers 是一种特殊语法, 运行向统一分规范中的另一个对象定义.

如下面代码, 可以将 order 数组中的每个项的定义提取为一个名为 `OrderItemSchema` 的模型. 然后使用一个 JSON 指针和特殊的 `$ref` 关键字来引用 `OrderItemSchema`

```JSON
{
    "OrderItemSchema": {
        "type": "object",
        "properties": {
            "product": {
                "type": "string"
            },
            "size": {
                "type": "string"
            },
            "quantity": {
                "type": "integer"
            }
        }
    },
    "Order": {
        "status": {
            "type": "string"
        },
        "order": {
            "type": "array",
            "items": {
                "$ref": '#/OrderItemSchema'
            }
        }
    }
}
```

JSON 指针使用特殊关键字 `$ref` 和 `JSONPath` 语法来指向 schema 中的另一个定义.  
在 `JSONPath` 的语法中, 文档的根(root)使用井号 `#` 表示, 嵌套属性的关系由斜线(slashes) `/` 表示.
例如, 如果响应创建 `#/OrderItemSchema` 模型的 `size` 属性的指针, 我们会使用如下的语法 `#/OrderItemSchema/size`.  
通过将通用的模式对象提取成壳重用的模型, 并使用 JSON 指针来引用他们, 从而对规范进行重构, 这有助于避免重复, 并报慈整洁和简洁.

除了指定类型之外, JSON Schema 还允许指定属性的格式(foramt), 可以自定义格式, 也可以使用 JSON Schema 内置的格式.  
例如, 一个代表日期的属性, 可以使用 `data` 格式, 这是 JSON Schema 支持的内置格式, 代表一个 ISO 日期(如2025-05-21)

```
{
    "created": {
        "type": "string",
        "format": "date"
    }
}
```

除了使用 JSON, JSON Schema 实际上还可以使用 YAML 来编写, 这种格式更加常见且更容易理解. OpenAPI 规范也通常以 YAML 格式提供, 后面部分使用 YAML 格式编写.

### Anatomy of an OpenAPI specification

**剖析 OpenAPI 规范**

OpenAPI 是一种用于文档化 Restful API 的标准规范格式. 它允许我们详细描述 API 的每一个元素, 包括其端点 endpoints、请求响应和有效载荷(payloads)的格式、安全方案(security schemes)等等. OpenAPI 最初于2010年以 Swagger 的名称创建, 2015年 Linux 基金会和主要公司的联盟共同赞助成立了 OpenAPI Initiative, 是一个旨在改进构建 RESTful API 的协议和标准的项目. 如今, OpenAPI 是迄今为止用于文档化 RESTful API 最流行的规范格式, 它拥有丰富的生态系统工具, 可用于 API 可视化、测试和验证.

OpenAPI 包含了和 API 交互所需要的一切信息, 一份 OpenAPI 规范由5个部分构成:

- `openapi`: 指明版本
- `info`: API 的基本信息, 例如标题和版本
- `servers`: 包含 API 可用的 URL 列表. 可以列出用于不同环境的多个 URL.
- `paths`: 描述 API 公开的端点, 包括预期的有效载荷(payloads)、允许的参数以及响应的格式. 这是规范中最重要的部分, 因为它代表了 API 的接口, 也是使用者为了学习如何与 API 集成会查看的部分.
- `components`: 定义了在整个规范中被引用的可重复元素. 例如模式(schemas)、参数、安全方案、请求体和响应等. 模式是对请求和响应中预期属性和类型的定义. OpenAPI 模式是使用 JSON Schema 语法定义的.

### Documenting the API endpoints

**文档化 API 端点**

OpenAPI 的 `path` 部分描述了 API 接口, 它列出了 API 公开的 URL 路径, 以及实现的 HTTP 方法, 预期的请求类型和返回的响应.  
每个路径都是一个对象, 其属性为它支持的 HTTP 方法, 这里将说明 URL 路径和 HTTP 方法的文档化.  
在之前定义了如下端点:

- `POST /orders`: 请求订单. 需要订单的细节信息.
- `GET /orders`: 返回订单列表. 接受 URL 查询参数, 并允许过滤结果.
- `GET /orders/{order_id}`: 返回订单细节信息
- `PUT /orders/{order_id}`: 更新订单细节信息, 由于这是一个 PUT 端点, 要求订单的全面信息.
- `DELETE /orders/{order_id}`: 删除订单
- `POST /orders/{order_id}/pay`: 为订单付款
- `POST /orders/{order_id}/cancel`: 取消订单

下面是 API 订单的高层定义, 声明了 URL 和每个 URL 所实现的 HTTP 方法, 并为每个端点添加了一个操作ID(operation ID), 以便在文档其他部分引用:

```yaml
paths:
  /orders:
    get:
      operationId: createOrder

  /orders/{order_id}:
    get:
      operationId: getOrder
    put:
      operationId: updateOrder
    delete:
      opertaionId: deleteOrder

  /orders/{order_id}/pay:
    post:
      operationId: payOrder

  /orders/{order_id}/cancel:
    post:
      operationId: cancelOrder
```

现在有了端点, 还需要填充其中的细节.

- 对于 `GET /orders` 端点, 需要描述接受它的参数
- 对于 `POST` 和 `PUT` 端点, 需要描述请求的有效载荷 payloads  
  此外, 还需要为每个端点描述其响应

### Documenting URL query parameters

**文档化 URL 查询参数**

URL query parameter 允许我们过滤和排序 GET endpoint 的结果. 在本章中, 将使用 OpenAPI 定义 URL query parameters. GET /orders endpoint 允许我们使用下面的参数过滤订单:

- `cancelled`: 订单是否被取消, 类型 boolean
- `limit`: 表示返回给用户的订单的最大数量

合并起来使用大概下面这样:

```
GET /orders?cancelled=true&limit=5
```

这个请求向服务器请求一个 5条已经取消 的订单的列表.

```
paths:
  /orders:
    get:
      parameters:
        - name: cancelled
          in: query
          required: false
          schema:
            type: boolean
        - name: limit
          in: query
          required: false
          schema:
            type: integer
```

定义一个参数需要一个名称 name, 这个名称就是实际 URL 中用来指引它的值. 还需要指定参数的类型, 在 OpenAPI 3.1 区分了四种类型参数: 路径参数(path parameters)、查询参数(query parameters)、头部参数(header parameters)和Cookie 参数(cookie parameters).

- 头部参数是在 HTTP 头部字段中的参数, 而 Cookie 参数则放在 Cookie 有效载荷中.
- 路径参数是 URL 路径的一部分, 通常用于标识一个资源.
- 查询参数是可选参数, 允许对端点的结果进行过滤和排序.  
  使用 `schema` 关键字参数来定义参数的类型, 并且在相关时, 也会指定参数的格式.

### Documenting request payloads

**文档化请求载荷**

一个请求代表 client 通过 POST 或 PUT 方法向 server 发送的数据. 这节介绍 API endpoints 的 request payloads.  
例如 `POST /orders` 方法:

```JSON
{
    "order": [
        {
            "product": "cappuccinio",
            "size": "big",
            "quantity": 1
        }
    ]
}
```

这个 payload 包含一个 order 属性, 代表了一系列的物品. 每个物品被定义为下面的3个属性和约束:

- product: 用户订购的产品类型
- size: 产品的大小. 有3种选择: small, medium 和 big
- quantity: 产品的数量. 可以是任何大于等于1的整数

下面展示如何为这个有效载荷 payload 定义模式

```yaml
paths:
  /orders:
    post:
      operationId: createOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                order:
                  type: array
                  items:
                    type: object
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
                        type: intger
                        required: false
                        default: 1
                    required:
                      - product
                      - size
```

通过 HTTP 方法的 `requestBody` 属性下的 `content` 属性来定义 payload, 并可以指定不同的有效载荷. 有效载荷可以指定为不同的格式, 在本例中, 只允许 JSON 格式数据, 媒体类型为 `application/json`.  
这里的有效载荷的模式是一个对象, 有一个属性 `order`, 其类型为数组, 数组中的元素的对象, 包含3个属性: `product`(类型为字符串), `size`(类型为字符串) 和 `quantity`(类型为整数).  
此外, 还为 `size` 属性定义了一个枚举(enumeration), 将可接受的值限制为 `samll`、`medium` 和 `big` 3种.  
最后, 还为 `quantity` 属性提供了默认值 `1`, 因为它是有效载荷中唯一非必须的字段.

### Refactoring schema definitions to avoid repetition

**重构 schema 定义从而避免重复**

在本节将介绍重构模式 refactoring schemas 的策略, 以保持 API 规范的整洁和可读性.  
上面的 `POST /orders` 端点定义很长, 包含多层缩进, 难以阅读, 意味着会变得难以扩展和维护.  
可以将有效载荷 payload 的模式移动到 API 规范的 `components` 部分. 该部分用于声明在整个规范中被引用的模式, 每个模式都是一个对象, 其中键是模式的名称, 而值是属性的对象.

```yaml
paths:
  /orders:
    post:
      operatoinId: createOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderSchema'    ①

components:   ②
  schemas:    ③
    CreateOrderSchema:
      type: object
      properties:
        type: array
        items:
          type: object
          properties:
            product:
              type: string
            size:
              type: string
              enum:
                - samll
                - medium
                - big
            quantity:
              type: intger
              required: false
              default: 1
        required:
          - product
          - size
```

1. 使用 JSON pointer 指向文档其他位置
2. schema 定义在 components 下面
3. 每个 schema 都是一个对象, 其中 key(CreateOrderSchema) 是名称, values(CreateOrderSchema下面的所有内容) 是描述属性 properties

将 `POST /orders` 请求有效载荷的模式移动到 API 的 `components` 部分, 能使文档更具可读性. 这样得以保持 `path` 部分的简洁, 并专注于端点的高层细节. 只需要使用一个 JSON 指针来引用 `CreateOrderSchema` 模式:

```
#/components/schemas/CreateOrderSchema
```

这份规范现在已经不错了, 但是可以更好. `CreateOrderSchema` 有些长, 并且包含了多层嵌套定义. 如果 `CreateOrderSchema` 的复杂性随着时间增长, 将越来越难以维护. 可以通过下面方式重构数组中订单项的定义, 使其更加具有可读性, 这个策略运行 API 的其他部分重用订单项的模式.

```yaml
components:
  schemas:
      OrderItemSchema:                     ①
        type: object
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
      CreateOrderSchema:
        type: object
        properties:
          order:
            type: array
            items:
              $ref: '#/OrderItemSchema'    ②
```

1. `OrderItemSchema`: 订单中的项
2. `CreateOrderSchema`: 使用一个 JSON pointer 指向 `OrderItemSchema`

现在 schemas 看起来就好多了, 并且可以在 `/POST /orders/{order_id}` 端点中重用它.  
`/orders/{order_id}` 代表一个单例资源(singleton resource), 因此 URL 包含一个路径参数, 即订单ID. 在 OpenAPI 中, 路径参数使用大括号`{}` 表示.

```
paths:
  /orders:
    get:
      ...

  /orders/{order_id}:          ①
    parameters:                ②
      - in: path               ③
        name: order_id         ④
        required: true         ⑤
        schema:
          type: string
          format: uuid         ⑥
    put:                       ⑦
      operationId: updateOrder
      requestBody:             ⑧
        required: true
        content:
          application/json:
            schema:
           $ref: '#/components/schemas/CreateOrderSchema
```

1. 定义订单的资源地址
2. 定义 URL path parameter
3. `order_id` 参数是 URL 路径的一部分
4. 参数名称
5. 必填参数
6. 具体参数格式(UUID)
7. 为当前 URL 定义 PUT 方法
8. 文档化 request body 的 PUT 端点

### Documenting API responses

**文档化响应体**

`GET /orders/{order_id}` 端点的响应类似下面这样:

```JSON
{
    "id": "924721eb-a1a1-4f13-b384-37e89c0e0875",
    "status": "progress",
    "created": "2022-05-01",
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

这个有效载荷展示了用户订购的产品, 下单时间以及订单状态. 类似之前 `POST` 和 `PUT` 端点定义的请求有效载荷, 因此可以重用之前的模式.

```yaml
components:
  schemas:
    OrderItemSchema:
      ...

GetOrderSchema:
①
  type: object
  properties:
    status:
      type: string
      enum:
      ②
        - created
        - paid
        - progress
        - cancelled
        - displatched
        - delivered
    created:
      type: string
      format: date-time
      ③
      order:
        type: array
        items:
          $ref: '#/components/schemas/OrderItemSchema'  ④
```

1. 定义 `GetOrderSchema` 模式
2. 使用枚举限制状态属性
3. 日期格式的字符串
4. 使用 JSON pointer 引用 `OrderItemSchema`

在上面的清单中, 使用一个 JSON 指针指向 `GetOrderSchema`. 另一种重用现有模式的方法是继承.

在 OpenAPI 中, 可以通过一种称为模式组合(model composition) 的策略来继承和扩展一个模式, 该策略允许将不同模式的属性组合到一个单一的对象定义中. 在这种情况下, 使用特殊关键词 `allOf` 来表示该对象需要包含列出的所有模式中的属性.

```yaml
components:
  schemas:
    OrderItemSchema:
      ...

    GetOrderSchema:
      allOf:                                                ①
        - $ref: '#/components/schemas/CreateOrderSchema'    ②
        - type: object                                      ③
          properties:
            status:
              type: string
              enum:
                - created
                - paid
                - progress
                - cancelled
                - dispatched
                - delivered
            created:
              type: string
              format: date-time
```

1. 使用 `allOf` 关键字继承其他 schemas 的属性
2. 使用 JSON pointer 引用其他的 schema
3. 使用一个新对象 `GetOrderSchema` 来包含特有的属性

**模型组合(Model composition)** 能使规范更简洁、更紧凑, 但它只在模式严格兼容的情况才有效.  
如果决定使用新的属性来扩展 `CreateOrderSchema`, 那么这个模式可能就不再能用于 `GetOrderSchema` 模型.  
从这个意义上讲, 有时候更好的做饭是寻找不同模式中的共同元素, 将其定义重构为独立的模式.

现在有了 `GET /orders/{order_id}` 端点响应有效载荷的模式, 就可以完善该端点的规范了. 把端点的响应定义为对象, 其中键是响应的状态码, 并描述响应的内容类型及其模式.

```yaml
paths:
  /orders:
    get:
        ...

  /orders/{order_id}:
    parameters:
      - in: path
        name: order_id
        required: true
        schema:
          type: string
          format: uuid

    put:
      ...

    get:                                                      ①
      summary: Returns the details of a specific order        ②
      operationId: getOrder
      responses:                                              ③
        '200':                                                ④
          description: OK                                     ⑤
          content:                                            ⑥
            application/json:
              schema:
                $ref: '#/components/schemas/GetOrderSchema'   ⑦
```

1. 定义 `GET /order/{order_id}` endpoint
2. 为该端点提供一个描述
3. 定义一个端点响应
4. 每个响应都是一个对象, 其中 key 为状态码
5. 响应的简单描述
6. 描述响应的内容类型
7. 使用 JSON pointer 引用 `GetOrderSchema`

根据上面内容可以看到, 在端点的 `responses` 部分定义了响应模式(schemas), 在这种情况下, 值提供了 `200 (OK)` 成功响应的规范, 但也可以为其他状态码编写文档.

### Creating generic responses

**创建同样响应**

本节介绍如何为 API 端点添加错误响应. 错误响应更具通用性, 因此可以使用 API 规范的 `components` 部分来提供这些响应的通用定义, 然后在端点中使用他们.  
这里在 API 的 `components` 部分的 `responses` 标头下定义通用响应. 下面展示了一个名为 `NotFound` 的 404 响应通用定义. 与任何其他响应意义, 也会为其有效载荷编写文档, 本例中有效载荷由 `Error` 模式定义.

```
components:
  responses:                                                ①
    NotFound:                                               ②
      description: The specified resource was not found.    ③
      content:                                              ④
        application/json:
          schema:
            $ref: '#/components/schemas/Error'              ⑤

  schemas:
    OrderItemSchema:
      ...
    Error:                                                  ⑥
      type: object
      properties:
        detail:
          type: string
      required:
        - detail
```

1. 通用响应定义在 `components` 部分的 `responses` 下
2. 为这个响应命名
3. 描述这个响应
4. 定义响应内容
5. 引用 `Error` 模式
6. 定义 `Error` 有效载荷的模式

上面这份针对 404 响应的规范可以在 `/orders/{order_id}` URL 路径下的所有端点规范中重复使用, 因为所有这些端点都是专门设计来针对特定资源的.

> 在 OpenAPI 的 [GitHub](https://github.com/OAI/OpenAPI-Specification/issues/521) 仓库中, 有一个请求是希望允许在 URL 路径下直接包含通用响应, 但目前尚未实现

下面定义 `/orders/{order_id}` 的 404 响应模式

```yaml
paths:
  ...

  /orders/{order_id}:
    parameters:
      - in: path
        name: order_id
        required: true
        schema:
          type: string
          "format": uuid
    get:
      summary: Returns the details of a specific order
      operationId: getOrder
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/GetOrderSchema'
        '404':                ①
          $ref: '#/components/responses/NotFound'  ②
```

1. 定义一个 404 响应
2. 使用 JSON 指针引用 `NotFound` 响应

剩下的一个端点是 `GET /orders`, 它返回一个订单列表, 该端点的有效载荷重用了 `GetOrderSchema` 来定义订单数组中的项目

```yaml
paths:
  /orders:
    get:                               ①
      operationId: getOrders
      responses:
        '200':
          description: A JSON array of orders
          content:
            application/json:
              schema:
                type: object
                properties:
                  orders:
                    type: array        ②
                    items:
                      $ref: '#/components/schemas/GetOrderSchema'  ③
                required:
                  - order
    post:
      ...

  /orders/{order_id}:
    parameters:
      ...
```

1. 定义 `/orders` URL 路径的新 `GET` 方法
2. `orders` 是一个数组
3. 数组的每个项目都由 `GetOrderSchema` 定义

现在, API 的端点已完全文档化. 可以在端点定义中使用更多的元素, 例如 `tags` 和 `externalDocs`. 些属性并非绝对必要, 但可以帮助为 API 提供更多结构, 或使其更易于对端点进行分组.

### Defining the authentication scheme of the API

**定义 API 的认证模式**

如果 API 受到到保护, API 规范必须描述用户如何进行身份认证和授权请求. API 安全定义位于规范的 `components` 部分, 在 `securitySchema` 标头下.

通过 OpenAPI, 可以描述不同的安全方案, 例如基于 HTTP 的认证、基于密钥的认证、OAuth2 开放授权 和 OpenID Connect.  
下面描述了3种方案: 一种用于 OpenID Connect, 一种用于 OAuth2, 还有一种用于 Bearer 授权.

- 这里使用 OpenID Connect 通过前端应用来授权用户访问  
   对于 OpenID Connect, 必须在 `openIdConnectUrl` 属性下提供一个配置 URL, 该 URL 描述了后端客户端如何认证工作
- 对于直接的 API 集成, 提供 OAuth 的客户端凭证流(client credentials flow)  
   对于 OAuth2, 必须描述可用的授权流(authentication flows), 以及客户端必须用于获取其授权令牌的 URL 和可用的作用域(scopes).  
  Bearer 授权告诉用户, 他们必须在 `Authorization` 头部中包含一个 JSON Web Token(JWT) 来授权其请求.

```yaml
components:
  responses:
    ...
  schemas:
    ...

  securitySchemes: ①
    openId: ②
      type: openIdConnect ③
      openIdConnectUrl: https://coffeemesh-dev.eu.auth0.com/.well-known/openid-configuration ④
    oauth2: ⑤
      type: oauth2 ⑥
      flows: ⑦
        clientCredentials: ⑧
          tokenUrl: https://coffeemesh-dev.eu.auth0.com/oauth/token ⑨
          scopes: {} ⑩
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT ⑪
...
security:
  - oauth2:
      - getOrders
      - createOrder
      - getOrder
      - updateOrder
      - deleteOrder
      - payOrder
      - cancelOrder
  - bearerAuth:
      - getOrders
      - createOrder
      - getOrder
      - updateOrder
      - deleteOrder
      - payOrder
      - - cancelOrder
```

1.  API components 部分的 securitySchemes 标头下的安全方案
2.  为安全方案提供一个名称(可以是任何名称)
3.  安全方案的类型
4.  描述后端 OpenID Connect 配置的 URL
5.  另一个安全方案的名称
6.  该安全方案的类型
7.  该安全方案下可用的授权流
8.  客户端凭证流的描述
9.  用户可以请求授权令牌的 URL
10. 请求授权令牌时可用的作用域
11. Bearer 令牌的格式是 JSON Web Token (JWT)

## Wrapping Up

- JSON Schema 是一个定义 JSON 文档中属性类型和格式的规范, 它有助于以一种独立于编程语言的方式定义数据验证模型.
- OpenAPI 是一种用于描述 RESTful API 的标准文档格式, 它使用 JSON Schema 来描述 API 的属性. 通过使用 OpenAPI, 你可以利用围绕该标准构建的整个工具和框架生态系统, 从而使 API 集成变得更加容易.
- JSON pointer 允许使用 `$ref` 关键字来引用一个模式(schema). 利用 JSON 指针, 可以创建可重用的模式定义, 这些定义可以在 API 规范的不同部分使用, 从而保持 API 规范的整洁和易于理解.
