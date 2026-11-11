# Alembic Introduction

## What Alembic Actually Does

Alembic is a **lightweight database migration tool for usage with SQLAlchemy**. It was created by the same author as SQLAlchemy and is designed to integrate closely with it.

Its core functions revolve around managing **changes to the database schema**:

1.  **Schema Evolution Management:** Alembic handles the incremental changes needed to update a database structure, known as “migrations”. These changes include adding a new column to a table, modifying the data type of an existing column, creating or dropping indices, and managing other schema objects (like views or stored procedures).

2.  **Creating Migration Scripts:** Alembic manages “revisions,” which are migration files that describe the steps needed to move the database schema from one state to the next. Each script typically contains two main functions:

    - The `upgrade()` function, which applies the necessary changes to advance the schema forward.

    - The `downgrade()` function, which reverts the changes, allowing for rollback to a previous schema version.

3.  **Auto-Generation:** One of Alembic’s key features is the ability to **auto-generate migration scripts** using the `--autogenerate` flag. It achieves this by comparing the **current state of the live database** with the Python **table metadata** defined in the application (e.g., SQLAlchemy models). It’s crucial to note that auto-generation does not detect all database changes, and the candidate migrations it produces always require manual review and correction if needed.

4.  **Applying Migrations:** Developers use the `alembic`` ``upgrade`` ``head` command to apply all pending migration scripts to the database, bringing it up to the latest version.

5.  **Asynchronous Support:** Alembic supports asynchronous environments by being configured to use SQLAlchemy’s Async Engine and async database drivers like `asyncpg`.

Alembic essentially manages **incremental changes to an existing database**, ensuring controlled movement between versions.

## What If We Don’t Use Alembic?

If you choose not to use Alembic, you must manually manage all database schema changes.

1.  **No Automated Schema Management:** **SQLAlchemy (Core and ORM) does not natively handle schema evolution**; it provides tools to create tables initially (`create_all()`) but does not manage incremental changes like `ALTER`` ``TABLE`. Without Alembic, every time you add, remove, or modify a column in your Python model definitions, you would have to manually write the required SQL Data Definition Language (DDL) statements (e.g., `ALTER`` ``TABLE...`` ``ADD`` ``COLUMN...`) and ensure they are run against every database instance (development, staging, production).

2.  **Relying on `create_all()` is Insufficient for Existing Databases:** For initializing a brand new database, you can rely on SQLAlchemy’s `Base.metadata.create_all(engine)` (or `SQLModel.metadata.create_all(engine)`). This command is highly convenient as it emits all necessary `CREATE`` ``TABLE` statements in one shot. However, **`create_all()` cannot update existing tables**; it only creates tables if they don’t already exist. If you modify a model and run `create_all()`, the existing database structure will not be updated, leading to a mismatch between your application’s models and the live database schema.

3.  **Increased Risk of Errors and Downtime:** Manually managing complex DDL operations carries a high risk of errors, particularly when dealing with data migration logic that often accompanies schema changes (e.g., setting default values for a new non-nullable column, or moving data between columns). Alembic forces a structured `upgrade`/`downgrade` flow, providing a safety mechanism for version control and rollback capability that would otherwise need to be custom-built.

4.  **Incoherent Environments:** In the long run, managing schema updates manually often leads to different environments (like development vs. production) running on inconsistent database versions, making debugging and deployment significantly harder. Alembic solves this by ensuring all environments can be driven programmatically to the same target revision.

To conceptualize the difference:

**SQLAlchemy’s `create_all()`** is like building a house from blueprints—it works perfectly if the land is empty, providing the foundation for a new structure.

**Alembic** is like managing historical renovation permits—it tracks every modification, addition, and structural change over time, ensuring every incremental change to the existing building is documented, reversible, and correctly applied across all copies of the building.

------------------------------------------------------------------------

## How to do ?

### Problem:

In `models.User` we want to add new field `address` in the existing table (`User`). Then how can we alter this table?

``` python
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50))
    email = Column(String(120), unique=True, index=True)

    address = Column(String(250)) # We want to add new columns
```

### `alembic` to rescue

Install `alembic`

``` sh
pip install alembic
```

### Create `alembic` migration project

``` sh
alembic init alembic
```

### `alembic` support 2 modes: Offline/Online migrations

- **Offline Migrations**

  - Alembic does not connect to the database.

  - Instead, it generates SQL statements as text output.

  - Useful for environments where you can’t or don’t want to connect to a DB.

- **Online Migrations**

  - Alembic establishes a live database connection (engine/connection object).

  - The migration scripts run SQL directly on the database.

  - Typical for normal usage (e.g., alembic upgrade head).

| Feature | Online Mode | Offline Mode |
|----|----|----|
| Database connection | ✅ Yes | ❌ No |
| Executes SQL directly | ✅ Yes | ❌ No (outputs SQL text) |
| Can autogenerate migrations | ✅ Yes | ❌ No |
| Typical use case | Apply migrations automatically | Generate SQL for manual review/deployment |
| Command example | `alembic`` ``upgrade`` ``head` | `alembic`` ``upgrade`` ``head`` ``--sql`` ``>`` ``script.sql` |

### `alembic` command lines

- Generate a revision migration file (\*.py)

``` sh
alembic revision --autogenerate -m "Initial schema"

PYTHONPATH=.. alembic revision --autogenerate -m "add new column Address"
```

- Then run from terminal

``` sh
PYTHONPATH=.. alembic upgrade head
```
