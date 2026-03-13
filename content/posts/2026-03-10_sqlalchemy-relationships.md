+++
date = '2026-03-10T8:00:00+01:00'
draft = true
title = 'SQLALchemy - Relationships'
categories = ['Note']
tags = ['Python', 'Database']
+++

## One-To-Many Relationships

在之前的文章介绍了如何产品表的，有趣的事，有些查询是为了查找产品制造商，而不是产品本身。
这里会介绍如何分组去重。

很多时候，分组都十分有用，尤其是使用聚合函数 aggregation functions 计算为存储在数据库中的分组信息。
当次要数据存储在与主要实体相同的表中时，会出现重复的问题。
因为表可能会变得比实际需要的大得多，而重复数据中的拼写差异可能导致分组结果错误。

### One-To-Many Relationships Implementation

关系数据库通过关系 relationships 创建链接，有两种主要的关系：

- One-to-many: 一对多
- Many-to-many: 多对多

一对多是说，有表 A 和 B，A 中的项可以对应 B 中的任意多项，而 B 中的项最多只能对应 A 中的一项。
该模式与电脑制造商和其产品的关系相同，制造商可以制造多种型号产品，而每种产品只能有一个制造商。
这里制造商就是一 "one"，产品就是多 "many"。

定义关系的第一步是先为这两个实体创建数据库表。
因此要定义 products 表和 manufacturer 表。

```Python
class Manufacturer(Model):
    __tablename__ = 'manufacturers'

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64), index=True, unique=True)

    def __repr__(self):
        return f'Manufacturer({self.id}, "{self.name}")'
```

原始的 Product 表中的 manufacturer 项被移除，并使用 `manufacturer_id` 替代：

```Python
from sqlalchemy import ForeignKey

class Product(Model):
    __tablename__ = 'products'

    id: Mapped[int] = mapped_column(primary_key)
    name: Mapped[str] = mapped_column(String(64), index=True, unique=True)
    manufacturer_id: Mapped[int] = mapped_column(ForeignKey('manufacturers.id'), index=True)
    year: Mapped[int] = mapped_column(index=True)
    country: Mapped[str] = mapped_column(String(32))
    cpu: Mapped[Optional[str]] = mapped_column(String(32))

    def __repr__(self):
        return f'Product({self.id}, "{self.name}")'
```

引用其他表的主键的列，被称为外键 foreign key，并会给予一个外键约束。
一个好的命名惯例是使用被引用的表，并添加后缀 `_id`。

`ForeignKey` 类会为该列创建一个外键约束 constraint。
传递给类的参数指明了目标主键，它可以是一个列对象，如 `Manufacturer.id` 也可以是一个 `<tablename>.<column>` 格式的字符串。
使用字符串格式可以引用文件中后面才定义的实体。

新的 `manufacturer_id` 列有一个索引，以帮组数据库高效地检索该列数据。
有些数据库会自动为外键添加索引，有些则不会，所以最好显示添加索引。
它永远都会有 `NOT NULL` 约束，这间接源于该列的 `Optional` 类型提示。
声明外键不能为空，确保不会出现没有制造商的产品。

一对多关系总是遵循制造模式，即 "many" 的一方(Product)包含一个外键列(manufacturer_id)，该列引用 "one" 的一方(Manufacturer)。

### SQLAlchemy Relationships

将数据分解到不同到表中会带来一个复杂的问题，因为现在 Product 只能提供制造商的 id 属性，而这个属性只是一个数字。
这个数字必须从新的 Manufacturer 表中加载制造商信息。

好在，SQLAlchemy 的 ORM 模块为关系提供了高级支持，是的处理外键导航的大部分工作变得透明。
要使用这些功能，涉及关系的两个模型类需要明确定义这种关系的 relationship 属性。
下面为 Product 和 Manufacturer 定义这些对象。

```Python
from sqlalchemy.orm import relationship

class Product(Model):
    # ...
    manufacturer: Mapped['Manufacturer'] = relationship(back_populates='products')

class Manufacturer(Model):
    # ...
    products: Mapped[list['Product']] = relationship(back_populates='manufacturer')
```

