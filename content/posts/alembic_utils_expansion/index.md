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

We want to build a many-to-many relationship between `orders` and `products`. To accomplish that, we will create an **association table.**

>[!TIP]
> Association tables are also referred to as join tables, junction tables or cross-reference tables.

An association table maps two (or more) tables together by using their primary keys as foreign keys. Each foreign key forms a **many-to-one** relationship between the association table and the individual data table. An association table creates a two-way link between the tables that represent each entity.

The best way to demonstrate this is by example! Let's begin by creating an association table to group together the products associated with each order.

To generate an association table, the following changes need to be made to `models.py`. Additions have been highlighted.

**`models.py`**:

```python {hl_lines=["32-36", "59-61","74-76"]}
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

order_products = Table(
    "order_products",
    Base.metadata,
    Column("order_id", ForeignKey("orders.order_id"), primary_key=True),
    Column("product_id", ForeignKey("products.product_id"), primary_key=True),
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
    products = relationship(
        "Product", secondary=order_products, back_populates="orders"
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
    orders = relationship(
        "Order", secondary=order_products, back_populates="products"
    )
```

In the SQLAlchemy ORM, a many-to-many relationship is implemented by using the relationship() function in combination with Table constructs to define the association table, and secondary keyword argument to define the association table that helps map the relations.