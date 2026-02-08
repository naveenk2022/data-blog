---
title: "PostgreSQL Entity type Version Control with Alembic Utils."
date: "2026-02-08"
tags: ['Python', 'PostgreSQL', 'SQLAlchemy', 'Alembic', 'ORM', 'Database Design', 'Database Migrations', 'Tutorial']
description: "Committing PostgreSQL entity types to version control using Alembic Utils."
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

# Installing Alembic Utils

In [part 1](posts/sqlalchemy_data_modeling/#prerequisites) of this series, we created a micromamba virtual environment called `data_model` with SQLAlchemy and Alembic installed. 

We will activate this virtual envrionment and then install Alembic Utils.

First, activate the virtual environment. 

```bash
micromamba activate data_model
```

Then, install the Alembic Utils extension within the environment. The documentation recommends using `pip`.

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

# Adding a view to the database

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
from db_views import order_line_items

register_entities([order_line_items])

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

The additions to the code import the newly defined view and register it. This view should now be detected by `alembic revision --autogenerate`. 

## Autogenerating a migration

We now autogenerate a revision with the command:

```bash
alembic revision --autogenerate -m "Created a view called order_line_items that returns order line items."
```

This should create a new migration script located at `/path_to_your_project/alembic/versions`.

`/path_to_your_project/alembic/versions/e2ba9d70fc4d_created_a_view_called_order_line_items_.py`:

```python
"""Created a view called order_line_items that returns order line items.

Revision ID: e2ba9d70fc4d
Revises: 6e58a26e5c70
Create Date: 2026-02-07 19:26:28.816521

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
from alembic_utils.pg_view import PGView
from sqlalchemy import text as sql_text

# revision identifiers, used by Alembic.
revision: str = 'e2ba9d70fc4d'
down_revision: Union[str, None] = '6e58a26e5c70'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    public_order_line_items = PGView(
        schema="public",
        signature="order_line_items",
        definition='select op.order_id , \n\top.product_id, \n\top.product_count,\n\tp.product_name, \n\tp.description, \n\tp.price as unit_price, \n\top.product_count * p.price as order_price\n\tfrom order_products op \njoin products p \non op.product_id = p.product_id'
    )
    op.create_entity(public_order_line_items)

    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    public_order_line_items = PGView(
        schema="public",
        signature="order_line_items",
        definition='select op.order_id , \n\top.product_id, \n\top.product_count,\n\tp.product_name, \n\tp.description, \n\tp.price as unit_price, \n\top.product_count * p.price as order_price\n\tfrom order_products op \njoin products p \non op.product_id = p.product_id'
    )
    op.drop_entity(public_order_line_items)

    # ### end Alembic commands ###
```

This migration script now creates a PostgreSQL view with the name `order_line_items`. 

> [!NOTE]
> The `PGView` object here is called `public_order_line_items` as the autogenerate function uses the schema name as a prefix to the object name. In this case, the schema used is `public`. 

## Applying the migration

We now apply the migration with the command:

```bash
alembic upgrade head
```

Connecting to the database shows us that the view has been migrated to the database. 

{{< figure
    src="erd_diagram_view.png"
    caption="The newly migrated view."
    align=center
>}}

# Expanding the data model's scope 

We would like to now expand the data model to track when the attributes of a particular product have been edited. To do this, we will do the following:

- Add a `last_edited` timestamp column to the `products` table. 
- Create a function that will update the `last_edited` column of a table.
- Create a trigger that will call the aforementioned function whenever a row of `products` is edited.

## Adding a timestamp column to the `products` table

Amend the data model in `models.py` to have a `last_edited` timestamp column.

```python {hl_lines=["94-96"]}
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
    last_edited = Column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )
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
> [!NOTE]
> The `onupdate=func.now()` parameter for `last_edited` will update the column when changes are issued through the ORM.
>
> To guarantee database level correctness, we will need to create a trigger as well to guarantee data accuracy
> even when changes come from outside of any applications that use the ORM.

Generate a revision with `alembic --autogenerate`.

```python
"""Added a column to products to track timestamps for row-level edits.

Revision ID: eebe110ef9d2
Revises: e2ba9d70fc4d
Create Date: 2026-02-08 16:29:55.536261

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision: str = 'eebe110ef9d2'
down_revision: Union[str, None] = 'e2ba9d70fc4d'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column('products', sa.Column('last_edited', sa.DateTime(timezone=True), server_default=sa.text('now()'), nullable=True))
    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_column('products', 'last_edited')
    # ### end Alembic commands ###
```

We will then apply this migration with the `alembic upgrade head` command.

## Creating a function to update the `last_edited` column of `products`

Create a script called `db_functions.py` in the project root. 

`db_functions.py`:

```python
from alembic_utils.pg_function import PGFunction

## Creating a function that updates the `last_edited` column of a table
timestamp_update_on_edit = PGFunction(
    schema="public",
    signature="timestamp_update_on_edit()",
    definition="""
RETURNS TRIGGER AS $$ BEGIN NEW.last_edited = now();
RETURN NEW;
END;
$$ language PLPGSQL
""",
)
```

With this code, we create a `PGFunction` class object that defines a PostgreSQL function called `timestamp_update_on_edit()`. When this function is called, the `last_edited` column is updated to have the time and date of the point in time when it was called.

## Creating a trigger to call the timestamp function on row edits

Create a script called `db_triggers.py` in the project root.

`db_triggers.py`:

```python
from alembic_utils.pg_trigger import PGTrigger

## Creating a trigger to apply `timestamp_update_on_edit()` to `products` on a row being updated
timestamp_update_apply_products_trigger = PGTrigger(
    schema="public",
    signature="timestamp_update_apply_products_trigger",
    on_entity="public.products",
    is_constraint=False,
    definition="""
BEFORE 
UPDATE 
  on products FOR EACH ROW EXECUTE FUNCTION timestamp_update_on_edit()
""",
)
```

This code defines a `PGTrigger` class object with the signature `timestamp_update_apply_products_trigger`. When a row is updated on `products`, this trigger is activated and calls the `timestamp_update_on_edit()` function, which will update the `last_edited` column.

>[!NOTE]
> In the ORM, as part of the definition of the `Product` object, the `onupdate=func.now()` parameter 
> updates the `last_edited` column when changes are issued through the ORM. 
>
> When we create a trigger to perform the timestamp update for `products` in the database upon a row being edited, 
> this guarantees correctness on a database level even when the changes come outside of the application. 

## Registering the newly defined entities

We will update `alembic/env.py` to then register our newly created entities with alembic_utils. 

`alembic/env.py`:

```python {hl_lines=["9-18"]}
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool
from alembic_utils.replaceable_entity import register_entities
from alembic import context
from models import Base
from db_views import order_line_items
from db_functions import timestamp_update_on_edit
from db_triggers import timestamp_update_apply_products_trigger

register_entities(
    [
        order_line_items,
        timestamp_update_on_edit,
        timestamp_update_apply_products_trigger,
    ]
)

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

With this update, our newly created function and trigger have been registered. They should now be detected when using `autogenerate`.

## Autogenerating and applying a revision to migrate the newly added entities

Create a revision script with:

```bash
alembic revision --autogenerate -m "Created a function to update the last_edited timestamp column, and created a trigger to call this function for the products table."
```

This should generate a revision file with the following content:

```python
"""Created a function to update the last_edited timestamp column, and created a trigger to call this function for the products table.

Revision ID: 172f9e67a364
Revises: eebe110ef9d2
Create Date: 2026-02-08 16:41:47.178179

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
from alembic_utils.pg_function import PGFunction
from sqlalchemy import text as sql_text
from alembic_utils.pg_trigger import PGTrigger
from sqlalchemy import text as sql_text

# revision identifiers, used by Alembic.
revision: str = '172f9e67a364'
down_revision: Union[str, None] = 'eebe110ef9d2'
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    public_timestamp_update_on_edit = PGFunction(
        schema="public",
        signature="timestamp_update_on_edit()",
        definition='RETURNS TRIGGER AS $$ BEGIN NEW.last_edited = now();\nRETURN NEW;\nEND;\n$$ language PLPGSQL'
    )
    op.create_entity(public_timestamp_update_on_edit)

    public_products_timestamp_update_apply_products_trigger = PGTrigger(
        schema="public",
        signature="timestamp_update_apply_products_trigger",
        on_entity="public.products",
        is_constraint=False,
        definition='BEFORE \nUPDATE \n  on products FOR EACH ROW EXECUTE FUNCTION timestamp_update_on_edit()'
    )
    op.create_entity(public_products_timestamp_update_apply_products_trigger)

    # ### end Alembic commands ###


def downgrade() -> None:
    # ### commands auto generated by Alembic - please adjust! ###
    public_products_timestamp_update_apply_products_trigger = PGTrigger(
        schema="public",
        signature="timestamp_update_apply_products_trigger",
        on_entity="public.products",
        is_constraint=False,
        definition='BEFORE \nUPDATE \n  on products FOR EACH ROW EXECUTE FUNCTION timestamp_update_on_edit()'
    )
    op.drop_entity(public_products_timestamp_update_apply_products_trigger)

    public_timestamp_update_on_edit = PGFunction(
        schema="public",
        signature="timestamp_update_on_edit()",
        definition='RETURNS TRIGGER AS $$ BEGIN NEW.last_edited = now();\nRETURN NEW;\nEND;\n$$ language PLPGSQL'
    )
    op.drop_entity(public_timestamp_update_on_edit)

    # ### end Alembic commands ###
```
The migration script should create the function and the trigger that we defined in the ORM within the database. After reviewing it, we apply the migration with the command `alembic upgrade head`.

Once these changes have been migrated, run the following SQL code in the database to get the triggers associated with the `products` table.

```sql
SELECT 
    tgname AS trigger_name
FROM 
    pg_trigger
WHERE
    tgrelid = 'public.products'::regclass
ORDER BY
    trigger_name;
```

This should give you the output:

```
RI_ConstraintTrigger_a_16447
RI_ConstraintTrigger_a_16448
RI_ConstraintTrigger_a_16471
RI_ConstraintTrigger_a_16472
timestamp_update_apply_products_trigger
```

Indicating the trigger has been successfully migrated. Run the following code to get the functions associated with the `public` schema:

```sql
SELECT
    routine_name
FROM 
    information_schema.routines
WHERE 
    routine_type = 'FUNCTION'
AND
    routine_schema = 'public';
```

This should give you the output:

```
routine_name
timestamp_update_on_edit
```

showing that our new function has been migrated to the database as well! 

We've successfully added a view, a function and a trigger to our database via migrations applied from our SQLAlchemy ORM. 

Browsing through `alembic/versions` should give us an idea of the growth of our data model and how it evolves to match the change in scope of our e-commerce workflow. 

```bash
alembic/versions/
├── 172f9e67a364_created_a_function_to_update_the_last_.py
├── 6e58a26e5c70_created_a_table_for_tags_and_created_a_.py
├── 960969e75871_created_orderproduct_as_an_association_.py
├── a0cf7f5d705d_initial_commit_created_customers_orders_.py
├── e2ba9d70fc4d_created_a_view_called_order_line_items_.py
└── eebe110ef9d2_added_a_column_to_products_to_track_.py
```

# Next Steps

In subsequent blog posts, we will cover: 

- Applying our migrations to a production environment
- Using the ORM as a module when coding a FastAPI server

# References

- The [repo for Alembic Utils.](https://github.com/olirice/alembic_utils)
- The [documentation for Alembic Utils.](https://olirice.github.io/alembic_utils/)