现在 Product 类型有一个 `manufacturer` 属性，代表 "many" 一方的关系。
该属性并不是数据库中的一列，但会通过外键的方式存储在数据库中。
这是高级别的 `manufacturer_id` 替代方案，能够透明地加载相关的模型对象。

在 Manufacturer 类中，增加了 `products` 类，代表 "one" 一方的关系。
从这个方向看去，一个产品制造商可以拥有许多相关产品。
因此，这个属性是一个列表，会自动填充对应的产品实例。

为关系属性提供的关系提示基于和列相同的 `Mapped[x]` 类型。
但对于这些属性，x 代表关系另一侧的模型，在 "一对一" 关系中直接使用模型类，在 "一对多" 关系中则使用模型类列表。
和前面的例子一样，可以直接使用类名称，或者使用字符串。
使用字符串通常是在需要向前引用时避免错误。
考虑到一致性，字符串是更好的选择。

这两个 `relationship()` 定义都由 `back_populates` 参数，每个都设置为另一侧关系属性的名称。
这是为了让 SQLAlchemy 理解这两个属性代表同一关系的两个方面。

## A Revised Importer Application

下面重新编写从 csv 导入数据的脚本：

```Python
# import_products.py
import csv
from db import Model, Session, engine
from models import Product, Manufacturer


def main():
    Model.metadata.drop_all(engine)  # warning: this deletes all data!
    Model.metadata.create_all(engine)

    with Session() as session:
        with session.begin():
            with open('products.csv') as f:
                reader = csv.DictReader(f)
                # 确保相同 生产商 的产品引用同一个 实例
                all_manufacturers = {}

                for row in reader:
                    row['year'] = int(row['year'])

                    manufacturer = row.pop('manufacturer')
                    p = Product(**row)

                    if manufacturer not in all_manufacturers:
                        m = Manufacturer(name=manufacturer)
                        # 添加供应商
                        session.add(m)
                        all_manufacturers[manufacturer] = m
                    # 添加产品，并关联到该供应商
                    all_manufacturers[manufacturer].products.append(p)

if __name__ == '__main__':
    main()
```

在这个版本中，`all_manufacturers` 在 csv 导入前的循环中，初始化为空表。
然后循环对 csv 的每一行进行处理，首先将 `manufacturer` 存入字典中，并将其去除后，实例化到 Product 表中。

---

上面代码插入数据的两种方式：

- `session.add(Model(...))`: 告诉数据库存入特定的表对象
- `parent.relationship.append(child)`: 关系联动，由于产品需要生产商的 id 进行插入，因此通过对应厂商的关系，来插入产品数据，这样 ORM 可以自动处理外键。

---

最后一行，新产品会追加 append 到生产商 `products` 的关系中，其操作方式类似列表。
这个代表 "多" 的一方 `products` 关系对象有 `append()` 和 `remove()` 方法。
SQLAlchemy 会自动将这些操作转换为对应的外键变更。

从细节上，`append()` 调用 `products` 关系属性实现了两件事：
首先，它通过 `manufacturer_id` 外键将 manufacturer 和 product 关联起来，当 session commit 的时候会自动设置。
其次，它间接地将新产品 product 包含在数据库会话中，因为它被之前添加的制造商 manufacturer 实例引用。
对产品进行清晰的 `session.add(p)` 不会造成任何损害，但并非必要。
当父对象已在会话中时，自动将其子对象假如会话的操作称为级联 cascade。

当 session 块结束当时候，所有 manufacturers 和 products 都通过一条原子操作加入了数据库。

## One-To-Many Relationship Queries

下面示例展示如何查询

```Python
from sqlalchemy import select
from db import Session
from models import Product, Manufacturer
```

加载 `ZX Spectrum` 产品：

```Python
p = session.scalar(select(Product).where(Product.name == 'ZX Spectrum'))
p
# Product(127, "ZX Spectrum")
```

