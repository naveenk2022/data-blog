---
title: "Data Model Version Control with Alembic."
date: "2026-01-26"
tags: ['Python', 'PostgreSQL', 'SQLAlchemy', 'Alembic', 'ORM', 'Database Design', 'Database Migrations', 'Tutorial']
description: "Developing version-controlled relational data models using Alembic."
author: ["Naveen Kannan"]
weight: 10
draft: false
ShowToc: true
TocOpen: true
cover:
  image: "https://www.sqlalchemy.org/img/sqla_logo.png"
  alt: "<alt text>"
  caption: "The SQLAlchemy logo."
  relative: false
  hidden: false
  hiddenInList: true
  hiddenInSingle: false
params:
  ShowCodeCopyButtons: true
  ShowReadingTime: true
---

# Introduction

As the scope of a data model (and by extension, its downstream APIs) changes, it will need to be updated and expanded to account for this new scope.

When changes are made to a data model, especially to a model that is split across development and production environments, schema drift becomes a constant problem that looms in the background.

Typical coding workflows are managed by version control as an industry standard. Database schemas, however, are typically not submitted to a version control system. As different team members collaborate on a data model split across different environments, the possibility of schema drift gradually grows in an environment where the data model is not tracked in a central repository.

This is where Alembic shines! It is the de-facto migration tool for SQLAlchemy, and is the ideal tool to version control a data model's development.

In this blog post, we will pick up from the progress in the previous entry in this series and continue to develop our data model. We will build more tables and relationships into our data model, and commit all of these changes to version control using Alembic.

## What we've already covered

In our [previous blog post:](posts/sqlalchemy_data_modeling)

- We used SQLAlchemy to create the first iteration of a rudimentary data model, with three tables.
- We configured Alembic to connect to a fresh development database and migrated our rudimentary data model.

Please refer to the first post in this series for a detailed explanation of SQLAlchemy and Alembic, and how it is relevant in our workflows here.

## What we'll cover in this post

- Creating a **many-to-many** association between orders and products.
- Creating a **many-to-many** association between products and tags.
- Committing all of our changes to version control with Alembic.

# Getting started

At this point, we have a rudimentary data model with three tables, and only two relationships.

There's `orders`,`customers` and `products`, with a one-to-many relationship between `customers` and `orders`, and a many-to-one relationship between `orders` and `customers`.

{{< figure
    src="../sqlalchemy_data_modeling/basic_data_model_erd.png"
    caption="Our initial data model."
    align=center
>}}

The classes for these tables in our ORM were defined as follows:

```python
class Customer(Base):
    __tablename__ = "customers"

    customer_id = Column(
        Integer, Identity(always=True), primary_key=True
    )
    first_name = Column(String, nullable=False)
    last_name = Column(String, nullable=False)
    email_address = Column(String, nullable=False)

    orders = relationship(
        "Order",
        back_populates="customer",
        cascade="all, delete-orphan",
    )

class Order(Base):
    __tablename__ = "orders"

    order_id = Column(
        Integer, Identity(always=True), primary_key=True
    )
    customer_id = Column(
        Integer,
        ForeignKey("customers.customer_id", ondelete="CASCADE"),
        nullable=False
    )
    order_date = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )

    customer = relationship(
        "Customer",
        back_populates="orders"
    )

class Product(Base):
    __tablename__ = "products"

    product_id = Column(
        Integer, Identity(always=True), primary_key=True
    )
    product_name = Column(
        String, nullable=False
    )
    description = Column(Text)
    price = Column(Float, nullable=False)
```

