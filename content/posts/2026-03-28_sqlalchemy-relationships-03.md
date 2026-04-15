+++
date = '2026-03-17T8:00:00+01:00'
draft = true
title = 'SQLAlchemy - Relationships: Advanced Many-To-Many Relationships'
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
    unit_price: Mapped[float]
    quantity: Mapped[int]
```

`OrderItem` 的主键由两个外键构成，且产品没有 `Optional` 类型提示，这意味着如果某个产品或订单被删除，那么引用该已删除的项目的表的所有条目也必须删除。
该模型有额外两列来存储产品的 unit price 单元价格和 quantity 数量，这些都是购买相关的必要信息。

有趣的是，在类似的多对多关系中，产品和国家之间的关系都使用 `secondary` 参数，这对于 SQLAlchemy 来说就知道如何去处理了。
不幸的是，`secondary` 参数无法在该示例中使用。
因为应用程序无法放弃对连接表的控制权，这是由于额外的列所导致的。

实现这一点的解决方案源于一个事实：多对多关系可以被视作两个一对多关系，或者更准确的说。
是第一个模型与连接表之间的一对多关系，以及连接表和第二个模型之间的多对一关系。
事实证明，如果分别考虑这段关系的两个部分，就没有必要依赖 `secondary` 参数。

下面可以看到，如何通过两边的多对多关系，通过一对多关系来实现。

```Python
class Product(Model):
    # ...
    order_items: WriteOnlyMapped['OrderItem'] = relationship(back_populates='product')
    # ...

class Order(Model):
    # ...
    order_items: Mapped[list['OrderItem']] = relationship(back_populates='order')
    # ...

class OrderItem(Model):
    # ...
    product: Mapped['Product'] = relationship(back_populates='order_items')
    order: Mapped['Order'] = relationship(back_populates='order_items')
    # ...
```

结果是增加了 4 个关系，每两个形成一个一对多关系。
一个 `Order` 实例使用 `order_items` 关系属性来获取订单中包含的订单项 Item 列表。
此关系返回的列表中的每个元素都将是 OrderItem 连接表模型的一个实例。
该实例提供了产品、单价和数量的信息。

从 `Product` 产品角度来开，`order_items` 关系属性代表了一系列购买记录，
这些记录同样是 `OrderItem` 的实例，每个实例都提供了对订单的引用、单价以及数量信息。
由于一个产品可能被多次销售，这部分关系通过 `write_only` 加载器定义，这使得应用程序能够通过筛选、排序选择和分页来查询此列表。

### A New Database Migration

现在 `models.py` 里面的 customers 和 orders 的实现已经完成了，现在就是要去迁移数据库了。

```console
$ alembic revision --autogenerate -m "customers and orders"
```

当确认生成的迁移脚本包括这 3 个新表后，将其应用到数据库：

```console
$ alembic upgrade head
```

之后数据库就可以接受 customers 和 orders 了。

### How to Create an Order

有额外数据的多对多模型很强大，但是它们对比简单的模型有很多副作用，因为该表需要应用自己管理。

这种情况下，创建订单的步骤如下：

- 首先，创建一个 `Customer` 实例，或加载一个现存的实例，如果客户相同的话
- 然后，创建一个 `Order` 实例，并将其和 `Customer` 关联。
  在构造器通过传入 `customer` 参数或者通过 `Customer.orders` 关系调用 `add()`
- 对于 `Order` 中的每一项，创建一个 `OrderItem` 实例附加 product, unit price 和 quanitty，并将其加入 `order_items` 关系

实际代码如下：

```Python
# import all necessary things and create a session
from models import Product, Customer, Order, OrderItem
from db import Session

session = Session()

# create a new customer
c = Customer(name='Jane Smith')

# create a new order, add it to the customer and to the database session
o = Order()
c.orders.add(o)
session.add(o)

# add the first line item in the order: product #45 to $45.5
p1 = session.get(Product, 45)
o.order_items.append(OrderItem(product=p1, unit_price=45.5, quantity=1))

# add the second line item: 2 of product #82 for $37 each
p2 = session.get(Product, 82)
o.order_items.append(OrderItem(product=p2, unit_price=37, quantity=2))

