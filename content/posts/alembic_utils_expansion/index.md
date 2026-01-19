---
title: "Expanding Data Models with Alembic."
date: "2026-01-13"
tags: ['Python', 'PostgreSQL', 'SQLAlchemy', 'Alembic', 'ORM', 'Database Design', 'Database Migrations', 'Tutorial']
description: "Expanding relational data models with Entity types using Alembic and Alembic-Utils."
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

As the scope of a data model and it's downstream APIs and front-end tools changes, it will need to be updated and expanded to account for this new scope.

When changes are made to a data model, especially to a model that is split across development and production environments, schema drift becomes a constant problem that looms in the background.

While typical software development is subsumed to a version controlled system as a standard, data model development is typically not submitted to a robust version control system. As different team members collaborate on a data model split across different environments,  the possibility of schema drift gradually grows when changes to the data model are not tracked in a central repository.

In this blog post, we will pick up from the progress in the previous entry in this series and continue to develop our data model. We will incorporate PostgreSQL entity types such as functions, views, materialized views, triggers, and policies into our data model, and commit all of them to version control.

## What we've covered

In our previous blog post:

- We used SQLAlchemy to create the first iteration of a rudimentary e-commerce data model.
- We used Alembic to migrate this rudimentary data model to our fresh development database.

## Brief Recap

### SQLAlchemy

[SQLAlchemy](https://www.sqlalchemy.org) is a Python SQL toolkit and ORM (Object Relational Mapper) that allows Python developers to work with relational databases using the Python language. SQLAlchemy is indispensable if you're a Python developer looking to create a relational data model for your database. The two most significant components of SQLAlchemy are it's **Object Relational Mapper (ORM)** and **SQLAlchemy Core**.

### Alembic

[Alembic](https://alembic.sqlalchemy.org/en/latest/index.html) is a database migration tool written by the author of SQLAlchemy. It helps create a version-controlled system of database migrations that can be used to upgrade a target database to any version of your data model. It uses SQLAlchemy as the underlying engine. By executing migrations in a sequential order, databases can be upgraded and downgraded across versions, even when you are building from scratch.
