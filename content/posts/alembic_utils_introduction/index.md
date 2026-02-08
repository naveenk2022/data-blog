---
title: "PostgreSQL entity type Version Control with Alembic Utils."
date: "2026-02-04"
tags: ['Python', 'PostgreSQL', 'SQLAlchemy', 'Alembic', 'ORM', 'Database Design', 'Database Migrations', 'Tutorial']
description: "Committing PostgreSQL entity types to version control using Alembic Utils."
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

PostgreSQL (and other RDBMS) have entity types such as [functions](https://www.postgresql.org/docs/current/sql-createfunction.html), [views](https://www.postgresql.org/docs/current/sql-createview.html), [materialized views](https://www.postgresql.org/docs/current/sql-creatematerializedview.html), [triggers](https://www.postgresql.org/docs/current/sql-createtrigger.html), and [policies](https://www.postgresql.org/docs/current/sql-createpolicy.html). These entity types do a lot of useful work, and a well designed database will use them heavily.

These entity types are not typically considered part of the SQLAlchemy ORM, and by extension, the data model itself. By default, Alembic has no functionality to detect the creation of new entity types when it autogenerates migration files. The default Alembic ORM also has no functionality to define these entity types with classes.

However, these entity types are part of a database's schema. Creation of/changes to them should be accounted for within the ORM and as part of migration version control, in much the same way as changes in a table are recorded in an ORM as part of migration version control.

In this blog post, we will pick up from the progress in the previous entry in this series and go over how we can use an extension to Alembic called [Alembic Utils](https://github.com/olirice/alembic_utils) to add support for autogenerating migrations for some PostgreSQL entity types.

## What we've already covered

In [part 1:](posts/sqlalchemy_data_modeling)

- We used SQLAlchemy to create the first iteration of a rudimentary data model, with three tables.
- We configured Alembic to connect to a fresh development database and migrated our rudimentary data model.

In [part 2:](posts/alembic_model_expansion)

- We created a **many-to-many** association between orders and products, and a **many-to-many** association between products and tags.
- We generated migration scripts using Alembic.
- We committed the migration scripts to version control.

{{< figure
    src="../alembic_model_expansion/erd_diagram_producttags.png"
    caption="The data model at the end of the previous post."
    align=center
>}}

## What we'll cover

In this blog post, we will cover how you can install Alembic Utils and begin incorporating entity types into the commerce data model. 

We will do the following:
- Update the virtual environment with Alembic Utils.
- Configure Alembic to use the newly installed extension.
- Create a function and a view in the ORM.
- Register these entities with Alembic Utils.
- Autogenerate a migration script with Alembic that we will review, and then apply the revisions.

# Getting started

[Alembic Utils](https://github.com/olirice/alembic_utils) is an Alembic/SQLAlchemy extension written in Python with an MIT license. 

## Installing Alembic Utils

In [part 1](posts/sqlalchemy_data_modeling/#prerequisites) of this series, we created a micromamba virtual environment called `data_model` with SQLAlchemy and Alembic installed. 

We will activate this virtual envrionment and then install Alembic Utils.

First, activate the virtual environment. 

```bash
micromamba activate data_model
```

Then, install the Alembic Utils extension. 

```bash
pip install alembic_utils
```

Export the new environment details using the following command:

```bash
micromamba env export > environment.yaml
```

> [!NOTE]
> Updating your `environment.yaml` file any time your dependencies change is good practice.
> 
> It makes your virtual environment reproducible and accessible! 

## Creating a View in the ORM

To explain what a view is, we can start by asking a straightforward high level question. What are the names of the products associated with a particular order? The stakeholders who have access to data might need to be able to access the products associated with a single specific order, or a group of specific orders. 

In our database, there is a table called `order_products` that maps the associations between `orders` and `products` using the primary keys from that table. However, just looking at this table alone won't give us the contextual information we need. We will need to write an SQL query to join the `order_products` table with the `products` table in order to retrieve the name and description of a particular order's products.

This is a query that can be easily **generalizable**. A query can be written to join **all** the products referenced in `order_products` with the relevant product data from `products`. This query can then be filtered to return the associations with a single order or group of orders. 

This query can be abstracted into a view. Views are named queries that allow for data to be queried from it **like a regular table**. The view is not physically stored, however. The query in the view is run each time the view itself is referenced in a query. 

To return the product data associated with each `product_id` reference in `order_products`, we could write a query like this.

```sql
select 
  op.order_id, 
  op.product_id, 
  op.product_count, 
  p.product_name, 
  p.description, 
  p.price 
from 
  order_products op 
  join products p on op.product_id = p.product_id
```

We can add an adjustment to this query to return the total price of each product associated with an order when accounting for the order count.

```sql
select 
  op.order_id, 
  op.product_id, 
  op.product_count, 
  p.product_name, 
  p.description, 
  p.price as unit_price, 
  op.product_count * p.price as order_price 
from 
  order_products op 
  join products p on op.product_id = p.product_id
```

We multiply the product count with the unit price to get the total cost of the product within an order. With this query, we can now filter to a specific set of `order_id` values to return all the products associated with those orders and their total cost! 

Creating a view from a query like this, that has the potential to be referenced multiple times, is great database design. We can now add this view to our ORM. 

First, we will create a script called `db_views.py` in the root of the project. This script will be where we store the views that we create for our ORM. 
 
`db_views.py`:

```python
from alembic_utils.pg_view import PGView

order_line_items = PGView(
    schema="public",
    signature="order_line_items",
    definition="""
select op.order_id , 
	op.product_id, 
	op.product_count,
	p.product_name, 
	p.description, 
	p.price as unit_price, 
	op.product_count * p.price as order_price
	from order_products op 
join products p 
on op.product_id = p.product_id 
""",
)
```

With this code, we first create a `PGView` class object called `order_line_items`. This defines a PostgreSQL view also called `order_line_items` that references the query that we wrote to return the line items (products) associated with orders. 

## Registering the newly defined entity

We will update `alembic/env.py` to then register our newly created entity with alembic_utils. 

`alembic/env.py`:

```python {hl_lines=["8-10"]}
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool
from alembic_utils.replaceable_entity import register_entities
from alembic import context
from models import Base
from db_views import order_product_summary

register_entities([order_product_summary])

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = Base.metadata

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```