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

In this blog post, we will pick up from the progress in the previous entry in this series and continue to develop our data model. We will incorporate PostgreSQL entity types such as functions, views, materialized views, triggers, and policies into our data model, and commit all of these changes to version control using Alembic and Alembic-Utils.

## What we've covered

In our [previous blog post:](posts/sqlalchemy_data_modeling)

- We used SQLAlchemy to create the first iteration of a rudimentary data model, with three tables.
- We configured Alembic to connect to a fresh development database and migrated our rudimentary data model. 

## What we'll cover 

Lorem Ipsum
