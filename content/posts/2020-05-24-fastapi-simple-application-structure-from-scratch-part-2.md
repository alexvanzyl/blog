---
title: "FastAPI: Simple application structure from scratch - Part 2"
description: In part 2 of this series we add migrations to our project.
date: 2020-05-24
author: Alex van Zyl
tags:
  - fastapi
  - python
images:
  - /uploads/fastapi-simple-application-structure-from-scratch-part-2.jpg
featuredImage: /uploads/fastapi-simple-application-structure-from-scratch-part-2.jpg
draft: false
---

Continuing where we left off in part one of this series, we will add migrations to our project using Alembic. 
  
## Series Content :book:  

- [Part 1]({{< ref "2020-05-19-fastapi-simple-application-structure-from-scratch.md" >}}): Laying the foundation  
- Part 2: Migrations (this post)
- [Part 3]({{< ref "2020-05-30-fastapi-simple-application-structure-from-scratch-part-3.md" >}}): Dockerize

{{< admonition type=warning title="Before getting started..." open=true >}}  
To avoid any issues, make sure to drop the posts table from your database and run `poetry install` again.
{{< /admonition >}}


## What we will cover in this post? :memo:  
  
- What is Alembic?  
- Install Alembic 
- Restructure project to support auto migrations
- Create a `migrations` directory  
- Configure Alembic
- Generate migration file  
  

