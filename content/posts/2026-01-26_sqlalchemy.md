+++
date = '2026-01-26T8:00:00+01:00'
draft = true
title = 'SQLALchemy - Database Tables'
tags = ['Python', 'Database']
+++

### SQLALchemy Core and SQLALchemy ORM

SQLALchemy 分位两个模块：Core 和 ORM (Object-Relational Mapping)。
Core 模块包含对所有受支持数据库方言的集成逻辑，一组用于描述数据库表的类，用于 Python 语言生成 SQL 语句。
ORM 模块在 Python 应用程序中引入了一层抽象，使得许多数据库操作可以根据对 Python 对象执行的操作自动推导出来。

### Database Engine

SQLALchemy 使用 engine 对象来管理数据库连接，包含 Core 和 ORM 应用。
`create_engine()` 函数通过数据库 url 创建一个 engine。

格式为：

```
{dialet}{+driver}://{username}:{password}@{hostname}:{port}/{database}
```

其中 SQLite 比较特殊，无需 driver。

对于导入 DATABASE_URL 有两种方式，一种是使用 `load_dotenv()` 加载 .env 文件的环境变量

```Python
import os
from dotenv import load_dotenv
from sqlalchemy import create_engine


load_dotenv()

engine = create_engine(os.environ['DATABASE_URL'])
```

另一种是通过 Pydantic `BaseSettings`

```Python
import os
from pathlib import Path
from pydantic_settings import BaseSettings
from pydantic impot SecretStr
from functools import lru_cache
from sqlalchemy import create_engine


class Settings(BaseSettings):
    POSTGRESQL_HOST: str
    POSTGRESQL_PORT: int
    POSTGRESQL_USER: str
    POSTGRESQL_PASSWORD: SecretStr
    POSTGRESQL_DB: str

    @property
    def postgres_db_url(self) -> str:
        # 通常无需显示地写 driver，只有更换 dirver 时才需要写
        return f'postgresql://{self.POSTGRESQL_USER}:{self.POSTGRESQL_PASSWORD.get_secret_value()}@{self.POSTGRESQL_HOST}:{self.POSTGRESQL_PORT}/{self.POSTGRESQL_DB}'

    class Config:
        env_file = str(Path(__file__).parent / '.env')
        case_sensitive = True
        extra = 'ignore'  # 忽略额外字段

@lru_cache
def get_settings() -> Settings:
    return Settings()

# 使用示例
settings = get_settings()
print('Host:', settings.POSTGRESQL_HOST)
print('Password:', settings.PASSWORD.get_secret_value())
engine = create_engine(settings.postgres_db_url)
```

`create_engine()` 函数有以下配置参数：

- `echo=True` 日志记录每个 SQL 语句，用于调试非常有用。

- `pool_size=<N>` 指定连接池大小，默认最多 5 个

- `max_overflow=<N>` 流量高峰期最多可以创建的超过连接池连接的大小，默认为 10

- `future=True` 告诉 SQLALchemy 1.4 使用 2.0 的新版 APIs

- `pool_recycle=<N>` 设置链接回收时间，单位秒。mysql 默认 8 小时，推荐设置一小时(3600)。

- `pool_pre_ping=True` 在每次从池中取出连接前先 ping 一下，检查是否活着，解决数据库偶尔重启掉线的问题，提高健壮性

### Model

当使用 ORM 模块时，数据库表在 Python 类中定义。
该程序需要为所有的类创建一个父类，以便配置所有表共享的设置。
父类在 SQLALchemy 中被称为 `declarative` 基类，通常叫做 `Model` 或 `Base`。
Model 类的子类集和代表了数据库的结构或模式，通常被称为应用程序的 “模型”。

Model 类必须从 SQLALchemy 的 `DeclartiveBase` 类继承。
下面是一个没有额外配置的示例：

```Python
# db.py
import os
from dotenv import load_dotenv
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclartiveBase

class Model(DeclartiveBase):
    pass

load_dotenv()
engine = create_engine(os.env['DATABASE_URL'])
```

下面使用一个新文件存 Product 表的创建：

```Python
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column
from db import Model

class Product(Model):
    __tablename__ = 'products'

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))
    manufacturer: Mapped[str] = mapped_column(String(64))
    year: Mapped[int]
    country: Mapped[str] = mapped_column(String(32))
    cpu: Mapped[str] = mapped_column(String(32))

    def __repr__(self):
        return f'Product({self.id}, "{self.name}")'
```

