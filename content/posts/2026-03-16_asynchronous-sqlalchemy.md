+++
date = '2026-03-16T8:00:00+01:00'
draft = false
title = 'SQLALchemy - Asynchronous'
categories = ['Note']
tags = ['Python', 'Database']
+++

## Engines, Metadata and Sessions

下面是一个异步的数据库链接示例：

```Python
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, create_async_engine, async_sessionmaker


class Model(DeclarativeBase):
    metadata = MetaData(naming_convention={
        'ix': 'ix_%(column_0_label)s',
        'uq': 'uq_%(table_name)s_%(column_0_name)s',
        'ck': 'ck_%(table_name)s_%(constraint_name)s',
        'fk': 'fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s',
        'pk': 'pk_%(table_name)s',
    })

pg_dsn: str = 'postgresql+psycopg://postgresql:postgresql@localhost:5432/postgresql'
engine = create_async_engine(pg_dsn)
session_factory = async_sessionmaker(
    bind=engine,
    expire_on_commit=False,  # !!! prevent implicit synchronous refresh
)
```

可以看到与同步代码其实十分类似，一个重要的不同点是 session 的 `expire_on_commit` 参数。
这会禁止 SQLALchemy 的默认行为：在会话 session 提交后将模型 model 标记为过期。
标记为过期的模型再次访问其任何属性时，会隐式地从数据库查询中刷新。
由于隐式的 implicit 数据库活动不能出现在任何异步应用中，因此不应该使用过期对象。
`expire_on_commit=False` 选项确保在提交后，不会将任何模型标记为过期。

在异步高并发环境下，仅靠 `expire_on_commit` 可以保证程序不报错，但是不能保证数据的一致性。
在 long-lived session 中可能会有陈旧模型 stale model，下面是一些解决方法：

1. 手动清理模型 `expunge` 或显示刷新 `refresh`，但是会多产生一次网络请求

   ```Python
   # refresh example
   async def create_new_post(db: AsyncSession, title: str, content: str):
       new_post = Post(title=title, content=content)
       db.add(new_post)

       await db.commit()  # 提交到数据库
       await db.refresh() # 刷新：强制从数据库拉取最新的一行数据

       print(f'PostID: {new_post.id}, CreateTime: {new_post.created_at}')
       return new_post

   # expunge example
   async def export_all_posts_title_to_csv(db: AsyncSession):
       result = await db.execute(select(Post))
       posts = result.scalars().all()

       titles = []
       for post in posts:
           titles.append(post.title)
           db.expunge(post)  # 踢出 Session
       return titles
   ```

   这样可以保证数据正确写入，但可能覆盖其他人的数据修改

2. 乐观锁 Optimistic Locking

   通过在数据表中加一个 `version` 版本号或者 `updated_at` 时间戳来实现。
   1. 读取：首先查询数据，得到 `version=1`
   2. 修改：在内存中修改数据，准备提交
   3. 检验并提交：系统向数据库发起带条件的 SQL: `UPDATE user SET name='新名称', version=2 WHERE id=1 AND version=1;`
      如果在提交之前有其他人提交了，则 `version` 就会被修改，导致更新失败。

3. 悲观锁 Pessmisitic Locking

   悲观锁是数据库底层真实存在的排他锁 Exclusive Lock：
   使用数据库的 `SELECT ... FOR UPDATE` 语法，当事务执行语句时，会将这条数据锁住，其他人无法修改。

一般来说，乐观锁的性能更好，因为不会导致数据阻塞，业务中使用较多。
而悲观锁虽然性能较差，但是对于敏感数据十分有用，例如金额、库存等数据，保证其一致性防止冲突。

一个有趣的方面是，SQLAlchemy 的 `MetaData` 并没有异步版本。
当使用 `create_all()` 建表和 `drop_all()` 删表函数时，没有 await 的版本。
SQLAlchemy 提供了一个 `run_sync()` 方法，可以用来运行这类同步数据库代码，并"等待"他们。

```Python
async with engine.begin() as connection:
    await connection.run_sync(Model.metadata.drop_all)
    await connection.run_sync(Model.metadata.create_all)
```

## Relationships Loaders

在大多数情况下，模型定义无需为异步应用修改。
需要仔细检查的是关系加载器。

你可能注意到，许多模型中的 `relationship()` 属性使用懒加载机制，在属性首次被访问时查询数据库关系。
此外，还有懒加载参数，`options()` 查询子句，以及 `WriteOnlyMapped` 类型提示，都可以被用于修改该行为。
默认加载行为，映射为 `lazy='select'` 或 `options(lazyload(...))`，与异步应用并不匹配。
因此需要切换成一个更加可预测的加载器。

下表是所有可选的加载器：

| When          | Loaders                                 |
| :------------ | :-------------------------------------- |
| Lazy load     | select (default), dynamic (legacy)      |
| Eager load    | joined, selection, subquery             |
| Explicit load | write_only, noload, raise, raise_on_sql |

