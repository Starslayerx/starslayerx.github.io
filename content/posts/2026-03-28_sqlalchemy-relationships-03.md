+++
date = '2026-03-17T8:00:00+01:00'
draft = true
title = 'SQLAlchemy - Relationships: Advanced Many-To-Many Relationships
categories = ['Note']
tags = ['Python', 'Database']
+++

前面已经介绍了数据库的设计模块，但有时候这些需要进行一些微调 `tweak`，本篇文章探索多对多关系中一直非常有用的变体。

## Many-To-Many Relationships with Additional Data

本章会为 RetroFun 添加一个订单 ordering 的子模块，会包含客户 customers 和订单 orders。
原有的表有 `products`, `manufacturers`, `products_countries` 和 `countries` 表，他们的关系是：

- `manufacturers`:`products` 为一对多
- `products`:`countries` 为多对多，通过 `products_countries` 中间表关联

现在新增这些表：`orders_items`, `orders` 和 `customers`。
新的 `customers` 和 `orders` 表是一对多关系，其中用户为一，订单为多。
`products` 和 `orders` 表是多对多关系，其中 `orders_items` 作为中间表，但是该中间表除了两个外键，还有额外的列。

该关系的连接表有效地代表了订单中的每一项，同时引用了订单和产品。
但这样不够高效，这周关系还需要为销售定义一个数量和单价。

在连接表中添加额外信息会使使其变得复杂。
在 `products` 和 `countries` 表之间的多对多关系没有任何额外的列，这会允许 SQLAlchemy 来完全控制是否根据需要，从表中插入或删除实体。
如果有额外的列，SQLAlchemy 不知道插入新项的时候，如何处理这些内容。

关系表中对于有额外列的多对多关系，要使用一种不那么自动化的工作流。
因为应用必须为关系表中的额外列提供值。

作为实现 RetroFun 订单功能的第一步，可以在应用中扩展 Customer 和 Order 模型。
并建立它们之间的一对多关系，将他们添加进入 `models.py` 文件的末尾。

```Python
from datetime import datetime
from uuid import UUID, uuid4


class Order(Model):
    __tablename__ = 'orders'

    id: Mapped[UUID] = mapped_column(default=uuid4, primary_key=True)
    timestamp: Mapped[datetime] = mapped_column(default=datetime.now(timezone.utc))
    customer_id: Mapped[UUID] = mapped_column(ForeignKey('customers.id'), index=True)
    customer: Mapped[int] = relationship(back_populates='orders')

    def __repr__(self):
        return f'Order({self.id.hex})'  # hex without '-'

class Customer(Model):
    __tablename__ = 'customers'

    id: Mapped[UUID] = mapped_column(default=uuid4, primary_key=True)
    name: Mapped[str] = mapped_column(String(64), index=True, unique=True)
    address: Mapped[str | None] = mapped_column(String(128),)
    phone: Mapped[str | None] = mapped_column(String(32))
    orders: WriteOnlyMapped['Order'] = relationship(back_populates='customer')  # Mapped[list['Order']]

    def __repr__(self):
        return f'Customer({self.id.hex}, "{self.name}")'
```

### UUID Primary Keys

该模型的一大区别在于，其主键使用 UUID，而不是 int 类型。

使用自增主键的问题在于，当 id 包含在 URL 或者邮件中时，会显露出数据库的大小。
大部分业务会倾向于对客户订单数量保密，因此在表中使用整数键并不是一个好主意。

避免泄露这一信息的方式就是使用非数字主键。
新的模型定义是使用 UUID，它们是 16 字节的二进制序列。
UUID 有很多类型，其中一种叫 UUID4，适合作为主键。

```Python
from uuid import uuid4
id = uuid4()

id.bytes
# b'TW.yS\x16K;\x80\xd1\xfb}\xd23[.'

id.hex
# '54572e7953164b3b80d1fb7dd2335b2e'
```

遗憾的是，SQLAlchemy 只能自动管理整数类型的主键。
为了避免每次添加新的 Order 或 Customer 实例时都手动创建并分配 UUID4，
这两个模型中的 id 列被赋予了一个默认参数 `default=uuid4`，
当应用程序未提供值时，该参数会自动为他们分配值。

默认参数可以设置为一个常数值，或者一个 Callable，例如一个函数。
注意在函数名称后面不要加括号 `()`。