Model 子类通过类属性定义：

`__tablename__` 属性定义数据库表名称，常见命名规范是表名称使用复数小写形式，这和模型名称形成对比，即使用单数的 camel case 驼峰命名法。

剩余定义的属性是表中的列，`Mapped[t]` 类型声明用于定义每个列，其中 `t` 是 Python 类型，例如 `int`, `str` 或 `datetime`。
对于 year 这样的普通列来说完全足够了，如果列需要额外的功能，就要将其赋值给一个 `mapped_column()` 构造器。

在上面的 `Product` 模型中，可以选择将 id 列作为一个 primary key 主键，其值必须唯一。
SQLALchemy 将主键设置为从 1 开始的自增列。

剩下的 str 类型通过 `String()` 添加了最大长度作为补充。
并非所有数据库都要求一定长度，但最好设置一下。

最后 `__repr__()` 方法是 Python 的特殊方法，告诉该对象如何打印。
该方法是可选的，但在调试和 shell 环境中使用的时候会很有用。

要创建模型类的实例，需使用标准构造函数，并以关键字参数形式传入模型属性的值。

```Python
c64 = Product(name='Commodore 64', manufacturer='Commodore')
```

上面代码会初始化一个新的 Product 实例，这个例子中除了指定的 name 和 manufacturer 之外都会被设置为 None。
尽管这个对象是一个模型实例，但此刻它只是一个普通的 Python 对象，尚未被存储在数据库中。

模型类的概念只适用于 ORM 模块，当使用 Core 时需要使用 Table 类来表示数据库表。

### Database Metadata

SQLALchemy 将所有数据库表的定义维护在一个 `MetaData` 对象中。
出于方便，它会初始化一个声明基类作为 MetaData 的一个属性。
对于 Model 类型，可以通过 `Model.metadata` 获取该值。
当一个 model class 例如 Product 定义的时候，SQLALchemy 会在对应的属性创建定义。

默认的 MetaData 配置存在一个总要限制，当项目达到一定规模或复制度时，这个限制必然会导致问题。
这与 `naming_convention` 相关，该选项用于指示 SQLALchemy 如何命名在数据库中创建的索引和约束。

默认的命名规范为 `MetaData` 索引提供了默认命名，但没有为 constraints 约束提供。
因此 SQLALchemy 不会以特定名称初始化它们，导致数据库使用任意名称。
当一个 constraint 约束需要被修改或删除的时候会导致问题，因为 SQLALchemy 无法通过约束名称来定位它。

为了避免可能出现的复杂情况，Model declarative base 声明基类可以通过更完整的命名约束进行初始化。

```Python
import os
from dotenv import load_dotenv
from sqlalchemy import create_engine, MetaData
from sqlalchemy.orm import DeclarativeBase


class Model(DeclarativeBase):
    metadata = MetaData(naming_convention={
        'ix': 'ix_%(column_0_label)s',
        'uq': 'uq_%(table_name)s_%(column_0_name)s',
        'ck': 'ck_%(table_name)s_%(constraint_name)s',
        'fk': 'fk_%(table_name)s_%(column_0_name)s_(referred_table_name)s',
        'pk': 'pk_%(table_name)s',
    })

load_dotenv()
engine = create_engine(os.environ['DATABASE_URL'])
```

MetaData 对象有一个 `create_all()` 方法，该方法具有重要作用，因为他会创建与已定义模型关联的所有数据库表。

```Python
Model.metadata.create_all(engine)
```

`create_all()` 方法向由 `engine` 表示的数据库发出 SQL 语句，以创建所有模型引用的数据库表。
该方法的局限性是，它只会创建数据库。
这意味着，当一个模型类型被修改了，该方法不能用于将变更传输至相应的数据库表。

一个修改现存表的变通方法是在调用 `create_all()` 之前，删除旧版的表，然后创建新版本的表。
对应的 MetaData 有一个 `drop_all()` 方法，该方法会从数据库删除所有的表。

```Python
Model.metadata.drop_all(engine)
Model.metadata.create_all(engine)
```

不幸的是，这样会导致数据丢失，后面会使用 Alembic 来做数据库迁移。

如果使用 Core 则必须手动创建数据库 metadata 对象。

### Sessions

另一个基于 ORM 的重要概念是 Session。
一个 Session 对象管理一系列示例的新建、读取、修改和删除。

