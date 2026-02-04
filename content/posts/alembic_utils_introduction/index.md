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