由于不能使用 Lazy loader，因此只剩下 Eager loader 和 Explicit loader 可选。

一个稳妥的做法是将所有的 lazy loading relationships 改为 `lazy='raise'`。
这样当 SQLAlchemy 需要延迟加载他们时，就会引发错误。
在此基础上，当需要加载关联关系时，应用可以显示地将 `options()` 子句中明确传递一个预加载器作为显示覆盖。

另一种方法时为所有的关系，选择一个合适的加载器，从而避免任何可能的懒加载。

- "一" 的一端 (Product) 使用 `joined` eager loader

  如果关系不是可选的 optional，则设置 `innerjoin=True` 使用 inner join，数据库可以使用更快的算法匹配。
  默认的 left join 则认为该字段可能为空 (optional)，需要将没有关系的字段也查找出来，速度略慢。

- "多" 的一端 (Manufacturer)，且数量巨大，使用 `write_only` 加载器

  当处理可能包含上万记录的关系时，使用 `write_only` 模式，不会将数据加载到内存中。
  它会返回一个查询构造器，需要显示访问，但不会 OOM 耗尽内存。

- "多" 的一端 (Manufacturer)，普通情况，使用 `selectin`

  该加载器会将查询拆解为两次 SQL 查询，而不是直接 join 产生笛卡尔积冗余。
  首先查询符合需求的 `manufacturer.id`，然后再用 `SELECT ... FROM product WHERE manufacturer.id IN product.manufacturer_id`，这样两次得到结果。

下面代码块展示了异步版本的 model.py 需要的修改：

```Python
class Product(Model):
    # ...
    manufacturer: Mapped['Manufacturer'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='products',
    )
    countries: Mapped[list['Country']] = relationship(
        lazy='selection',
        secondary=ProductCountry,
        back_populates='products',
    )
    order_items: WriteOnlyMapped['OrderItem'] = relationship(back_populates='product')
    blog_articles: WriteOnlyMapped['BlogArticle'] = relationship(back_populates='products')
    # ...


class Manufacturer(Model):
    # ...
    products: Mapped[list['Product']] = relationship(
        lazy='selection',
        cascade='all, delete-orphan',
        back_populates='manufacturer',
    )
    # ...


class Country(Model):
    # ...
    products: Mapped[list['Product']] = relationship(
        lazy='selection',
        casacde='all, delete-orphan',
        back_populates='countries',
    )
    # ...


class Order(Model):
    # ...
    customer: Mapped['Customer'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='orders',
    )


class Customer(Model):
    # ...
    orders: WriteOnlyMapped['Order'] = relationship(back_populates='customer')
    product_reviews: WriteOnlyMapped['ProductReview'] = relationship(back_populates='customer')
    blog_users: WriteOnlyMapped['BlogUser'] = relationships(back_populates='customer')
    # ...


class OrderItem(Model):
    # ...
    product: Mapped['Product'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='order_items',
    )
    order: Mapped['Order'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='order_items',
    )
    # ...


class ProductReview(Model):
    # ...
    product: Mapped['Product'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='product_reviews',
    )
    customer: Mapped['Customer'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='product_reviews',
    )
    # ...


class BlogArticle(Model):
    # ...
    author: Mapped['BlogAuthor'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='articles'
    )

    product: Mapped[Optional['Product']] = relationship(
        lazy='joined',
        back_populates='blog_articles'
    )

    views: WriteOnlyMapped['BlogView'] = relationship(
        back_populates='article'
    )

    language: Mapped[Optional['Language']] = relationship(
        lazy='joined',
        back_populates='blog_articles'
    )

    translation_of: Mapped[Optional['BlogArticle']] = relationship(
        lazy='joined',
        remote_side=id,
        back_populates='translations'
    )

    translations: Mapped[list['BlogArticle']] = relationship(
        lazy='selectin',
        back_populates='translation_of'
    )
    # ...


class BlogAuthor(Model):
    # ...
    articles: WriteOnlyMapped['BlogArticle'] = relationship(
        back_populates='author'
    )
    # ...


class BlogUser(Model):
    # ...
    customer: Mapped[Optional['Customer']] = relationship(
        lazy='joined',
        back_populates='blog_users'
    )

    sessions: WriteOnlyMapped['BlogSession'] = relationship(
        back_populates='user'
    )
    # ...


class BlogSession(Model):
    # ...
    user: Mapped['BlogUser'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='sessions'
    )

    views: WriteOnlyMapped['BlogView'] = relationship(
        back_populates='session'
    )
    # ...


class BlogView(Model):
    # ...
    article: Mapped['BlogArticle'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='views'
    )

    session: Mapped['BlogSession'] = relationship(
        lazy='joined',
        innerjoin=True,
        back_populates='views'
    )
    # ...


class Language(Model):
    # ...
    blog_articles: WriteOnlyMapped['BlogArticle'] = relationship(
        back_populates='language'
    )
    # ...
```