在会话中累计的更改，会在会话刷新 flush 时，通过数据库事务 transaction 的上下文传递到数据库中。
这一操作在大多数情况下由 SQLALchemy 在需要时自动触发。
一次刷新操作会将修改写入数据库，但保持数据库事务处于打开状态。

当会话提交后，对应的数据库事务也会提交，造成数据库的永久修改。
数据库事务是关系数据库最重要的优势之一，旨在保护数据的完整性。

下面示例展示了如何将 c64 对象加入数据库：

```Python
from sqlalchemy.orm import Session

with Session(engine) as session:
    try:
        session.add(c64)
        session.commit()
    except:
        session.rollback()
        raise
    print(c64)
```

管理数据库会话 session 的最佳方式就是通过上下文管理器 context manager。
这能确保会话正确地关闭和清理。

Session 通过 engine 对象进行初始化。
类似地，也可以使用 `future=True` 参数将 SQLALchemy 1.4 的 API 设置为 2.0 版本。

Session 对象被设计成累计修改，直到提交或回滚。
`add()` 方法用来在会话中插入一个新对象，`try/block` 块保证 session 总是提交成功或回滚。

之前说过 SQLALchemy 默认设置主键列为整数自增的。
当会话刷新时，数据库会为新增条目的 `id` 属性分配下一个可用数字。
若是首次条目，则会分配数字 1。
任何其他没有设定的属性都会在数据库中被记录为 NULL。

SQLALchemy 提供了一个更加简洁地 session 交互方式。
一个正常的应用中会有许多地方需要创建会话，如果每次都必须传递 engine 和其他选项，就不太方便。
`sessionmaker` 工厂函数提供了一种创建定制化 Session 类的方法：

```Python
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(engine)
with Session() as session:
    # ...
```

将所有的逻辑都放到 `try/except` 块中仍会显得乏味。
下面的例子在上下文管理器中使用 `begin()` 替代异常处理：

```Python
with Session() as session:
    with session.begin():
        session.add(c64)
    print(c64)
```

通过 `session.begin()` 创建的上下文管理器在内部实现了 `try/except` 逻辑。
如果发生了异常，则会回滚到之前的状态。
下面是一个完整的创建会话的示例：

```Python
import os
from dotenv import load_dotenv
from sqlalchemy import create_engine, MetaData
from sqlalchemy import DeclarativeBase, sessionmaker

class Model(DeclarativeBase):
    metadata = MetaData(naming_convention={
        'ix': 'ix_%(column_0_label)s',
        'uq': 'uq_%(table_name)s_%(column_0_name)s',
        'ck': 'ck_%(table_name)s_%(constraint_name)s',
        'fk': 'fk_%(table_name)s_%(column_0_name)s_(referred_table_name)s',
        'pk': 'pk_%(table_name)s',
    })

load_dotenv()
engine = create_engine(os.environ['DATABASE_URL'])
Session = sessionmaker(engine)
```

### Queries

#### Query Definition

SQLALchemy 提供了 `select()` 函数来实现 SELECT 关键字一样的查询功能。
假设通过 ORM 定义了一个 `Product` 表，下面来查询表中所有元素：

```Python
from sqlalchemy import select
q = select(Product)
```

`select()` 函数接收需要检索的条目作为参数。
当以模型类作为参数时，SQLALchemy ORM 会自动获取该模型的所有属性，并以透明方式返回 Python 对象。
可以打印查看 SQL 查询语句：

```Python
print(q)
# SELECT products.id, products.name, productsroducts.manufacturer, products.year, products.country, products.cpu
# FROM products
```

#### Query Execution

当查询对象创建后，需要将其传递给 session，这将通过 engine 维护的连接将其发送到数据库驱动程序中执行。
最常见的方法是使用 `execute()` 命令：

```Python
r = session.execute()
list(r)
# [(Product(1, Acorn Atom),),
#  (Product(2, BBC Micro),),
#  ...,
# (Product(149, GEM 1000),)]
```

`execute()` 方法返回一个可迭代的结果对象，通过 list() 可以将其转换为列表并展示。

- `execute()` 执行结果对象可以使用 `all()` 方法获取和 list() 一样的结果：

  ```Python
  session.execute(q).all()
  ```

- `first()` 返回结果的第一行，如果没有结果返回 `None`

- `one()` 只返回第一个结果，如果有零或多余一个结果，则抛出异常

- `one_or_none()` 只返回第一个结果，如果没有结果返回 `None`，如果超过一个结果则抛出异常

获取 iterable 可迭代的结果十分高效，SQLALchemy 只会迭代需要的行。
这意味着，对于很大结果的查询，无需将所有内容存储在列表中，而可以只处理加载的结果。