那么如何查询 product 的 manufacturer 呢？在单表情况下是 `p.manufacturer`。
而现在是拥有一个关系，而不是一个字符串列，而该关系会透明地返回模型实例：

```Python
p.manufacturer
# Manufacturer(63, "Sinclair Research")
```

该生产商的实际名称存储在模型的 `name` 属性中：

```Python
p.manufacturer.name
# 'Sinclair Research'
```

也可以从 manufacturer 开始浏览数据库，例如查询 "Texas Instruments" 的相关产品：

```Python
m = session.scalar(
    select(Manufacturer)
    .where(Manufacturer.name == 'Texas Instruments')
)
m
# Manufacturer(66, "Texas Instruments")
m.products
# [Product(132, "TI-99/4"), Product(133, "TI-99/4A")]
```

如果要同时查找产品和制造商的信息，在单表中很容易实现。
但在分开的表中，需要使用 `join` 来实现：

```Python
query = select(Product.name, Manufacturer.name).join(Product.manufacturer)
session.execute(query).all()
# [('Acorn Atom', 'Acorn Computers Ltd'), ...]
```

在该查询中，`select()` 选择了两个不同表的属性。
每当查询涉及多个表的时候，SQLAlchemy 都需要知道如何将这些表连接 join 起来。
当使用 ORM 模块，`join()` 方法的参数可以是两个关系属性之一，SQLAlchemy 会据此自动推断出所有信息。
由于者两个关系通过 `back_populates` 选项连接起来，通常来说，在 `join()` 子句中指定两者中的哪一个并不重要。
在这个查询例子中，将 `Product.manufacturer` 传递给 `join()` 意味着 Product 将作为连接的左侧，Manufacturer 将会作为右侧。
如果传递的是 `Manufacturer.products`，那么两侧将会反转，但结果不会变。
但在某些情况下，左右侧的实体会很重要。

可以使用 `print()` 打印查看实际的 SQL 语句：

```Python
print(query)
# SELECT products.name, manufacturers.name AS name_1
# FROM products JOIN manufacturers ON manufacturers.id = products.manufacturer_id
```

这样可以理解 SQLAlchemy 如何自定义连接条件。

下面这个查询根据字母表排序，计算每个生产商产品的数量：

```Python
from sqlalchemy import fun

query = (
    select (Manufacturer, func.count(Product.id))
    .join(Manufacturer.products)
    .group_by(Manufacturer)
    .order_by(Manufacturer.name)
    )
session.execute(query).all()
# [(Manufacturer(1, "Acorn Computers Ltd"), 6), ...]
```

这个查询并不是第一个包含多个结果的例子，但它是第一个结果包含模型和其他值的例子。
注意到，Manufacturer 在 `select()` 内，同时也在 `group_by()` 内。
当 `group_by()` 接收一个模型类作为参数，而非单一属性时，分组操作将依据该模型的所有属性组合进行。
同样的，可以查询其 SQL:

```Python
print(query)
# SELECT manufacturers.id, manufacturers.name, count(products.id) AS count_1
# FROM manufacturers JOIN products ON manufacturers.id = products.manufacturer_id GROUP BY manufacturers.id, manufacturers.name ORDER BY count(*)
```

SQLAlchemy 极大地简化了这种查询操作，考虑到 `manufacturers` 表列的添加或删除列时，该查询会动态调整，并仍能够将模型作为一个整体进行分组，无需任何更改。

## Lazy vs. Eager Relationships

当访问模型中这些神奇属性会发生呢，一个窥探的方式是在创建引擎的 `create_engine()` 函数添加 `echo=True` 参数：

```Python
engine = create_engine(os.environ['DATABASE_URL'], echo=True)
```

运行后会看到下面类似的输出