## What is Alembic? :thinking:
From Alembic's GitHub [repository](https://github.com/sqlalchemy/alembic).  
> Alembic is a database migrations tool written by the author of [SQLAlchemy](http://www.sqlalchemy.org).  
  
## Install Alembic 

Since we will be running many commands in our virtual environment, let's switch that shell now.  
 
```shell  
poetry shell  
```  

The first thing we need to do is add Alembic as a dependency.  
  
```shell  
poetry add alembic  
```  
  
## Create a migrations directory :file_folder:
Now that we have Alembic installed let's go ahead and generate the `migrations` directory.  
  
```shell  
alembic init migrations  
```

After executing the above command a `migrations` directory and `alembic.ini` file will be generated. Our new project structure will look like this now.

```
.
├── alembic.ini
├── app
├── migrations
├── mypy.ini
├── poetry.lock
├── pyproject.toml
├── README.md
└── tests
```
The `alembic.ini`  file holds the configurations parameter for Alembic, such as the path to our migration scripts. Let's also go over the content of the `migrations` directroy.

```
migrations
├── env.py
├── README
├── script.py.mako
└── versions
```
The `env.py` file reads the configurations from the `alembic.ini` and handles running the migration scripts. We can also edit this file and tweak to our needs, something we will do later.

The `README` file can be used to add informative details about our migrations. We will leave this file as-is is for now.

The `script.py.mako` file is a  [Mako](https://www.makotemplates.org/)  template file. Alembic uses this template when generating revision files  (migration scripts).

Finally, we have the `versions` directory this is where all our revision files will live.

## Restructure project to support auto migrations

In order for us to take advantage of Alembics [auto generated migrations](https://alembic.sqlalchemy.org/en/latest/autogenerate.html) we need to restructure our project and make some minor changes to existing files.   
  
Let's create a new `db` directory now.

```shell
mkdir app/db
```

We going to add three new files to this directroy, `__init__.py`, `base.py` and `session.py`

```shell
touch app/db/__init__.py app/db/base.py app/db/session.py
```

What we now have is an empty `db` package. This will replace the `db.py` module and we will delete this file later.

### Edit `__init__.py` file.

`app/db/__init__.py`

```python
from .base import Base  # noqa

# Import all the models, so that the Base class 
# has them before being imported by Alembic.
from .. import models # noqa
```

The schema for each table is derived from the attributes specified on each model. We import all our models afterwards to make sure the `Base.metadata` attribute gets populates with all the table schemas. Alembic makes use of this when using `autogenerate`.
  
{{< admonition type=info open=true >}}
`Base.metadata.tables` contains a collection of SQLAlchemy [`Table`](https://docs.sqlalchemy.org/en/13/core/metadata.html#sqlalchemy.schema.Table) objects. 
{{< /admonition >}}

### Edit `base.py` file.

We going to move the original `Base` class from the `db.py` module into this file, but we will make a few changes to it. 

First, we going to update our `Base` class to automatically generate the `__tablename__` attribute for or models. The name will be derived from the model class name. For example, in our case, the `Post` class would generate a `post` table. 

I prefer my table names to be in plural form, so instead of `post` it would `posts`. For this, we can make use of the [inflect.py](https://github.com/jazzband/inflect) module. If you share the same preference, then let's install that now quick.

```shell
poetry add inflect
```

`app/db/base.py`

```python
from typing import Any    
    
import inflect     
from sqlalchemy.ext.declarative import as_declarative, declared_attr    
   
p = inflect.engine()    
    
    
@as_declarative()    
class Base:    
    id: Any    
    __name__: str    
    
    # Generate __tablename__ automatically in plural form.   
    # i.e 'Post' model will generate table name 'posts'   
    @declared_attr    
    def __tablename__(cls) -> str:    
        return p.plural(cls.__name__.lower())

```

The main changes to the `Base` class are we added a new `__name__` and declarative `__tablename__` attribute.

This means we no longer need to declare the `__tablename__` attribute on our models. Let’s edit the Post model in our models.py file to reflect this change.

### Update `models.py` file

`app/models.py`
```python
from uuid import uuid4

from sqlalchemy import Column, String, Text
from sqlalchemy.dialects.postgresql import UUID

from .db.base import Base


class Post(Base):
    id = Column(UUID(as_uuid=True), primary_key=True, index=True, default=uuid4)
    title = Column(String)
    body = Column(Text)

```

We updated our import statement for the `Base` class and dropped the `__tablename__` attribute on the `Post` model.

### Edit `session.py` file

Again we can copy the session details from `db.py` file into this one.

`app/db/session.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from ..config import settings

engine =  create_engine(settings.SQLALCHEMY_DATABASE_URI, pool_pre_ping=True)
SessionLocal =  sessionmaker(autocommit=False, autoflush=False, bind=engine)

```

### Update `main.py` file

We need to update our import statements on line 8 and 9. Also, let's comment out line 13 since we will be using migrations.

`app/main.py`

```python
...
from . import actions, schemas
from .db.session import SessionLocal
...

# Create all tables in database.
# Comment this out if you using migrations.
# models.Base.metadata.create_all(bind=engine)
...
```

We can now delete the `db.py` file.

```shell
rm app/db.py
```

## Configure Alembic :gear:

### Update `alembic.ini` file

We going to configure our connection details in the `env.py` file. So let's open up the `alembic.ini` and comment out the line `sqlalchemy.url = driver://user:pass@localhost/dbname`, like below. 

`alembic.ini`
```
...
# the output encoding used when revision files
# are written from script.py.mako
# output_encoding = utf-8

# sqlalchemy.url  = driver://user:pass@localhost/dbname
...
```

### Update `env.py` file

Let's configure the `env.py` file now. 

`migrations/env.py` line 21:22

```python
...
# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata

from app.db import Base # noqa
target_metadata = Base.metadata
...
```

{{< admonition type=note open=true >}}   
We import the `Base` class from `app.db` and not `app.db.base` for reason explained before. 
{{< /admonition >}}

Next we will create our own function the will get the URL to our database connection.

`migrations/env.py` line 31:38

```python
...
def get_url():
    from app.config import settings

    user = settings.POSTGRES_USER
    password = settings.POSTGRES_PASSWORD
    server = settings.POSTGRES_SERVER
    db = settings.POSTGRES_DB
    return f"postgresql://{user}:{password}@{server}/{db}"
...
```

All we are doing here is importing and using our database configuration settings to generate the connection URL.

To make use of this new function we need to update the `run_migrations_offline` and `run_migrations_online` functions. Let's do that now.

`migrations/env.py`  line 53 and 72:78

```python
...
def run_migrations_offline():
    ...
    url = get_url()
    ...

def run_migrations_online():
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()
    connectable = engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    ...
...
```

The final `env.py` file should like this now.

`migrations/env.py`
```python
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata

from app.db import Base # noqa
from app import models # noqa
target_metadata = Base.metadata

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.


def get_url():
    from app.config import settings

    user = settings.POSTGRES_USER
    password = settings.POSTGRES_PASSWORD
    server = settings.POSTGRES_SERVER
    db = settings.POSTGRES_DB
    return f"postgresql://{user}:{password}@{server}/{db}"


def run_migrations_offline():
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = get_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online():
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()
    connectable = engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()

```

##  Generate migration file :sparkles:
Finally, at this point we can now generate our first migraion script.

```shell
alembic revision --autogenerate -m "Create posts table"
```

This will generate a new migration file in the `migrations/versions/` directory. In my case it created a file named `ee48ba03fe9f_create_posts_table.py` with the below content.

`migrations/versions/ee48ba03fe9f_create_posts_table.py`
```python
"""Create posts table

Revision ID: ee48ba03fe9f
Revises: 
Create Date: 2020-05-24 15:39:50.164129

"""
import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

# revision identifiers, used by Alembic.
revision = "ee48ba03fe9f"
down_revision = None
branch_labels = None
depends_on = None


def upgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table(
        "posts",
        sa.Column("id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("title", sa.String(), nullable=True),
        sa.Column("body", sa.Text(), nullable=True),
        sa.PrimaryKeyConstraint("id"),
    )
    op.create_index(op.f("ix_posts_id"), "posts", ["id"], unique=False)
    # ### end Alembic commands ###


def downgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_index(op.f("ix_posts_id"), table_name="posts")
    op.drop_table("posts")
    # ### end Alembic commands ###
```

To run this migration now all we need to do is run the following command.

```shell
alembic upgrade head
```

We can also roll back changes by running the below.

```shell
alembic downgrade head
```

## Conclusion :bulb:
Alembic is now in place to manage all our migration scripts. We had to make some minor changes to achieve this and refactored `db.py` file into multiple files that make up a package.

The final code for this post can be found on [GitHub](https://github.com/alexvanzyl/fastapi-simple-app-example/tree/part-2).  
  
If you enjoyed reading this article and would like to stay tuned for more, or just want to connect, follow me on twitter [@alexvanzyl](https://twitter.com/alexvanzyl).