```Python
r = session.execute(q)
for row in r:
    print(row)
```

注意处理得到的结果格式：

```Python
(Product(1, "Acorn Atom"),)
```

这不是单纯的一个 `Product` 对象，而是一个元组，因为查询有时候可能会每行返回多个结果。
如果确定返回结果是单个元素，则可以使用 `scalars()` 来执行查询：

```Python
session.scalars(q).all()
# [Product(1, Acorn Atom), ..., Product(149, GEM 1000)]
```

使用该方法时会返回一个不同的结果对象，该对象只包含每行的第一个值。
如果查询结果每行有多个值，则多余的值会被丢弃。
除了 `all()` 其他的方法例如 `first()`, `one()`, `one_or_none()` 也都能使用 `scalars()` 方法的结果对象。

下面还有一些额外的方法(q 为查询)：

- `scalar(q)` 和 `scalars(q).first()` 相同
- `scalar_one(q)` 和 `scalars(q).one()` 相同
- `scalar_one_or_none(q)` 和 `scalars(q).one_or_none()` 相同

#### Filters

`select(Table)` 的查询会返回所有项，但实际上并不常用。
很多时候实际上只需要迭代某些项，可以使用 filter 过滤器来只迭代满足要求的项。
在选择后面添加 `where()` 方法：

```Python
# 筛选特定供应商的产品
q = select(Product).where(Product.manufacturer == 'Commodore')
session.scalars(q).all()
# [Product(39, PET),
#  ...
#  Product(48, Amiga)]

# 筛选 1990 年后的产品
q = select(Product).where(Product.year >= 1990)
```

`where()` 可以多次使用指定多个条件，下一个示例仅检索 Commodore 在 1980 年生产的产品：

```Python
q = (select(Product)
        .where(Product.manufacturer == 'Commodore')
        .where(Product.year == 1980)
)
```

多个条件也可以写在一起

```Python
q = select(Product).where(Product.manufacturer == 'Commodore', Product.year == 1980)
```

这样会使用 AND 逻辑将多个条件结合起来，有时候查询可能希望使用 OR 逻辑将操作结合。
SQLALchemy 提供了 `or_()` 函数来实现，下面例子是去过滤在 1970 年之前或 1970 年之后的产品。

```Python
q = select(Product).where(
    or_(Product.year < 1970, Product.year > 1990)
)
```

当然也可以显示使用 `and_()` 函数，此外还有 `not_()` 函数。

另一个很有用的过滤是 LIKE 操作，该操作用来实现简易的搜索函数。
下面例子是寻找产品名称中有 Sinclair 产品：

```Python
q = select(Product).where(Product.name.like('%Sinclair%'))
```

列属性上面的 `like()` 方法接收一个搜索模式字符串，并返回所有匹配该模式的结果。
该模式定义了要搜索的文本，其中使用 `%` 字符作为通配符，可匹配零个、一个或多个字符。
并使用 `_` 来匹配单个字符。

下面是一些例子：

- `Sinclair%` 以 Sinclair 开头的项
- `%Sinclair` 以 Sinclair 结尾的项
- `% Sinclair` 以 空格+Sinclair 结尾的项
- `R__%` 以 R 开头并跟随两个字符的项
- `_` 单字符长的项

`like()` 函数是大小写敏感的，如果希望忽略大小写形式，应该使用 `ilike()` 函数。

可以通过 `where()` 子句指定下限和上限两个条件来请求一些列条目。
但更清晰的做饭是使用模型类列属性提供的 `between()` 方法。
例如，下面例子返回 1970 年代生产的产品：

```Python
q = select(Product.year.between(year, 1970, 1979))
```

如果要查看生生的 SQL，可以直接打印出来

```Python
print(q)
# SELECT products.id, products.name, products.manufacturer, products.year, products.country, products.cpu
# FROM products
# WHERE products.year BETWEEN :year_1 AND :year_2
```

此处可见，查询过滤器中定义的文字并未插入到渲染后的 SQL 语句中。
相反，他们由 `:year_1` 和 `:year_2` 这样的占位符替代。
这是防止 SQL 注入的良好实践，SQLAlchemy 会自动实现。

出于调试目的，可能希望查询包含实际字面值的 SQL 查询。
尽管这种做法可能不安全，但下面展示了如何指示 SQLAlchemy 在渲染查询时，同时显示所有字面量参数：

