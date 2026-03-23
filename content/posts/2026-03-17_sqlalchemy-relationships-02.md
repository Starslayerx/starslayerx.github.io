+++
date = '2026-03-17T8:00:00+01:00'
draft = true
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
