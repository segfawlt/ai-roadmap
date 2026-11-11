# Problem

When you separate `alembic` migration source to a separate project, if you don’t want to build a seperate Python package (which can install locally or globally), then you may need to import `relatively`

## Here is solution

``` mermaid
        graph TD
    root["project root<br>(run from terminal here)"]
    root --> dbmig["db-migration/<br>(alembic)"]
    root --> source["source/"]
    dbmig --> envpy["env.py<br>(imports from source)"]
    source --> modelpy["model.py"]
    source --> dbpy["db.py<br>(imports .db from current dir)"]

    envpy -- "from source import model" --> modelpy
    note["Run: PYTHONPATH=.. alembic revision --autogenerate -m 'Add users table'<br>(go up to parent to resolve imports)"]
```

## Problem 2:

## Problem: Import

When you run `main.py` directly (inside `source`), Python sets the working directory to /source, so the root folder (Unit 06-Authentication/) — which contains db_common — is not in the import path.

Source code

Unit 06-Authentication/ │ ├── db_common/ │ ├── **init**.py │ └── db.py │ ├── db-migration/ │ └── source/ ├── auth/ │ ├── **init**.py │ ├── auth_in_fastapi.py │ ├── crud.py │ └── util.py │ ├── main.py ├── models.py ├── schema.py └── docker-compose.yml

Unit 06-Authentication/

Unit06Authentication/ │ ├── app/ │ ├── **init**.py │ ├── main.py │ │ │ ├── auth/ │ │ ├── **init**.py │ │ ├── auth_in_fastapi.py │ │ ├── crud.py │ │ └── util.py │ │ │ ├── core/ │ │ ├── **init**.py │ │ └── db.py │ │ │ ├── models.py │ ├── schema.py │ ├── db_migration/ │ ├── docker-compose.yml └── requirements.txt
