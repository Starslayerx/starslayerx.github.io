+++
date = '2026-03-17T8:00:00+01:00'
draft = false
title = 'SQLAlchemy - Relationships: Many-To-Many Relationships'
categories = ['Note']
tags = ['Python', 'Database']
+++

多对多类型，何其名称暗示的一样，当无法认定任何一方为 "一" 的时候使用。

## Many-To-Many Relationships

在标准的 one-to-many 一对多关系中，"多" 的一方会有一个 foreign key 外键指向 "一" 的一方。
但是在尝试建立 Product 和 Country 之前的关系时，在产品中添加指向国家的外键不行，因为这样一个产品就只能来自一个国家。
在国家中添加外键指向产品也不行，这样一个国家就只能对应一种产品。

### How Many-To-Many Relationships Work

为了实现多对多关系，需要两个一对多关系来表示这种复杂的关系。
由于无法在两个表之间建立直接的多对多关系，因此需要添加一个称为连接表 join table 的第三个表。
每一边都和 join table 维护一个一对多关系，这意味着连接表有两个外键，指向两个表。

例如：`products` 表的 id 和 `products_countries` 的 products_id 形成 1:N 关系，`countries` 表的 id 和 `cuontry_id` 形成 1:N 关系。
这里的 `products_countries` 表就是 join table 连接表，常见的命名规范就是使用构成关系的两个实体的名称。

如果一个产品在 3 个国家生产，那么 join table 连接表就会有 3 对实例。
从另一个方面来看，对于一个已经创建了 7 个产品的国家，将会有对应数量的条目。
其 country_id 被设置为该国家，每个条目都将其与其中一个产品关联起来。

### A Simple Many-To-Many Relationship Implementation

管理所有这些连接表的外键关系听起来像是噩梦，但 SQLAlchemy 会完成大部分困难的工作。
实现这种关系的第一步是创建连接表，该表将被 SQLAlchemy 管理，因此所有通过 Declarative Base 提供的类型都是非必要的。
相反，该表可以通过 SQLAlchemy Core 提供的 `Table` 类型创建。
为了方便引用 Product 表和还未创建的 Country 表，join table 管理表应被添加到这些类型的上面。

```Python
from sqlalchemy import Table, Column

ProductCountry = Table(
    'products_countries',
    Model.metadata,
    Column('product_id', ForeignKey('products.id'), primary_key=True, nullable=False),
    Column('country_id', ForeignKey('countries.id'), primary_key=True, nullable=False),
)
```

`Table` 构造器第一个参数为表名，后面跟着数据库 metadata 元数据实例。
剩下两个是 `Column` 参数作为外键 foreign keys。
由于使用 `Column()` 构造器，而不是类型提示 typing hints，需要添加 `nullable=False` 选项来确保这些列必须存在。

不像其他从模型派生的表不同，改表没有一个 `id` 作为主键 primary key，相反将这两个外键声明为主键。
当多个列被声明为主键时，SQLAlchemy 会创建复合主键。
根据定义，主键必须是唯一的，因此对于这个关系，该表不会允许行有相同的外键值。

---

> 这里采用 Table() 的写法，因为没有额外的字段，是单纯的关联表，不会对其进行任何直接的操作。
> 如果有额外字段，可以这样写：

```Python
class ProductCountry(Model):
    __tablename__ = 'products_countries'

    product_id: Mapped[int] = mapped_column(ForeignKey('products_id'), primary_key=True)
    country_id: Mapped[int] = mapped_column(ForeignKey('countries_id'), primary_key=True)
    added_at: Mapped[datetime] = mapped_column(default=func.now())  # 额外字段
```

---

更新后的 `Product` 模型和新的 `Country` 模型如下：

