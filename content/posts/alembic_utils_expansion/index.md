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

In this blog post, we will pick up from the progress in the previous entry in this series and continue to develop our data model. We will incorporate PostgreSQL entity types such as functions, views, materialized views, triggers, and policies into our data model, and commit all of these changes to version control.

## What we've covered

In our [previous blog post:](posts/sqlalchemy_data_modeling)

- We used SQLAlchemy to create the first iteration of a rudimentary data model.
- We configured Alembic to connect to a fresh development database and migrated our rudimentary data model. 

## What we'll cover 

Lorem Ipsum

## Brief Concept Recap

I'll briefly cover the concepts we went over in the previous blog post.

### SQLAlchemy

[SQLAlchemy](https://www.sqlalchemy.org) is a Python SQL toolkit and ORM (Object Relational Mapper) that allows Python developers to work with relational databases using the Python language. SQLAlchemy is indispensable if you're a Python developer looking to create a relational data model for your database. The two most significant components of SQLAlchemy are it's **Object Relational Mapper (ORM)** and **SQLAlchemy Core**.

### Alembic

[Alembic](https://alembic.sqlalchemy.org/en/latest/index.html) is a database migration tool written by the author of SQLAlchemy. It helps create a version-controlled system of database migrations that can be used to upgrade a target database to any version of your data model. It uses SQLAlchemy as the underlying engine. By executing migrations in a sequential order, databases can be upgraded and downgraded across versions, even when you are building from scratch.
