#  pgvector base

A clean template for me to develope app based on `pgvector` and using alembic to control table migration. 




# Structure

The directories structure is as below: 

```
$ tree
.
├── README.md
├── alembic.ini                 # alembic setting file generated by alembic
├── poetry.lock                 # poetry related files
├── pyproject.toml
├── docker-compose.yaml         # container for postgresSQL
├── .gitignore
├── migrations                  # migration related scripts generated by alembic
│   ├── README
│   ├── env.py
│   ├── script.py.mako
│   └── versions
└── src    
    ├── models                  # modules for placing the models
    └── db_models.py            # table orm model

```


# Installation

## Prerequisites

- Python 3.11

## Template setup

1. **Clone the repository**: Fist of all, clone this repository using as follows: 

    ```sh 
    git clone https://github.com/Kuroflectr/pgvetor_base.git
    ```

2. **Install dependencies**: under the root directory, use `poetry` to install all the packages that will be used: 

    ```sh
    poetry install
    ```



# Usage

1. **Create/ Edit `db_model.py`**: The following code defines the table we want to migrate. 

    ```python
    from pgvector.sqlalchemy import Vector
    from sqlalchemy import Column, Integer, String
    from sqlalchemy.orm import declarative_base

    Base = declarative_base()


    # Create a table Model which inherit from Base
    class AnkiNoteModel(Base):
        """Create a table based on the specified model as follows:"""

        __tablename__ = "vector_table_1"

        product_id = Column(Integer, primary_key=True)
        product_name = Column(String, nullable=False)
        description = Column(String, nullable=False)
        vector = Column(Vector(dim=1536))

    ```

    In this repository I have already created this file under `src/models`, but you can still make a new one of your own (or modify the columns to suit your needs).  



2. **Edit `alembic.ini` (optional)**:  Edit `sqlalchemy.url` in `alembic.ini` like: 
    ```sh
    sqlalchemy.url = postgresql+psycopg2://postgres:postgres@localhost/pgvector_db
    ```

    Only modify this when you changed `docker-compose.yaml` (see Explaination )

3. **Edit `env.py` (optional)**: Import the defined `Base` into the env.py

    ```python
    from models.db_model import Base
    ```

    Also, edit the following part: 
    ```python
    target_metadata = Base.metadata
    ```
    Only modify this when you created new table ORM model scripts or changed the path of `db_model.py` 

4. **Migration preparation**: After `env.py`, `db_model.py`, and `alembic.ini` are done, create a migration script using: 

    ```sh 
    alembic revision --autogenerate -m "Create initial tables"
    ```
    This will generate a python script (e.g., `{unique_id}_create_initial_tables.py`) under `migrations/versions/` to execute migration.

5. **Modify the migration script**: By default alembic doesn't deal with the pgvector packages so we have to add it into `{unique_id}_create_initial_tables.py` manually: 

    ```python 
    import pgvector
    ```

6. **Apply migration**: Use the following command to apply migration. 
    
    ```sh
    alembic upgrade head
    ```


# Explaination

## Modification of `env.py`

There are several scripts which are automatically generated by 


`env.py` serves as a template to produce the code for migration. However, when migrating the model defined in sqlalchemy to PostgreSQL DB, it lacks the definition of vector column which causes errors during the migration. As a result, we need to add extra functions to fix it. 


Add the functions under it: 
```python
def create_vector_extension(connection) -> None:
    try:
        with Session(connection) as session:  # type: ignore[arg-type]
            # The advisor lock fixes issue arising from concurrent
            # creation of the vector extension.
            # https://github.com/langchain-ai/langchain/issues/12933
            # For more information see:
            # https://www.postgresql.org/docs/16/explicit-locking.html#ADVISORY-LOCKS
            statement = sqlalchemy.text(
                "BEGIN;" "CREATE EXTENSION IF NOT EXISTS vector;" "COMMIT;"
            )
            session.execute(statement)
            session.commit()
    except Exception as e:
        raise Exception(f"Failed to create vector extension: {e}") from e


def do_run_migrations(connection) -> None:
    # Need to hack the "vector" type into postgres dialect schema types.
    # Otherwise, `alembic check` does not recognize the type

    # This line of code registers the custom vector type with SQLAlchemy so that SQLAlchemy knows how to
    # handle this custom type when interacting with the database schema.
    # It tells SQLAlchemy that the vector type in PostgreSQL should be mapped
    # to the pgvector.sqlalchemy.Vector type in Python.
    # ref: https://github.com/sqlalchemy/alembic/discussions/1324
    connection.dialect.ischema_names["vector"] = pgvector.sqlalchemy.Vector

    context.configure(
        connection=connection,
        target_metadata=target_metadata,
    )

    with context.begin_transaction():
        context.run_migrations()
```


## Container for postgres

I configured the postgresSQL using a container which is defined in `docker-compose.yaml`: 

```
version: '3.8'
services:
db:
    image: ankane/pgvector
    environment:
    POSTGRES_USER: postgres 
    POSTGRES_PASSWORD: postgres
    POSTGRES_DB: pgvector_db
    ports:
    - "5432:5432"
    volumes:
    - pgdata:/var/lib/postgresql/data
    deploy:
    resources:
        limits:
        cpus: '0.50'
        memory: '512M'
        reservations:
        cpus: '0.25'
        memory: '256M'
volumes:
    pgdata:
```

The following arguments 
```
POSTGRES_USER: postgres 
POSTGRES_PASSWORD: postgres
POSTGRES_DB: pgvector_db
```
decides the url that should be put into `alembic.ini`. 