```Python
class Product(Model):
    __tablename__ = 'products'

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64), index=True, unique=True)
    manufactuer_id: Mapped[int] = mapped_column(ForeignKey('manufacturers.id'), index=True)
    year: Mapped[int] = mapped_column(String(32))
    # One-to-one relationship:
    # - Mapped['ClassName']: 指向的类名称
    # - back_populates: 双向绑定，目标类中的属性名称
    manufacturer: Mapped['Manufacturer'] = relationship(back_populates='products')
    # Many-to-many relationship:
    # - secondary: 指向中间表
    countries: Mapped[list['Country']] = relationship(secondary=ProductCountry, back_populates='products')

    def __repr__(self):
        return f'Product({self.id}, "{self.name}")'

class Country(Model):
    __tablename__ = 'countries'

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(32), index=True, unique=True)
    products: Mapped[list['Product']] = relationship(secondary=ProductCountry, back_populates='countries')
```

---

在 SQLAlchemy 中：

- 一对多关系：“多” 的一方需要定义外键，“一” 的一方不需要定义外键。双方都需要编写关系，“一” 的一方是列表类型关系，并且一般会写出复数形式
- 多对多关系：双方都不定义外键，而是专门定义一张连接表，包含两个外键，这两个外键共同构成主键。双方关系类型都指向对方，但是要通过 `secondary` 参数指向中间表 (连接表 join table)

---

`relationship()` 的 `secondary` 参数告诉 SQLAlchemy 该关系是通过一个辅助表来支持（join table）。
注意这里的 join table 是直接通过其类名称引用的，因此连接表需要在前面定义好。

### Product Importer Script Updates

下面要更新产品导入代码，如下：

```Python
import csv
from db import Model, Session, engine
from models import Product, Manufacturer, Country

def main():
    Model.metadata.drop_all(engine)  # warning: this deletes all data!
    Model.metadata.create_all(engine)

    with Session() as session:
        with session.begin():
            with open('products.csv') as f:
                reader = csv.DictReader(f)
                all_manufacturers = {}
                all_countries = {}

                for row in reader:
                    row['year'] = int(row['year'])
                    manufacturer = row.pop('manufacturer')
                    countries = row.pop('country').split('/')
                    p = Product(**row)

                    if manufacturer not in all_manufacturers:
                        m = Manufactruer(name=manufacturer)
                        session.add(m)
                    all_manufacturers[manufacturer].products.append(p)
                    for country in countries:
                        if country not in all_countries:
                            c = Country(name=country)
                            session.add(c)
                            all_countries[country] = c
                        all_countries[country].products.append(p)

if __name__ == '__main__':
    main()
```

### Many-To-Many Relationship Queries

同样的，添加到 Country 和 Product 表中的 `products` 与 `relationship` 关系也可以简化多对多的关系查询方式。

查询之前首先要导入相关组件：

```Python
from db import Session
from models import Product, Manufacturer, Country
from sqlalchemy import select, func

session = Session()
```

然后查询：

```Python
p = session.scalar(
    select(Product)
    .where(Product.name == 'Timex Sinclair 1000')
)

p
# Product(138, "Timex Sinclair 1000")

p.countries
# [Country(1, "UK"), Country(3, "USA"), Country(22, "Portugal")]
```

这里的 `countries` 关系使用懒加载，因此会在首次查询的时候，隐式获取国家列表。
同样地，可以通过国家查询产品：

```Python
c = session.scalar(
    select(Country)
    .where(Country.name == 'Portugal')
)
c.products
# [Product(138, "Timex Sinclair 1000"),
#  Product(139, "Timex Sinclair 1500"),
#  Product(140, "Timex Sinclair 2048"),
#  Product(141, "Timex Computer 2048"),
#  Product(142, "Timex Computer 2068"),
#  Product(143, "Komputer 2086")]
```

下面这个复杂查询展示了所有涉及多个国家的产品，以及具体的国家数量：

```Python
country_count = func.count(Country.id).label(None)
query = (
    select(Product, country_count)
    .join(Product.countries)
    .group_by(Product)
    .having(country_count > 1)
    .order_by(Product.name)
)
query.execute(query).all()
```

