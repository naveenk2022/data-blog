---
title: "Relational Data Model design with SQLALchemy."
date: "2026-01-13"
tags: ['Database','Database Design','PostgreSQL','Data Model','RDBMS','SQLAlchemy','Alembic']
description: "An introduction to using SQLAlchemy and Alembic for relational data model design. "
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

### Alembic

[Alembic](https://alembic.sqlalchemy.org/en/latest/index.html) is a database migration tool written by the author of SQLAlchemy. It helps create a version-controlled system of database migrations that can be used to upgrade a target database to any version of your data model. It uses SQLAlchemy as the underlying engine. By executing migrations in a sequential order, databases can be upgraded and downgraded across versions, even when you are building from scratch. 

Alembic is a great way to test your migrations and schema design in a development environment before transferring these migrations to a production environment. An empty database environment can be converted into a database with a production ready schema with just a single command using Alembic.

## What we'll cover

In this blog post, we will cover how you can begin using SQLAlchemy and Alembic to start developing the first iteration of your data model. 

In future posts, we will cover how you can use Alembic to generate database migrations and help grow your database within a version-controlled system, and how this data model can then be plugged into a FastAPI server as a module to help design a modern API service.

# Prerequisites

The code in this blog post was written with Python 3.13.1, Alembic 1.14.0 and SQLAlchemy 2.0.36.

You will need the following prerequisites: 

- A PostgreSQL server initialized and running, with a database that has been initialized but is otherwise empty. We will refer to this server as `development` from here.

- An installation of `mamba`,`micromamba` or `conda` as a package manager to create a new environment. I like to use `micromamba`. [You can find instructions on how to install `micromamba` here.](https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html) 

We will create a new virtual environment using Python 3.13.1 as the pinned version. Create an environment named `data_model` as follows:

```python
micromamba create -n data_model python=3.13.1 sqlalchemy=2.0.36 alembic=1.14.0 -y
```

This will create a micromamba environment with the pinned versions of Python and SQLAlchemy that can then be used to follow this tutorial. This environment can be activated with the command `micromamba activate data_model`. 

> [!NOTE]
> Using a virtual environment is a good way to keep your base environment uncluttered and free of broken dependencies. 
> It is good practice to begin any Python project with a virtual environment, even for development workflows. 
> Other options for virtual environment creation include `uv` and `Poetry`.

Lets now get into actually creating some data models! 

# Creating our first data model

## Setting up the Alembic environment

Before we can start working with our PostgreSQL database, we will first set up the migration environment using Alembic. The reason we set this up first is because we will use Almbic's config files to connect to our database first before we can begin creating our data model. Alembic uses SQLAlchemy core's `Engine` object as a dependency when connecting to a database.

In the root of your project directory, run:

```python
alembic init alembic
```
This will generate a directory of scripts. The root of your project directory will look like this:

```bash
.
├── alembic
│   ├── README
│   ├── env.py
│   ├── script.py.mako
│   └── versions
└── alembic.ini
```

The directory now inclues these files of importance:

### `alembic.ini`
This is Alembic's main configuration file. When Alembic is run, it looks in the directory for this file. This file is where you will add the SQLAlchemy URL to connect to your database. You can also define **multiple** environments to run migrations in.

For now we will go with the boilerplate file generated by the alembic init file. Here is what the default file should look like:

```ini 
# A generic, single database configuration.

[alembic]
# path to migration scripts
# Use forward slashes (/) also on windows to provide an os agnostic path
script_location = alembic

# template used to generate migration file names; The default value is %%(rev)s_%%(slug)s
# Uncomment the line below if you want the files to be prepended with date and time
# see https://alembic.sqlalchemy.org/en/latest/tutorial.html#editing-the-ini-file
# for all available tokens
# file_template = %%(year)d_%%(month).2d_%%(day).2d_%%(hour).2d%%(minute).2d-%%(rev)s_%%(slug)s

# sys.path path, will be prepended to sys.path if present.
# defaults to the current working directory.
prepend_sys_path = .

# timezone to use when rendering the date within the migration file
# as well as the filename.
# If specified, requires the python>=3.9 or backports.zoneinfo library.
# Any required deps can installed by adding `alembic[tz]` to the pip requirements
# string value is passed to ZoneInfo()
# leave blank for localtime
# timezone =

# max length of characters to apply to the "slug" field
# truncate_slug_length = 40

# set to 'true' to run the environment during
# the 'revision' command, regardless of autogenerate
# revision_environment = false

# set to 'true' to allow .pyc and .pyo files without
# a source .py file to be detected as revisions in the
# versions/ directory
# sourceless = false

# version location specification; This defaults
# to alembic/versions.  When using multiple version
# directories, initial revisions must be specified with --version-path.
# The path separator used here should be the separator specified by "version_path_separator" below.
# version_locations = %(here)s/bar:%(here)s/bat:alembic/versions

# version path separator; As mentioned above, this is the character used to split
# version_locations. The default within new alembic.ini files is "os", which uses os.pathsep.
# If this key is omitted entirely, it falls back to the legacy behavior of splitting on spaces and/or commas.
# Valid values for version_path_separator are:
#
# version_path_separator = :
# version_path_separator = ;
# version_path_separator = space
# version_path_separator = newline
version_path_separator = os  # Use os.pathsep. Default configuration used for new projects.

# set to 'true' to search source files recursively
# in each "version_locations" directory
# new in Alembic version 1.10
# recursive_version_locations = false

# the output encoding used when revision files
# are written from script.py.mako
# output_encoding = utf-8

sqlalchemy.url = driver://user:pass@localhost/dbname


[post_write_hooks]
# post_write_hooks defines scripts or Python functions that are run
# on newly generated revision scripts.  See the documentation for further
# detail and examples

# format using "black" - use the console_scripts runner, against the "black" entrypoint
# hooks = black
# black.type = console_scripts
# black.entrypoint = black
# black.options = -l 79 REVISION_SCRIPT_FILENAME

# lint with attempts to fix using "ruff" - use the exec runner, execute a binary
# hooks = ruff
# ruff.type = exec
# ruff.executable = %(here)s/.venv/bin/ruff
# ruff.options = --fix REVISION_SCRIPT_FILENAME

# Logging configuration
[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARNING
handlers = console
qualname =

[logger_sqlalchemy]
level = WARNING
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```

To begin with, we will substitute the `sqlalchemy.url` parameter for the `alembic` section with the connection string used for our database. As an example, here's a dummy connection string.

```ini
sqlalchemy.url = postgresql://nk_dev:placeholder_password@localhost/development
```

In the above code snippet:

- The database driver is `postgresql`
- The database username is `nk_dev`
- The password of the user is `placeholder_password`
- The database host is `localhost`
- The name of the database is `development`


> [!Warning]
> The above is an example to begin testing with. 
> This isn't the ideal way to store your PostgreSQL server connection details. 
> Ideally you manage them with a key vault or via injection of environment variables at the time of virtual environment activation.
> Remember to **never** include the alembic.ini file within version control. Always have your config file (`alembic.ini`) added to `.gitignore`.
>
> Here are the contents of the `.gitignore:` file:
>```
>alembic.ini
>```

A more [in-depth explanation of the `alembic.ini` file can be found in the official documentation here.](https://alembic.sqlalchemy.org/en/latest/tutorial.html#editing-the-ini-file)

### The `alembic` directory

This directory is the home of the migration environment. The relevant contents of this directory include:

- **`env.py`**

This is a Python script that can be run when the migration tool is invoked. You can do some cool stuff with this script, like registering entities like views, functions and triggers, and also generating filters to ignore tables that you don't want to migrate with Alembic. 

- **`versions/`**

This directory holds the individual version scripts. This directory will have the scripts in it run sequentially when upgrading to or downgrading to a particular version of your database. As you begin generating migrations and applying them, this folder will grow. Each migration will become it's own script, containing both an `upgrade` and a `downgrade` parameter for that particular version. 

Now that all of this is finally sorted, we can get to creating basic data models! 

## Creating a file for our data models

Create a file in the project directory called `models.py`. This file will be where we store our user-defined Python classes that will then be mapped into database tables in the database using Alembic to run the actual migrations.