```Python
print(q.compile(compile_kwargs={'literal_binds': True}))
# SELECT products.id, products.name, products.manufacturer, products.year,products.country, products.cpu
# FROM products
# WHERE products.year
# BETWEEN 1970 AND 1979
```

#### Order of Results

上面查询结果是按照数据库选择的顺序返回的，但关系数据库可以直接将结果高效地进行排序。
`order_by()` 方法可以用来对结果进行排序。

```Python
q = select(Product).order_by(Product.name)
session.scalars(q).all()
```

也可以使用 `desc()` 让其逆序排序，例如下面例子按年份逆序排序：

```Python
q = select(Product).order_by(Product.name.desc())
session.scalars(q).all()
```

有时候单一的排序标准是不够的。
例如，在最后结果中会有一些计算机都是同一年制造的，这些计算机之间就是任意排序了。
`order_by()` 方法接受多个参数，每个参数会添加一层排序。

例如在之前的例子上，添加一层按产品名称排序：

```Python
q = select(Product).order_by(Product.year.desc(), Product.name.asc())
```

注意其中的 `asc()` 方法，是按照产品名称升序排序，有时候这样写会更加清晰可读。

#### Access to Individual Columns

在之前的所有查询例子中，都是返回一整行的每一项。
`select()` 方法实际上十分灵活，可以选择指定的项。

```Python
q = select(Product.name)
session.scalars(q).all()
# [ 'Acorn Atom', ..., 'GEM 1000' ]

q = select(Product.name, Product.manufacturer)
session.execute(q).all()
# [('Acorn Atom', 'Acorn Computers Ltd'), ('BBC Micro', 'Acorn Computers Ltd'), ...]
```

#### Aggregation Function

Aggregation Function: 聚合函数

`select()` 函数也可以配合 SQL 函数使用，这些 SQL 函数会对检索到的数据进行即时求值，从而实现数据转换。
一个很有用的函数是 `count()`，它会将所有结果行替换为数量。
下一个例子阐明了如何查询数据库中数据的数量：

```Python
from sqlalchemy import func

q = select(func.count(Product.id))
r = session.scalar(q)
# 149
```

上面使用的 `count()` 函数将结果降低为单个值，`scalar()` 方法用于检索它。
在这个例子中，使用 `Product.id` 作为参数是任意的，任何 Product 类的列都可以，结果一样，因为数据本身不重要。
下面是不需要任何列的结果统计方式：

```Python
q = select(func.count()).select_from(Product)
r = session.scalar(q)
# 149
```

在这种形式，`count()` 函数没有给予任何参数。
使用此格式时，必须添加 `select_from()` 方法来配置查询中使用的表。
因为 SQLALchemy 无法根据传递给 `select()` 函数的参数自动确定该表。

还有一队有用的 SQL 函数 `min()` 和 `max()`。
以下示例返回数据库中产品制造的首个和最后年份：

```Python
q = select(func.min(Product.year), func.max(Product.year))
r = session.execute(q)
r.first()
# (1969, 1995)
```

该方法要使用 `execute()` 因为有两个结果。
`min()` 和 `max()` 函数会将结果缩减为单行，因此像之前示例那样使用 `all()` 来获取结果已无必要。
当预先确定查询结果只有单行时，使用 `first()` 或 `one()` 方法更为便捷，其中后者会在查询时返回非单行时抛出异常。

#### Result Grouping

当前数据库仅将 product 作为首要实体，但有时应用程序要检索相关数据属性，例如制造商。
下面是获取制造商列表的尝试：

```Python
q = select(Product.manufacturer).order_by(Product.manufacturer)
session.scalars(q).all()
# ['Acorn Computers Ltd', 'Acorn Computers Ltd', ..., 'West Computer AS']
```

但显然这里存在一个问题。
尽管查询只检索制造商信息，但查询的表包含产品数据，因此每个结果行对应一个产品。
如果某家制造商在数据库中有多个产品，他就会多次出现。
以字母顺序排列时，排名首位的制造商 Acorn Computers Ltd 就连续出现六次。

每当数据库查询可能返回重复结果时，添加 `distinct()` 子句会指示数据库合并相同的结果。

```Python
q = select(Product.manufacturer).order_by(Product.manufacturer).distinct()
session.scalars(q).all()
# ['Acorn Computers Ltd', 'AGAT', ..., 'West Computer AS']
```