这里查询了产品和每个产品的国家数量，后者创建了一个 label 标签（重命名），并存储到 `country_count` 里面。
后面再使用 `select()` 和 `having()` 的时候就不需要重复编写了。

为了统计国家数量，需要将 products 和 countries 关联起来。
根据产品进行分组，将结果折叠成每行一个产品，但第二个结果运行 `count()` 聚合函数，并将国家列表替换为国家数量。
`having()` 语句过滤分组结果，并只保留有多个国家的产品。

这次查询的 `join()` 方法很有趣，因为多对多关系无法通过单个 join 查询得到。
实际上，在 SQL 中无法直接将 `products` 和 `countries` 连接起来，因为他们之间没有可以连接的共同属性。

实际上，多对多关系需要两步连接。
首先 `products` 表和 `products_countries` 表连接起来，然后 `products_countries` 和 `countries` 连接起来。
SQLAlchemy 这里会做一些隐式的工作来实现。

可以打印出实际 SQL 语句看看具体实现 `print(query)`:

```sql
SELECT products.id, products.name, products.manufacturer_id, products.year, products.cpu, count(countries.id) AS count_1
FROM products JOIN products_countries AS products_countries_1 ON products.id = products_countries_1.product_id JOIN countries ON countries.id = products_countries_1.country_id GROUP BY products.id, products.name, products.manufacturer_id, products.year, products.cpu
HAVING count(countries.id) > :param_1 ORDER BY products.name
```

一对一关系和多对多关系可以一起使用，从而产生更加有趣的查询。
下一个查询会获取在 UK 生产过的制造商：

```Python
query = (
    select(Manufacturer)
    .join(Manufacturer.products)
    .join(Product.countries)
    .where(Country.name == 'UK')
    .order_by(Manufacturer.name)
    .distinct()
)

session.scalars(query).all()
```

该查询目的是得到一个制造商列表，因此 `select()` 里面只有一个对象。
但是查询需要访问国家数据，而国家与制造商之间没有之间关联。
唯一的方法是遍历现有关系，直到创建连接。
在这种情况下，第一个制造商和产品关联起来，然后产品和国家关联起来。
该连接的结果是，该查询可以访问到 制造商、产品、国家 的三元组，并可以在上面添加任何过滤器。
例如，只保留国家是 UK 的三元组。

`distinct()` 语句在查询中是有必要的，因为很多国家是 Uk 的三元组都有一样的制造商。
如果不指明，则会全部返回为多行，导致重复结果。

分组也可以应用于链式关联。
下面这个查询获取在多个国家生产过的制造商列表，以及国家数量：

```Python
country_count = func.count(Country.id.distinct()).label(None)
query = (
    select(Manufacturer, country_count)
    .join(Manufacturer.products)
    .join(Product.countries)
    .group_by(Manufacturer)
    .having(country_count > 1)
)

session.execute(query).all()
```

`country_count` 聚合函数需要使用 `distinct()` 子句进行扩展，因为简单的行计数会包含由连接操作产生的重复条目。

例如下面这个示例：

| Manufacturer | Product | Country |
| :----------- | :------ | :------ |
| Acme         | A       | USA     |
| Acme         | B       | USA     |

查询中的 `group_by(Manufacturer)` 语句会将这两行合并为一行，因为他们有相同的 `Manufacturer` 叫做 Acme。
一个简单的 `func.count()` 函数作为查询的第二个值，会进行统计计数。
这个例子中结果是 2，尽管只有一个国家。
`distinct()` 语句要求数据库移除重复的数量。

### Deleting From Many-To-Many Relationships

配置了 `secondary` 选项的多对多关系的优势在于，SQLAlchemy 会自动处理连接表的所有维护工作，包括删除操作。
当删除一个实体时，SQLAlchemy 会找到关系另一边的链接。

