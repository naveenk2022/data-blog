---
title: "Data Model Version Control with Alembic."
date: "2026-01-13"
tags: ['Python', 'PostgreSQL', 'SQLAlchemy', 'Alembic', 'ORM', 'Database Design', 'Database Migrations', 'Tutorial']
description: "Developing data models with version control using Alembic and Alembic-Utils."
author: ["Naveen Kannan"]
weight: 10
draft: true
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

As the scope of a data model (and by extension, it's downstream APIs) changes, it will need to be updated and expanded to account for this new scope.

When changes are made to a data model, especially to a model that is split across development and production environments, schema drift becomes a constant problem that looms in the background.

Typical coding workflows are managed by version control as an industry standard. Data model development, however, is typically not submitted to a version control system. As different team members collaborate on a data model split across different environments, the possibility of schema drift gradually grows in an environment where the data model is not tracked in a central repository.

This is where Alembic shines! It is the de-facto migration tool for SQLAlchemy, and is the ideal tool to version control a data model's development.

In this blog post, we will pick up from the progress in the previous entry in this series and continue to develop our data model. We will incorporate PostgreSQL entity types such as functions, views, materialized views, triggers, and policies into our data model, and commit all of these changes to version control using Alembic and Alembic-Utils.

## What we've covered

In our [previous blog post:](posts/sqlalchemy_data_modeling)

- We used SQLAlchemy to create the first iteration of a rudimentary data model, with three tables.
- We configured Alembic to connect to a fresh development database and migrated our rudimentary data model.

Please refer to the first post in this series for a detailed explanation of SQLAlchemy and Alembic, and how it is relevant in our workflows here.

## What we'll cover

TODO

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

There are a lot of changes we need to make to make this data model usable. To start with:

- A connection needs to be built between orders and products. A single order can have multiple products, and a single product can be associated with multiple orders.
- Products need to be associated with tags to enable better categorization within the product catalog. Each product can be associated with multiple tags, and
a single tag can be associated with multiple products.
- We also want to display when the data associated with a product listing was updated.

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

    product_count = Column(Integer)
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
> Remember that there is **direct** connection between `Order` and `Product` in the data model!

**For the `Product` Class, this code:**

- Creates a relationship called `orders`, that points **towards** the `OrderProduct` class. This now establishes a one-to-many relationship between `Product` and `OrderProducts`. A single `Product` can be referenced by **multiple** instances of `OrderProduct`.

- Similar to the `products` relationship defined in `Order`, this is a conceptual representation, which is why it is called `orders`.

By wiring these connections, we abstract away the many-to-many relationship between `Orders` and `Products`.

> [!TIP]
> The `back_populates` parameter establishes a bidirectional link between the two ORM relationships. It ensures that objects on both sides of each relationship synchronize in-Python state changes.