# write the order (along with the customer and order items) to the database
session.commit()

# check the UUID and the timestamp defaults assigned to the new order
o.id
o.timestamp
```

---

SQLAlchemy 对象生命周期

1. Transient 瞬时态
   调用 `o = Order()` 创建对象时，是 “瞬时” 的。
   数据库此时并不知道它的存在，也没有主键 ID，如果修改它的属性，Session 不会理会。

2. Pending 待定态
   当执行 `session.add(o)` 时，对象就进入了 “待定” 态。
   对象被数据库收编，session 将其放入一个内部队列。
   数据库此时仍然没有这条数据，但 Session 已经准备好在下一次 “刷新” 时发送 `INSERT` 语句。

3. Persistent 持久态
   当执行 `session.commit()` 或者 `session.flush()` 后，对象就变成了 “持久” 的。
   此时对象与数据库的某一行记录一一对应，对象会获得真实的数据库 `id` 主键。
   如果修改了持久对象的属性，Session 会在下一次 commit 时自动发出 `UPDATE` 语句。

4. Detached 脱离态
   如果关闭了 Session `session.close()`，原本持久态的对象就变成的 “脱离” 态。
   此时，对象仍存在于内层中，且带着之前从数据库拿到的 ID。
   由于连接断开了，Session 不再追踪这个对象的任何变化。
   如果此时尝试访问一个从数据库延迟加载的属性 Lazy loading，会报 `DetachedInstanceError` 错误。

---

值得注意的是，用于添加和移除关系项的方法并不总是一致的。
对于使用标准 Python 列表表示的关系，使用了 `append()` 和 `remove()` 方法。
例如上面的 `Order.order_items` 关系使用 `append()` 添加。

使用 `write_only` 加载器的关系并不列表语义，因为相关项不会直接加载。
这些关系的类型是 `WriteOnlyCollection`，并且有 `add()` 和 `delete()` 方法。
上面的例子 对 `Customer.orders` 关系使用 `add()` 方法。

### Deletions

与以往的关系一样，理解关系合适删除也很重要。
如果至少在一个订单条目中引用的 `Product` 或 `Order` 实体被删除，
SQLAlchemy 会尝试为 `OrderItem` 实例的无效外键中添加一个 `NULL`。
该模型中的两个外键都可以定义为非空，因此该尝试会失败。
对于 products 和 countries 关系而言，这不是一个问题，
因为关系的次要选项确保的当其中一个关联实体被删除时，连接表中的元素也会被删除。

处理多对多关系且为未使用 `secondary` 选项时，有两种方法应对此问题。
一种解决方案是实现一个 `"delete"` 级联 cascade，类似 `Manufacturer.products` 中配置的关系。

```Python
class Manufacturer:
    ...
    products: Mapped[list['Product']] = relationship(
        cascade='all, delete-orphan',
        back_populates='manufacturer',
    )
```

这样会自动删除 `OrderItem` 孤儿项，类似 SQLAlchemy 的 `secondary` 关系。
另一个选项是当 `OrderItem` 实例的时候，假定产品或订单不能删除。

第二个选择是必要的什么都不做选项，因为在删除 product 或 order 的时候 SQLAlchemy 会报错。
在连接表中设置外键为必须项，是一种便捷的方法，以确保只有在没有 OrderItem 实例指向他们时，才能删除产品或订单。
要删除一个订单 Order，应用程序首先需要删除所有和其相关联的 `ItemOrder` 实体，然后才能移除 `Order` 实体。

由于数据库中没有其他指向 `OrderItem` 的外键，因此这些实例可以安全地被删除，并不引发数据库完整性错误。
`OrderItem` 实体可以从关系的任意一侧删除，只要从 Order 或 Product 模型中的相应 `order_items` 关系中移除实例即可。

```Python
# this assums 'oi' has the OrderItem instance to delete
# delete the OrderItem from an Order instance "o"
o.order_items.remove(oi)
session.commit()

# delete the OrderItem from a Product instance "p"
p.order_items.remove(oi)
session.commit()
```

### An Order Importer Script

为了能够对充满订单的运行查询和进行实验，一下脚本从 CSV 文件导入了大量随机生成的订单，代码在 `import_orders.py` 文件中：

```Python
import csv
from datetime import datetime
from sqlalchemy import select, delete