下面的例子会删除一个 country，这会使得所有的相关产品一同被移除。

```Python
c = session.get(Country, 22)
c
# Country(22, "Portugal")
p = session.get(Product, 138)
p
# Product(138, "Timex Sinclair 1000")
p.countries
# [Country(1, "UK"), ...]
session.delete(c)
session.commit()
p.countires
# [Country(1, "UK"), Country(3, "USA")]
```

同时也可以断开实体的这种多对多关系，无需删除任何实体。
该分离操作可以通过列表语义从关系的任意一方发起：

```Python
c = session.get(Country, 1)
c
# Country(1, "UK")

p = session.get(Product, 138)
p
# Product(138, "Timex Sinclair 1000")

c in p.countries
# True

p.countries.remove(c)
session.commit()
# [Country(3, "USA")]

c in p.countries
# False
```

上面的例子从国家中删除了一个产品，也可以通过产品来删除国家：

```Python
c.products.remove(p)
```

制造商和产品之间的一对多关系被设置为不可为空的外键，来强制所有产品都有制造商。
而在多对多关系中，并无法自动确保每个实体至少与另一侧的实体存在一个链接。
下面的例子只移除在该国家生产的产品，保留不是该国家的产品：

```Python
c = session.get(Country, 1)
c
# Country(1, "UK")

p = session.get(Product, 1)
p
# Product(1, "Acorn Atom")

p.countries
# [Country(1, "UK")]

c.products.remove(p)
session.commit()
p.countries()
# []
```

对于多对多关系，若必须至少存在一个关联实体，应用程序需手动执行验证检查，并防止移除最后一个关联链接。

### Database Migrations

由于 `drop_all()` 和 `create_all()` 这类方法会移除所有的表，并且需要重新导入数据。
因此这种数据库迁移方式十分受限制，下面介绍如何通过 Alembic 进行数据库迁移。

#### Introduce to Alembic

首先安装

```console
uv add alembic
```

然后初始化迁移仓库：

```console
alembic init migrations
```

`migrations` 参数是项目目录下面的子目录名称，用于存储迁移脚本。
此外，还会在项目目录下创建 `alembic.ini` 文件。
对于一个源码控制的的项目，`alembic.ini` 文件和 `migrations` 子目录都应该作为源码对待。

通过 `alembic init` 命令初始化的文件并不知晓当前项目使用的数据库情况。
为了使用 Alembic 指明项目的数据库，需要做出一些简单的配置修改。
一般是要编辑 `migrations/env.py` 文件。

在 `env.py` 文件中，从 `db.py` 导入 `engine` 和 `Model`：

```Python
from db import Model, engine
import models
```

然后定位到这一行

```Python
target_metatada = None
```

并将其修改为

```Python
target_metadata = Model.metadata
config.set_main_option(
    'sqlalchemy.url',
    engine.url.render_as_string(hide_passowrd=False),
)
```

`target_metadata` 变量是应用使用的元数据实例。
对于 SQLAlchemy ORM 来说这个 `target_metatada` 是 Model declarative base 类型的 metadata 属性。

第二行会在 Alembic 配置对象中为 `sqlalchemy.url` 选项插入一个值。
该选项是 Alembic 获取数据库 url 的地方。
由于应用已经创建了 engine 对象，通过该对象进入数据库是最便捷的方法。

打开 `alembic.ini` 文件，会看到同样的 `sqlalchemy.url` 选项被初始化为一个占位符 URL。
如果愿意，可以在该文件中直接输入 URL，不必须使用该函数。
使用 `env.py` 的好处就是不需要在不同地方配置数据库 URL。

还有一个需要修改的地方，对于 SQLite 而言尤其重要。
在 `env.py` 中找到 `run_migrations_online()` 函数，在接近文档末尾部分。
该函数会调用一个类似这样的函数：

```Python
context.configure(
    connection=connection,
    target_metatada=target_metatada,
)
```