你可能会想将 `distinct()` 子句与 `count()` 聚合函数结合使用，以查明数据库中存在多少制造商。
不幸的是，`distinct()` 函数无法与 `count()` 聚合函数结合使用，因为数据库会在 `distinct()` 前计算数量。

如果要计算唯一结果的数量，可以在 `count()` 函数内部对要计数的对象调用 `distinct()` 方法。

```Python
q = select(func.count(Product.manufacturer.distinct()))
r = session.scalar(q)
# 76
```

使用 `distinct()` 来合并结果的能力很有限，因为只有当两行结果完全相同时，数据库才会把它们折叠成一行。
举例来说，这种方式无法 “列出每个厂商以及它最早和最晚活跃年份”。
因为一旦把 `Product.year` 作为第二个字段加入查询，不同年份会让行变得不再相同，从而改变 `distinct()` 能合并哪些结果。

`group_by()` 函数提供了更加灵活的结果分组方案。
上述返回制造商列表的查询同样可以通过 `group_by()` 实现：

```Python
q = (select(Product.manufacturer)
        .group_by(Product.manufacturer)
        .order_by(Product.manufacturer))
```

这样的结果相同，但使用 `group_by()` 可以添加额外的查询列，只要通过函数将每个组的数据聚合成单一值。
下一个示例将获取制造商列表，包括其运营的起始与结束年份，以及他们生产的型号数量：

```Python
q = (select(
        Product.manufacturer,
        func.min(Product.year),
        func.max(Product.year),
        func.count()
    )
    .group_by(Product.manufacturer)
    .order_by(Product.manufacturer)
)

session.execute(q).all()
# [('Acorn Computers Ltd', 1980, 1995, 6), ..., ('West Computer AS', 1984, 1984, 1)]
```

上面执行逻辑是：

- 分组 (Group By): 将 Product 根据 manufacturer 把数据扔进不同的“桶” 里
- 聚合 (Aggregation): 每个桶内计算 `func.min/max/count`
- 选择 (Select): 根据选择要求把结果挑出来
- 排序 (Order By): 根据排序要求，把结果排序

> DISTINCT 对“结果整行”去重，GROUP BY 是按某些列建立分组

在前面已经介绍过，`where()` 方法可以用于筛选查询返回的结果集，并会在结果分组前进行计算。
因此该子句无法用于筛选分组后的结果。
`having()` 子句用于筛选分组和聚合后的结果，下面是一个示例，用于获取拥有 5 个或更多型号的制造商列表及其实际数量：

```Python
q = (select(
        Product.manufacturer,
        func.count()
    )
    .group_by(Product.manufacturer)
    .having(func.count() >= 5)
    .order_by(Product.manufacturer)
)

session.execute(q).all()
# [('Acorn Computers Ltd', 6), ('Amstrad', 7), ..., ('Timex Sinclair', 6)]
```

你可能注意到在这个例子中 `count()` 出现了两次。
首先在 `select()` 部分出现一次，然后在 `having()` 方法中又出现一次。
为了确保 `count()` 只用编写一次，使用 `label()` 方法计算结果关联一个标签，随后在需要该数据的两个位置直接引用此标签即可。

```Python
num_products = func.count().label(None)
q = (select(
        Product.manufacturer,
        num_products
    )
    .group_by(Product.manufacturer)
    .having(num_products >= 5)
    .order_by(Product.manufacturer)
)
```

`label()` 方法的参数是标签的名称，当传入 `None` 时由 SQLALchemy 自动生成，确保选取唯一名称。
让 SQLALchemy 自动命名是可行的，关键在于将标签实例赋值给 `num_products` 变量。
当然，如果希望为标签指定名称，也同样允许：

```Python
num_products = func.count().label('num_products')
```

#### Pagination

对于很长的查询结果，一个常见的方式是限制最大数量。
`limit()` 方法用于设置查询的最大返回数量。
交互式或网络应用中，用户通常可以按指定页面大小的增量向前或向后流览结果。

实现查询结果的分页功能，需使用 `limit()` 方法设定每页大小，并指定从何处开始获取数据。
选择开始位置最简单的方法是在查询中添加 `offset()` 方法。
该方法用于检索结果的起始索引，上述例子查询可通过添加 `offset(0)` 进行泛化。
若要获取包含三条结果的第二页数据，可按如下方式构架查询：

```Python
q = select(Product.order_by(Product.name).limit(3).offset(3))
session.execute(q).all()
# [Product(131, "Aamber Pegasus"), Product(84, "ABC 80"), Product(5, "Acorn Archimedes")]
```