```text
2026-03-12 15:58:35,606 INFO sqlalchemy.engine.Engine select pg_catalog.version()
2026-03-12 15:58:35,607 INFO sqlalchemy.engine.Engine [raw sql] {}
2026-03-12 15:58:35,617 INFO sqlalchemy.engine.Engine select current_schema()
2026-03-12 15:58:35,617 INFO sqlalchemy.engine.Engine [raw sql] {}
2026-03-12 15:58:35,620 INFO sqlalchemy.engine.Engine show standard_conforming_strings
2026-03-12 15:58:35,620 INFO sqlalchemy.engine.Engine [raw sql] {}
2026-03-12 15:58:35,627 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2026-03-12 15:58:35,630 INFO sqlalchemy.engine.Engine SELECT manufacturers.id, manufacturers.name
FROM manufacturers
WHERE manufacturers.name = %(name_1)s::VARCHAR
2026-03-12 15:58:35,630 INFO sqlalchemy.engine.Engine [generated in 0.00009s] {'name_1': 'Texas Instruments'}
```

直到 BEGIN 字段之前的，都属于初始化阶段内容。
SELECT 语句是 `scalar()` 调用的实际执行，随后是一行摘要，显示查询中使用的占位符值。

现在尝试访问人任何列属性的时候，不会有额外日志输出，而访问关系的时候会有日志信息：

```Python
m  # Manufacturer(66, "Texas Instruments")

m.name  # 'Texas Instruments'

m.products  # [Product(132, "TI-99/4"), Product(133, "TI-99/4A")]
# 2026-03-12 16:19:37,852 INFO sqlalchemy.engine.Engine SELECT products.id AS products_id, products.name AS products_name, products.manufacturer_id AS products_manufacturer_id, products.year AS products_year, products.country AS products_country, products.cpu AS products_cpu
# FROM products
# WHERE %(param_1)s::INTEGER = products.manufacturer_id
# 2026-03-12 16:19:37,853 INFO sqlalchemy.engine.Engine [generated in 0.00274s] {'param_1': 66}
```

这样因为所有 query 得到的属性都缓存在数据库 session 中。
而访问关系的时候，SQLAlchemy 在独立执行数据库查询，从而提供制造商相关的产品列表。
如果重新访问同一个关系，响应会立刻返回，因为结果缓存到了数据库 session 中。
从关系的另一端进行导航时，可以观察到类似的行为。

当 SQLAlchemy 通过这种方式工作的时候，它使用的是懒加载 "lazy" 的关系加载方式。
通过这种方式，应用可以不必知道这两者之间的关系 relationship，并将他们当初普通的属性使用。

懒加载看上去很好，但它有缺点。
由于数据库查询是隐式的，会让应用失去对发生给数据库的查询数量，如果数量太多则会影响数据库的性能。

下面展示一个懒加载可能导致问题的例子：

```Python
q = select(Product.name, Manufacturer.name).join(Product.manufacturer)
```

该查询十分高效，因为单个数据库操作成对返回所有的产品和制造商名称。
一个不了解懒加载机制的开发者可能不同的方式获取到相同的信息。
例如利用 `manufacturer` 关系 relationship 使用循环：

```Python
query = select(Product)
for q in session.scalars(query):
    print(q.name, p.manufacturer.name)
```

第一眼看上去好像是获取数据对的合理方式。
但开启 `echo=True` 参数执行这段代码，会看到大量的数据库信息输出。
这个循环所需要的查询次数是：存储在变量 q 中的初始查询需要一次查询，加上每个制造商额外的一次懒加载查询。

### Relationship Loaders

好消息是 SQLAlchemy 提供了一些关系的配置选项，使其更加方便和高效。
SQLAlchemy 使用关系加载器 relationship loader 来加载一个或多个对象到 session 中。
默认的 loader 是 _select_ loader。

另一种加载器是连接加载器 _joined_ loader，这种加载器在主查询中扩展一个 `join` 子句，在获取父对象的同时，从数据库中读出关联对象。

_select_ loader 是懒 "lazy" loader，因为数据库关系的查询会被推迟。
而 _joined_ loader 是 "eager" loader 预加载器，因为 relationship 数据会在父对象请求的时候同时加载进来。

通过 `options()` 方法来启用该 loader

```Python
from sqlalchemy.orm import joinedload

q = select(Product).options(joinedload(Product.manufacturer))
```