此调用可用于配置 Alembic，特别是其迁移生成器，下面显示的更改启用了 `render_as_batch` 选项。

```Python
context.configure(
    connection=connection,
    target_metatada=target_metatada,
    render_as_batch=True,
)
```

该选项用于解决 SQLite 的迁移限制。
当该选项开启后，如果有 SQLite 不支持的迁移请求，Alembic 会隐式地创建新修改后的表，并将所有的数据都移动到新表下，最后删除旧的表。
对于其他数据库，该选项没有任何作用，因此可以省略该部分。

在文件前面有一行看上去并没有使用的 `import models`。
如果使用代码检测器，它可能将该导入标记为非必须。
无论代码检测工具怎么说，这行代码都是必须的。
SQLAlchemy 只有在 import 之后程序才能知道这些信息。
如果没有导入，这些模型就不会在 Python 的内存中存在，SQLAlchemy 和 Alembic 就无法知道关于它们的信息。

#### Create a Migration Script

Alebmic 使用迁移脚本来追踪数据库的变化。
一个迁移脚本包含了对当前数据库修改的代码，而不用移除任何表或数据。

Alembic 可以通过对比当前数据库表和模型类的区别，而自动生成迁移脚本。
例如，模型中存在但是数据库中不存在表，Alembic 会认为这是一个新表并需要创建。
如果一个表在数据库中存在，但是在没有被任何模型类引用，则会认为该表被删除了。
然后生成对应的迁移脚本，生成的迁移脚本目标始终是使数据库进行必要的更改，从而反应模型的状态。

为了生成一个初始迁移，需要数据库是空的，这样 Alembic 会创建对应的表。

```Python
from db import Model, engine
import models
Model.metadata.drop_all(engine)
```

这里的 `import models` 就是之前说的，需要通过导入来让 SQLAlchemy 知道有哪些模型。
允许上面语句后，数据库就会被清空。
现在就可以通过下面命令生成数据库迁移脚本了：

```console
alembic revision --autogenerate -m "products, manufacturers, countries"
```

alembic 会在终端输出一些日志，指明检测到新的表和索引。
然后会在底部显示生成的迁移脚本，会生成一个 `{code}_products_manufacturers_countries.py` 格式的文件。
`{code}` 是一个唯一的迁移标记，剩下的名称根据 `-m` 选项生成。

生成的迁移脚本是一个 Python 模块，有 `upgrade()` 和 `downgrade()` 这两个函数。
`upgrade()` 函数对数据库进行修改，而 `downgrade()` 函数会撤消这些修改。
每个迁移脚本都会有这两个函数，使得 Alembic 能够进行链式更新，或通过调用链取消。

Alembic 使用的自动迁移方法非常方便，但并不是万无一失。
因此最好审查一下脚本，确认他们是正确的。

#### Database Upgrade

`alembic revision` 命令会创建一个迁移脚本，但它并不会执行脚本。
这时候数据库还是空的，下一条命令运行迁移脚本：

```Python
alembic upgrade head
```

这条命令会去执行 `upgrade()` 函数，现在所有表的创建方式和 `create_all()` 函数类似。
Alembic 还可以使用 `downgrade` 命令来撤消数据库迁移，通过 `current` 来展示当前数据库迁移的情况。

#### A Migration-Aware Product Importer

该数据库迁移并不了解程序先要存储的数据，因此表都是空的。
`import_products.py` 脚本可以用于插入产品，制造商和国家数据，但是 `main()` 函数开头的 `drop_all()` 和 `create_all()` 函数需要移除。

现在的导入脚本会首先尝试删除所有的列，然后再从 csv 文件中导入，如下：