from db import Session
from models import Product, Customer, Order, OrderItem


def main():
    with Session() as session:
        with session.begin():
            session.execute(delete(OrderItem))
            session.execute(delete(Order))
            session.execute(delete(Customer))

    with Session() as session:
        with session.begin():
            with open('orders.csv') as f:
                reader = csv.DictReader(f)
                all_customers = {}
                all_products = {}

                for row in reader:
                    if row['name'] not in all_customers:
                        c = Customer(
                            name=row['name'],
                            address=row['database'],
                            phone=row['phone'],
                        )
                        all_customers[row['name']] = c

                        o = Order(timestamp=datetime.strptime(row['timestamp'], '%Y-%m-%d %H:%M:%S'))
                        all_customers[row['name']].orders.add(o),
                        session.add(o)

                        product = all_products.get(row['product1'])
                        if product is None:
                            product = session.scalar(
                                select(Product) .where(Product.name == row['product1']),
                            )
                            all_products[row["product1"]] = product

                        o.order_items.append(
                            OrderItem(
                                product=product,
                                unit_price=float(row["unit_price1"]),
                                quantity=int(row["quantity1"]),
                            ),
                        )


                        if row["product2"]:
                            product = all_products.get(row["product2"])
                            if product is None:
                                product = session.scalar(
                                    select(Product).where(Product.name == row["product2"])
                                )
                                all_products[row["product2"]] = product
                            o.order_items.append(
                                OrderItem(
                                    product=product,
                                    unit_price=float(row["unit_price2"]),
                                    quantity=int(row["quantity2"]),
                                )
                            )

                        if row["product3"]:
                            product = all_products.get(row["product3"])
                            if product is None:
                                product = session.scalar(
                                    select(Product).where(Product.name == row["product3"])
                                )
                                all_products[row["product3"]] = product
                            o.order_items.append(
                                OrderItem(
                                    product=product,
                                    unit_price=float(row["unit_price3"]),
                                    quantity=int(row["quantity3"]),
                                )
                            )
```

第一份代码块删除所有的订单和客户信息，然后在第二个代码块读取 CSV 文件并按行处理。
没一行的数据关于订单的信息，包括这些字段：

- name: 客户名称
- address: 客户地址
- phone: 客户电话
- timestamp: 订单日期和时间
- product1, unit_price1, quantity1: 该订单中的第一行项目
- product2, unit_price2, quantity2: 该订单中的第二行项目，quantity2=0 表示该订单没有第二行
- product3, unit_price3, quantity3: 该订单中的第三行项目, quantity3=0 表示该订单没有第三行

`all_customers` 字典用于追踪从文件中读取订单时常见的 `Customer` 实例。
`all_products` 字典维护了一个从数据库加载的产品的缓存，以减少发出的查询数量。

创建 `Order` 实例并将其和 customer 链接起来。
`timestamp` 属性会显式根据 CSV 内容设置文件日期和时间，防止模型中的默认条款将所有订单时间设置为脚本运行时间。

为了完成订单，会创建一个、两个或三个 `OrderItem` 实例并将其附加到订单上。
对于每个订单项，如果之前未见过该产品，则通过查询从数据库中加载，
如果在 `all_products` 缓存中找到，则从缓存中加载。

### Queries

对于一个有大量 customers 和 orders 的数据库，现在就可以实现很多有趣的查询了。

```Python
from sqlalchemy import select, func
from db import Session
from models import Product, Customer, Order, OrderItem

session = Session()
```

现在来加载一些 `write_only` 类型是关系。
这个查询通过客户姓名获取客户信息，然后访问订单关系属性：

```Python
c = session.scalar(
    select(Customer)
    .where(Customer.name == 'John Butler')
)
c.orders
# <sqlalchemy.orm.writeonly.WriteOnlyCollection at 0x10987e520>
```

可以看到，关系的类型并不是通过列表来显示的。
但是该 `WriteOnlyCollection` 对象有一个 `select()` 方法，
该方法会返回一个查询对象，当查询被执行时，他才会去获取相关的对象。

```Python
session.scalars(
    c.orders.select()
).all()
```