两个类中的 `__repr__()` 方法都使用 UUID 对象的 hex 属性，将二进制 UUID4 解码为可打印的十六进制字符串，以便于调试。
customers 和 orders 之间的一对多关系是通过在 “多” 的一方添加外键建立的，在这个例子中是 `Order` 模型。
里面有 UUID 类型的 `customer_id` 列，对应 `Customer.id` 类型。

### Date and Time Columns

Order 模型有一个 `datetime` 类型的 `timestamp` 列。
和 UUID 类型相同，SQLAlchemy 会自动将 Python `datetime` 对象映射到合适的数据库类型。

`timestamp` 列也有一个默认值，将 `datetime.utcnow` 作为默认参数，
会使 SQLAlchemy 在每次向数据库添加新订单时使用 `datetime.utcnow()`

> 实际上 `datetime.utcnow()` 方法已经不再使用，可以使用 `datetime.now(timezone.utc)` 结合 `partial()` 替代
> `timestamp: Mapped[datetime] = mapped_column(default=partial(datetime.now, timezone.utc))`
> 这里不能直接传递带参数的函数，因为 default 这里要传递的是函数本身，而不是运行结果，除了 partial 也可以使用 lambda 函数 `lambda: datetime.now(timezone.utc)`

这实际上意味着在添加订单时，时间戳列可以留空，而在提交操作时间，当前日期和实际将自动设置。
`__repr__()` 方法都使用 `hex` 十六进制方式解码二进制的 UUID4。

customers 和 orders 之间的外键关系通过在 “多” 的一方添加外键实现，即 `Order` 模型。

### Date and Time Columns

Order 模型具有一个使用 datetime 类型声明的 timestamp 列。
与 UUID 类型类似，SQLAlchemy 会自动将 Python 的 datetime 对象映射到数据库中相应的类型。

当处理日期和实际这些值的时候，维护不同的时区很重要。
一种明智的方法是将所有时间存储为 UTC 时区，因此设置为 utcnow 而不是 now。
该应用程序能够将从数据库获取的 UTC 时间戳，在呈现给用户时转换为客户端时区。
对于网络应用而言，这一转换通常由浏览器完成。

### Write-Only Relationships

这些模型之间的关系定义类似 manufacturers 和 products 之间的关系。
`Order.customer` 关系使用默认的延迟加载策略，而 `Customer.orders` 关系则采用了新的定义：

```Python
orders: WriteOnlyMapped['Order'] = relationship(back_populates='customer')
```

`WriteOnlyMapped` 类型提示使用 `lazy='write_only` 选项。
如果不使用类型提示，可以在 `relationship()` 中直接指定 lazy 参数。

通常，关系的惰性求值只在某些情况下才有意义。
哪些加载 “一对一” 端的关系，比如偶尔使用的 `Order.customer` 非常适合进行惰性求值。
同样，哪些预期元素数量较少的关系也是如此。

`Customer.orders` 之所以定义为 `write_only` 关系在于懒加载对其而言并不是很有用。
对于重复购买的客户而言，这可能是一个很长的关系链，将客户的所有订单作为一个列表获取，并不太有用。
有用的可能是只请求客户最近的订单，例如上个月的订单。
但是懒加载关系不支持定义过滤器，去检索一定范围内的元素。

`write_only` 加载器的便利之处在于，它不会尝试加载关系，而是生成一个查询对象。
可以自行执行，可能是在过滤器、排序或任何需要执行的选项后。
该关系也有 `add()`, `add_all()` 和 `remove()` 方法可以从关系中添加或移除元素。

### Association Object Pattern

下一步是创建 Order 和 Product 模型之间的多对多关系，这将定义每个订单的内容。
对于一个有额外列的多对多关系，关联表可以使用 `Table` 实例并通过 SQLAlchemy 管理。
因为关系需要额外的数据，因此该模型使用 Model 的子类创建，来允许应用管理额外的列。
SQLAlchemy 将这种定义为多对多关系的替代方法称为关联对象模式 Association Object Pattern。

下面是新的连接表代码：

```Python
class OrderItem(Model):
    __tablename__ = 'orders_items'

    product_id: Mapped[int] = mapped_column(ForeignKey('products.id'), primary_key=True)
    order_id: Mapped[UUID] = mapped_column(ForeignKey('orders.id'), primary_key=True)
```