```Python
import csv
from sqlalchemy import delete
from db import Session
from models import Product, Manufacturer, Country, ProductCountry

def main():
    with Session() as session:
        with session.begin():
            session.execute(delete(ProductCountry))
            session.execute(delete(Product))
            session.execute(delete(Manufacturer))
            session.execute(delete(Country))

    with Session() as session:
        with session.begin():
            with open('products.csv') as f:
                reader = csv.DictReader(f)
                all_manufacturers = {}
                all_countries = {}

                for row in reader:
                    row['year'] = int(row['year'])

                    manufacturer = row.pop('manufacturer')
                    countries = row.pop('country').split('/')
                    p = Product(**row)

                    if manufacturer not in all_manufacturers:
                        m = Manufacturer(name=manufacturer)
                        session.add(m)
                        all_manufacturers[manufacturer] = m
                    all_manufacturers[manufacturer].products.append(p)

                    for country in countries:
                        if country not in all_countries:
                            c = Country(name=country)
                            session.add(c)
                            all_countries[country] = c
                        all_countries[country].products.append(p)

if __name__ == '__main__':
    main()
```

首先通过 SQLAlchemy 的 `delete()` 函数删除 `products`, `manufacturers` 和 `countries` 以及 `products_countries` 表中的数据。
剩下其他部分没有变化，现在可以运行脚本导入数据了。

```Python
uv run python import_products.py
```

### Exercises

1. 在 UK 或 USA 制造的产品

```Python
query = (
    select(Product)
    .join(Product.countries)
    .where(Country.name.in_(['UK', 'USA']))
)
session.scalars(query).all()
```

或者不进行连接，使用 `any` 子查询

```Python
query = (
    select(Product)
    .where(Product.countries.any(
        Country.name.in_(['UK', 'USA'])
    ))
)
session.scalars(query).all()
```

2. 不在 UK 或 USA 制造的产品.

```Python
query = (
    select(Product)
    .where(~Product.countries.any(
        Country.name.in_(['UK', 'USA'])
    ))
)
```

如果要使用连接

```Python
query = (
    select(Product.id)  # 只查 id
    .join(Product.countries)
    .where(not_(Country.name.in_(['UK', 'USA'])))
    .distinct()
)
query = session.scalars(query).all()
```

3. 任何拥有基于 Z80 CPU 产品的国家

```Python
query = (
    select(Country)
    .join(Country.products)
    .where(Product.cpu.like('%Z80%'))
    .distinct()
)
```

4. 在 1970 年代有产品的国家，并根据字母表排序

```Python
query = (
    select(Country)
    .join(Country.products)
    .where(Product.year.between(1970, 1979))  # BETWEEN 是双闭区间
    .order_by(Country.name)
    .distinct()
)
```

5. 拥有最多产品的 5 个国家，如果出现平局则按照字母表排序

```Python
product_count = func.count(Product.id).label(None)
query = (
    select(Country, product_count)
    .join(Country.products)
    .group_by(Country)
    .order_by(product_count.desc(), Country.name)
    .limit(5)
)
```

6. 在 UK 或 USA 有 3 个以上产品的制造商

```Python
product_count = func.count(Product).label(None)
query = (
    select(Manufacturer, product_count)
    .join(Manufacturer.products)
    .join(Product.countries)
    .where(Country.name.in_(['UK', 'USA']))
    .group_by(Manufacturer)
    .having(product_count > 3)
)
```

7. 在不止一个国家有产品的制造商

```Python
product_count = func.count(Product.id).label(None)
query = (
    select(Manufacturer)
    .join(Manufacturer.products)
    .join(Product.countries)
    .group_by(Manufacturer)
    .having(product_count > 1)
)
```

8. 应该和美国联合制造的产品

```Python
query = (
    select(Product)
    .join(Product.Countries)
    .where(Country.name.in_(['UK', 'USA']))
    .group_by(Product)
    .having(func.count(Country.id) > 1)
)
```

这里的技巧是，使用 `where()` 过滤出在 UK 和 USA 生产的产品。
然后统计产品的国家数量，如果数量大于 1，则是这两个国家都有的产品。