>[!NOTE]
>We generated an initial migration, but right now, this migration environment lives locally. We need to push this to a GitHub repository!
>
>The data model at the end of the first blog post has been pushed to GitHub and can be [found here](https://github.com/naveenk2022/commerce_data_model).
>
> This repository will evolve along with the blog! Every change we make moving forward will be committed to version control.


There are a lot of changes we need to make to make this data model usable. To start with:

- A connection needs to be built between orders and products. A single order can have multiple products, and a single product can be associated with multiple orders.
- Products need to be associated with tags to enable better categorization within the product catalog. Each product can be associated with multiple tags, and
a single tag can be associated with multiple products.

>[!TIP]
> This change in relationship between orders and products is a **many-to-many** relationship!
>
> When we expand our data model to include tags, the relationship between tags and products will also be a **many-to-many** relationship.

Let's begin implementing this change in scope within the data model. With each step, we will also commit our work to version control using Alembic.

## Many-to-many relationship between `orders` and `products`

We want to build a many-to-many relationship between `orders` and `products`. We also want to expand this relationship to track the **count** of each product in an order.

To accomplish that, we will create an **association table.**

>[!TIP]
> Association tables are also referred to as join tables, junction tables or cross-reference tables.

An association table maps two (or more) tables together by using their primary keys as foreign keys. Each foreign key forms a **many-to-one** relationship between the association table and the individual data table. An association table creates a two-way link between the tables that represent each entity.

The best way to demonstrate this is by example! Let's begin by creating an association table to group together the products associated with each order.

To generate an association table, the following changes need to be made to `models.py`. Additions have been highlighted.

**`models.py`**:

```python {hl_lines=["32-48", "67-71","81-86"]}
from sqlalchemy import (
    Column,
    Integer,
    String,
    Float,
    ForeignKey,
    Identity,
    Text,
    DateTime,
)
from sqlalchemy.orm import declarative_base, relationship
from sqlalchemy.sql import func

Base = declarative_base()


class Customer(Base):
    __tablename__ = "customers"

    customer_id = Column(Integer, Identity(always=True), primary_key=True)
    first_name = Column(String, nullable=False)
    last_name = Column(String, nullable=False)
    email_address = Column(String, nullable=False)

    orders = relationship(
        "Order",
        back_populates="customer",
        cascade="all, delete-orphan",
    )


class OrderProduct(Base):
    __tablename__ = "order_products"
    order_id = Column(
        Integer,
        ForeignKey("orders.order_id", ondelete="CASCADE"),
        primary_key=True,
    )

    product_id = Column(
        Integer,
        ForeignKey("products.product_id", ondelete="CASCADE"),
        primary_key=True,
    )

    product_count = Column(Integer, server_default="1", nullable=False)
    order = relationship("Order", back_populates="products")
    product = relationship("Product", back_populates="orders")


class Order(Base):
    __tablename__ = "orders"

    order_id = Column(Integer, Identity(always=True), primary_key=True)
    customer_id = Column(
        Integer, ForeignKey("customers.customer_id", ondelete="CASCADE"), nullable=False
    )
    order_date = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    customer = relationship(
        "Customer",
        back_populates="orders",
    )
    products = relationship(
        "OrderProduct",
        back_populates="order",
        cascade="all, delete-orphan",
    )


class Product(Base):
    __tablename__ = "products"

    product_id = Column(Integer, Identity(always=True), primary_key=True)
    product_name = Column(String, nullable=False)
    description = Column(Text)
    price = Column(Float, nullable=False)
    orders = relationship(
        "OrderProduct",
        back_populates="product",
        cascade="all, delete-orphan",
    )
```

### Creating the association table

We create a class to define the association table, since the association table contains an additional column beyond just the associations with `Order` and `Product`. This table is `OrderProduct`.

The`order_id` and `product_id` columns in this table are designated as **foreign keys**.

We then create associations between `OrderProduct` and the entity tables using the `relationship` parameter. `OrderProducts` has relationships built to both `Order` and `Product`.

### Explaining the relationship between `Order` and `Product`

There is no **direct  relationship** between `Order` and `Product`. This relationship is mediated by the `OrderProduct` class, which is an association table that decomposes the relationship between `Order` and `Product`.

A single instance of an `Order` can belong to multiple rows of `OrderProduct`.

A single instance of a `Product` can belong to multiple rows of `OrderProduct`.

However, every instance of `OrderProduct` can only have:

- A **single** `Order`

- A **single** `Product`.

In essence, the many-to-many relationship between `Order` and `Product` is abstracted into being indirect via `OrderProduct`. Each row in `OrderProduct` represents a single product within a single order, along with additional metadata about that association, such as the product count for that order.

The following code snippets establish this indirect relationship between `Order` and `Product`.

**`OrderProduct`**:
```python
    order = relationship("Order", back_populates="products")
    product = relationship("Product", back_populates="orders")
```

**`Order`**:
```python
products = relationship(
        "OrderProduct",
        back_populates="order",
        cascade="all, delete-orphan",
    )
```

**`Product`**:
```python
orders = relationship(
        "OrderProduct",
        back_populates="product",
        cascade="all, delete-orphan",
    )
```

**For the `OrderProduct` Class, this code:**

- Creates a relationship called `order`, that points **towards** the `Order` class. This now establishes a many-to-one relationship between `OrderProduct` and `Order`. Each instance of `OrderProduct` references exactly **one** `Order`.

- Creates a relationship called `product`, that points **towards** the `Product` class. This now establishes a many-to-one relationship between `OrderProduct` and `Product`. Each instance of `OrderProduct` references exactly **one** `Product`.

**For the `Order` Class, this code:**

- Creates a relationship called `products`, that points **towards** the `OrderProduct` class. This now establishes a one-to-many relationship between `Order` and `OrderProducts`. A single `Order` can be referenced by **multiple** instances of `OrderProduct`.

> [!TIP]
> Even though this relationship points to `OrderProduct`, it is named `products`.
> This is intentional, and it refers to the conceptual representation of this relationship, not the literal schema itself.
>
> - Conceptually:
> An order has **multiple products.**
> - Relationally:
> An `Order` can be referenced by multiple `OrderProduct` rows.
>
> This relationship is named `products` to reflect the *conceptual* link between orders and products, and not the direct *relational* link between `Orders` and `OrderProducts`.
>
> Remember that there is no **direct** connection between `Order` and `Product` in the data model!

**For the `Product` Class, this code:**

- Creates a relationship called `orders`, that points **towards** the `OrderProduct` class. This now establishes a one-to-many relationship between `Product` and `OrderProducts`. A single `Product` can be referenced by **multiple** instances of `OrderProduct`.

- Similar to the `products` relationship defined in `Order`, this is a conceptual representation, which is why it is called `orders`.

By wiring these connections, we abstract away the many-to-many relationship between `Orders` and `Products`.

> [!TIP]
> The `back_populates` parameter establishes a bidirectional link between the two ORM relationships. It ensures that objects on both sides of each relationship synchronize in-Python state changes.

# Auto-generating migrations

We now have a end state of our data model defined in the ORM. To generate a migration, we need to allow Alembic to compare the current state of the database against the ORM, and have it automatically generate a migration script that we **must review** before applying.

We configured and set up Alembic in the previous blog post, so this time, we just need to run the following command:

```bash
alembic revision --autogenerate -m "Created OrderProduct as an association table to track Order-Product association."
```

This should autogenerate a migration script, located at `/path_to_your_project/alembic/versions/960969e75871_created_orderproduct_as_an_association_.py`. The `versions` directory will have the following content:

```bash
alembic/versions/
└── a0cf7f5d705d_initial_commit_created_customers_orders_.py
└── 960969e75871_created_orderproduct_as_an_association_.py
```
The `versions` directory now contains two migration scripts! One is the migration script for our initial commit in the previous blog post, and one is the newly generated script for creating the OrderProducts table.

>[!NOTE]
> In the previous blog post, the initial commit migration script had the ID `de5f32c11361`, and not `a0cf7f5d705d`.
>
>The reason for this change is that I created a new revision script for the initial migration, and the revision ID is randomly generated!
>
> Other aspects of the migration script are otherwise exactly the same, and the scripts are otherwise functionally identical.

The newly generated migration should have the following content:

```python
"""Created OrderProduct as an association table to track Order-Product association.

Revision ID: 960969e75871
Revises: a0cf7f5d705d
Create Date: 2026-01-26 17:30:18.314422

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = '960969e75871'
down_revision: Union[str, None] = 'a0cf7f5d705d'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table('order_products',
    sa.Column('order_id', sa.Integer(), nullable=False),
    sa.Column('product_id', sa.Integer(), nullable=False),
    sa.Column('product_count', sa.Integer(), server_default='1', nullable=False),
    sa.ForeignKeyConstraint(['order_id'], ['orders.order_id'], ondelete='CASCADE'),
    sa.ForeignKeyConstraint(['product_id'], ['products.product_id'], ondelete='CASCADE'),
    sa.PrimaryKeyConstraint('order_id', 'product_id')
    )
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_table('order_products')
    # ### end Alembic commands ###
```

# Reviewing the auto-generated migration

Remember that Alembic's documentation states,
> It is **always** necessary to manually review and correct the **candidate migrations** that autogenerate produces.

Reviewing our migration script, it looks like:

- Alembic has created a table called `order_products`.
- `order_id` and `product_id` are both **foreign keys**. They are the primary keys from the tables `orders` and `products` respectively.
- `order_id` and `product_id` are also **primary keys** for the `order_products` table.
- `order_products` has a column called `product_count`, with a default value of 1.

>[!TIP]
>`order_id` and `product_id` are both selected as primary keys to ensure that duplicate rows won’t be persisted within the `order_products` regardless of issues on the application side.

This migration script looks good! We can go ahead and apply it to the database.

>[!NOTE] Why does the migration script look so sparse compared to the complex ORM relationships?
> **`Order`**:
> ```
> products = relationship(
>         "OrderProduct",
>         back_populates="order",
>         cascade="all, delete-orphan",
>     )
> ```
> This code defines the behavior within the **ORM**, and will become relevant when you interact with the database via SQLAlchemy.
> It defines the behaviour of these classes within Python and also documents the relationship on a conceptual level.
> It **doesn't create anything within the database**.
>
> On the database level, it is really straightforward to build this relationship,
> which is why the migration script seems so sparse in comparison.
> Within the migration script, on the database level, we're just creating `order_products`
> as a table with foreign keys pointing to the parent entity tables.
> The relational aspect of the data model is much lower level than the conceptual data model, which is what we define in the ORM.
>
> The ORM helps us build a conceptual data model with rich, complex relationships as seen in our code.
> Alembic reads the ORM and generates migrations to create a lower-level relational data model **from** the ORM.

# Applying the migration

We can then apply the migration script with the command:

```bash
alembic upgrade head
```

This should give us the following output:

```bash
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade a0cf7f5d705d -> 960969e75871, Created OrderProduct as an association table to track Order-Product association.
```

Connecting to the database shows the following ERD:

{{< figure
    src="erd_diagram_orderproducts.png"
    caption="Our newly iterated data model!"
    align=center
>}}

We've successfully created the association table, and we have built an indirect **many-to-many** association between `Orders` and `Products`! When we examine the contents of the `alembic_version` table, it now contains a single row with the value `960969e75871`, which is the revision it has applied. It applied this one specifically because we had alembic upgrade to the `head` version, which is the latest version.

# Committing to GitHub

At this point, we've made changes to `models.py` and we have a new migration script called `960969e75871_created_orderproduct_as_an_association_.py`. Both these changes need to be pushed GitHub. I recommend committing and pushing these changes as a single commit with the same commit message that we used for auto-generating a migration script.

Remember to always commit migration scripts **together with** model changes.

# Further iterating

Let's expand this data model more. A great addition to this data model would be the ability to create and assign tags to products for greater searchability.

To start with, these tags will be really simple. A tag will be a text-based descriptor that can be applied to any product. For example, a laptop as a product might have a tag called 'computer' assigned to it.

To reflect this, we will create a table containing tags, and build a **many-to-many** relationship between products and tags. Each product can have multiple tags, and each tag can be assigned to multiple products. For this, like before, we will build an association table.

We will do the following:
- Create a class for `Tags` in the ORM
- Create an association table called `product_tags` to associate products and tags
- Update the relationships for `Products` to reflect the new relationship to the `Tags` class.

>[!TIP]
> In the interest of brevity, this section won't be as thoroughly explanatory as the section for creating `OrderProducts`.
> This process is very similar to what we've covered thus far!

## Creating a `Tags` class

**`models.py`** (Changes to code have been highlighted):

```python {hl_lines=["10", "75-84","99","102-107"]}
from sqlalchemy import (
    Column,
    Integer,
    String,
    Float,
    ForeignKey,
    Identity,
    Text,
    DateTime,
    Table,
)
from sqlalchemy.orm import declarative_base, relationship
from sqlalchemy.sql import func

Base = declarative_base()


class Customer(Base):
    __tablename__ = "customers"

    customer_id = Column(Integer, Identity(always=True), primary_key=True)
    first_name = Column(String, nullable=False)
    last_name = Column(String, nullable=False)
    email_address = Column(String, nullable=False)

    orders = relationship(
        "Order",
        back_populates="customer",
        cascade="all, delete-orphan",
    )


class OrderProduct(Base):
    __tablename__ = "order_products"
    order_id = Column(
        Integer,
        ForeignKey("orders.order_id", ondelete="CASCADE"),
        primary_key=True,
    )

    product_id = Column(
        Integer,
        ForeignKey("products.product_id", ondelete="CASCADE"),
        primary_key=True,
    )

    product_count = Column(Integer, server_default="1", nullable=False)
    order = relationship("Order", back_populates="products")
    product = relationship("Product", back_populates="orders")


class Order(Base):
    __tablename__ = "orders"

    order_id = Column(Integer, Identity(always=True), primary_key=True)
    customer_id = Column(
        Integer, ForeignKey("customers.customer_id", ondelete="CASCADE"), nullable=False
    )
    order_date = Column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False,
    )
    customer = relationship(
        "Customer",
        back_populates="orders",
    )
    products = relationship(
        "OrderProduct",
        back_populates="order",
        cascade="all, delete-orphan",
    )


product_tags = Table(
    "product_tags",
    Base.metadata,
    Column(
        "product_id",
        ForeignKey("products.product_id", ondelete="CASCADE"),
        primary_key=True,
    ),
    Column("tag_id", ForeignKey("tags.tag_id", ondelete="CASCADE"), primary_key=True),
)


class Product(Base):
    __tablename__ = "products"

    product_id = Column(Integer, Identity(always=True), primary_key=True)
    product_name = Column(String, nullable=False)
    description = Column(Text)
    price = Column(Float, nullable=False)
    orders = relationship(
        "OrderProduct",
        back_populates="product",
        cascade="all, delete-orphan",
    )
    tags = relationship("Tag", secondary=product_tags, back_populates="products")


class Tag(Base):
    __tablename__ = "tags"

    tag_id = Column(Integer, Identity(always=True), primary_key=True)
    name = Column(String, nullable=False, unique=True)
    products = relationship("Product", secondary=product_tags, back_populates="tags")
```

>[!NOTE]
> The `product_tags` implementation here is different from the `OrderProducts` class.
>
> This is because `product_tags` is a straightforward, simple association table,
> which is what the documentation for SQLAlchemy says to use when your association table has no metadata.
>
> This `secondary` table pattern is only usable when all you have in the table are foreign keys.
> `product_tags` has no attributes apart from its foreign keys, so we use the association table with the `secondary` table pattern.
>
> The reason `OrderProducts` was mapped as a class was because we have additional metadata beyond just the foreign keys in that table.
>
> `OrderProducts` goes from just a simple association table to a full entity in it's own right when you add even a single extra column beyond the foreign keys.
> The recommended approach for adding metadata to an association table is to map the association table directly to a full class.


## Generating a migration

We create an auto-generated migration script with the following command:

```bash
alembic revision --autogenerate -m "Created a table for tags, and created a table called product_tags as an association table between products and tags."
```

## Reviewing the migration script

The contents of the migration script are:

```python
"""Created a table for tags, and created a table called product_tags as an association table between products and tags.

Revision ID: 6e58a26e5c70
Revises: 960969e75871
Create Date: 2026-01-26 18:32:43.007353

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = '6e58a26e5c70'
down_revision: Union[str, None] = '960969e75871'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table('tags',
    sa.Column('tag_id', sa.Integer(), sa.Identity(always=True), nullable=False),
    sa.Column('name', sa.String(), nullable=False),
    sa.PrimaryKeyConstraint('tag_id'),
    sa.UniqueConstraint('name')
    )
    op.create_table('product_tags',
    sa.Column('product_id', sa.Integer(), nullable=False),
    sa.Column('tag_id', sa.Integer(), nullable=False),
    sa.ForeignKeyConstraint(['product_id'], ['products.product_id'], ondelete='CASCADE'),
    sa.ForeignKeyConstraint(['tag_id'], ['tags.tag_id'], ondelete='CASCADE'),
    sa.PrimaryKeyConstraint('product_id', 'tag_id')
    )
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_table('product_tags')
    op.drop_table('tags')
    # ### end Alembic commands ###
```

This migration script (with the revision ID `6e58a26e5c70`) looks good!

You should now have three scripts inside `alembic/versions`.

```bash
alembic/versions/
├── 6e58a26e5c70_created_a_table_for_tags_and_created_a_.py
├── 960969e75871_created_orderproduct_as_an_association_.py
└── a0cf7f5d705d_initial_commit_created_customers_orders_.py
```

## Applying the migration script

Apply the migration script by running:

```bash
alembic upgrade head
```

Connecting to the database shows the following ERD:

{{< figure
    src="erd_diagram_producttags.png"
    caption="Tags are now associated with Products!"
    align=center
>}}

We've successfully built a connection between Products and Tags via an association table!

Browsing through `alembic/versions` should give us an idea of the growth of our data model and how it evolves to match the change in scope of our e-commerce workflow. With these three migration scripts, we've gone from a rudimentary data model to a more complex one, while being able to track how the data model has evolved from its conception.

# Next Steps

In subsequent blog posts, we will cover:

- Using an Async engine for asynchronous database connections
- Using the `alembic_utils` library to incorporate views, functions and triggers into the version control system.
- Configuring Alembic to connect to a production environment, and seamlessly bring the production environment's schema up-to-date with our final revision.


# References

- SQLAlchemy's [documentation on many-to-many relationships.](https://docs.sqlalchemy.org/en/14/orm/basic_relationships.html#many-to-many)
- Alembic's [documentation on auto-generating migrations.](https://alembic.sqlalchemy.org/en/latest/autogenerate.html#what-does-autogenerate-detect-and-what-does-it-not-detect)