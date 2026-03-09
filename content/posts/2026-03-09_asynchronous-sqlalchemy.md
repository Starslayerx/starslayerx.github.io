+++
date = '2026-03-09T8:00:00+01:00'
draft = true
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
