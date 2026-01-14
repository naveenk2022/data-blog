---
title: "Relational Data Model design with SQLALchemy."
date: "2026-01-13"
tags: ['Database','Database Design','PostgreSQL','Data Model','RDBMS','SQLAlchemy']
description: "An introduction to using SQLAlchemy for relational data model design. "
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

## Data Modeling
Data modeling is the process of creating a representation of data that defines the way it is structured and used in complex systems. When it comes to relational databases, designing a normalized data model is essential for efficient database operation, and to make your data consistent and clean.

With a fully normalized data model, data in your relational database becomes:

- Understandable

A good data model can decompose the complexity of real-world systems and the data they generate into a relational model that is easy to read and comprehend, and standardizes the relationship of one data domain with every other data domain.

- Scalable

A well defined and **normalized** data model becomes scalable, and allows for sustainable development of data-driven solutions for users of your database as the scope of your data grows.

- Modular

A database with a well designed and thoroughly documented data model becomes easy to use by downstream teams such as API designers and front-end developers, minimizing the communication needed for these teams to function independently. Good design will naturally flow downstream of a properly normalized database.

Efficient data model design can help your database be a bedrock for your fellow team members to rely on.

## SQLAlchemy

[SQLAlchemy](https://www.sqlalchemy.org) is a Python SQL toolkit and ORM (Object Relational Mapper) that allows Python developers to work with relational databases using the Python language. SQLAlchemy is indispensable if you're a Python developer looking to create a relational data model for your database. The two most significant components of SQLAlchemy are it's **Object Relational Mapper (ORM)** and **SQLAlchemy Core**.

- SQLAlchemy Core

SQLALchemy Core is the foundation of SQLAlchemy, and is independant of the ORM. It provides the connectivity to the database, and by allowing for the construction of SQL expressions that can then be executed against a target database, enabling interaction with a database to in the form of queries.

- SQLAlchemy ORM

The SQLAlchemy ORM (Object Relational Mapper) uses Core as a foundation. The ORM will be the bulk of what we discuss in this blog post. The ORM builds on Core to help create user-defined Python classes that can be mapped into database tables. The ORM is what allows SQLAlchemy to define a data model for a relational database.

{{< figure
    src="https://docs.sqlalchemy.org/en/20/_images/sqla_arch_small.png"
    caption="The major components of SQLAlchemy. "
    attr="Taken from the [SQLAlchemy documentation.](https://docs.sqlalchemy.org/en/20/intro.html)"
    align=center
>}}

## What we'll cover

In this blog post, we will cover how you can begin using SQLAlchemy to start developing the first iteration of your data model. I will go over how SQLAlchemy establishes a connection to your database, and how you can start building a relational data model.

In future posts, we will cover how you can use Alembic to generate database migrations and help grow your database within a version-controlled system, and how this data model can then be plugged into a FastAPI server as a module to help design a modern API service.