这会告诉 SQLAlchemy 覆盖默认的懒加载机制，并使用 joined loader 将 `manufacturer` 关系加载到 session 中。
与其在每个查询中选择 loader，也可以直接在 relationship 中选择默认的 loader。
通过关系传递 `lazy='joined'` 参数来实现：

```Python
class Product(Model):
    # ...
    manufacturer: Mapped['Manufacturer'] = relationship(lazy='joined', back_populates='products')
```

除了 _select_ 和 _joined_ 还有其他的 loader，下面是完整的介绍：

- select 懒加载：`select()` 语句，当关系属性第一次访问时。这是默认行为，作为查询选项，可以通过 `lazyload()` 函数启用此加载器。

- immediate: 在父实体加载的同时，通过 `select()` 语句加载相关实体。两种唯一区别在于，后者会预先执行所有关系查询，而不是按需查询。此加载器可以通过 `immediateload()` 函数启用。

- joined 预加载：加载父实体的时候同时加载相关联实体，通过使用 join 来扩展父查询来实现。在查询中使用 `joinedload()` 实现。

- subquery 子查询：在主查询后立即加载关联实体，它会发起一次额外的查询，将原始查询转换为一个子查询 subquery，并于关联表进行连接。在 `select()` 中通过 `subqueryload()` 函数启用。

- selectin 谓词加载：在主查询后，立即加载相关关联实体，它会发起一次额外的查询，并使用 IN operator 运算符指定主键列表。在查询选项中，通过 `selectinload()` 启动。

- write_only 只写加载：禁用关系的自动加载。仅 SQLAlchemy 2.0+ 可用，并且只能用于 `relationship`，没有对应的 `option` 版本，后面会介绍。

- noload 不加载：类似 `write_only` 但是灵活性更低，不建议使用。

- raise 与 raise_on_sql：这两种模式会在需要隐式加载关系时抛出异常，它们常用于检测不经意间触发的隐式 I/O 操作。对应的 `select()` 选项是 `raiseload()`。

- dynamic 动态加载：一种遗留加载器 legacy loader，与新版本不兼容。

| 加载时机 (When)                   | 加载器 (Loaders)                                       |
| :-------------------------------- | :----------------------------------------------------- |
| **懒加载 (Lazy load)**            | `select` (默认), `dynamic` (遗留)                      |
| **预加载 (Eager load)**           | `joined`, `selectin`, `subquery`, `immediate`          |
| **显式/禁用加载 (Explicit load)** | `write_only` (2.0+), `noload`, `raise`, `raise_on_sql` |

使用上面这个表，将选择降低到了三组，每个组内的加载器尽在实现上有所不同，但操作方式类似。
很难决定应该使用那种加载机制，一个修改默认加载机制的方式是，当数据库遇到太多小的关系查询的时候去修改提升性能。
在项目早期阶段，并无需担心这个问题，因此一个明智的决定是继续使用懒加载器。
并将任何可能的改动推迟到以后，等到这些关系更频繁使用时再考虑。

## Deletion of Related Objects with a Cascade

> 通过级联删除相关对象

通常，在一对多关系中，从 ”多“ 方删除操作的方式相同，不会出现问题。

```Python
p = session.get(Product, 24)
m = p.manufacturer
```

然后可以删除产品

```Python
session.delete(p)
session.commit()
```

但是通过同样方式删除 manufacturer 会出问题：

```Python
session.delete(m)
session.commit()
```

会有如下报错

```text
[ traceback omitted ]sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) NOT NULL constraint failed: products.manufacturer_id[SQL: UPDATE products SET manufacturer_id=? WHERE products.id = ?][parameters: ((None, 25), (None, 26), (None, 27), (None, 28), (None, 29), (None, 30))](Background on this error at: https://sqlalche.me/e/14/gkpj)
```

如果制造商拥有上述被删除的那款产品，则删除操作本应该成功。
但由于制造商还拥有其他几款产品，这些产品仍然存在，并且他们的 `manufacturer_id` 都指向这条记录。
如果删除制造商，这些产品的外键将会变为无效
