# Exercise: Setup PostgreSQL on Docker **CE** (not Desktop)

## Goal

Run a Postgres instance in **Docker Engine - Community** and verify connectivity.

## Tasks

1.  **Install Docker CE (Engine) on Linux** (Ubuntu/Debian recommended).

    - In the case you use Windows, set up WSL

    - Verify with:

      ``` bash
      docker --version
      docker info | grep -E "Server|Engine|Community"
      docker compose version   # plugin v2 preferred
      ```

    - **Acceptance proof:** `docker`` ``info` shows `Server:`` ``Docker`` ``Engine`` ``-`` ``Community`. **Any evidence of “Docker Desktop” = fail.**

2.  **Start PostgreSQL via Docker Compose (or docker run)**

    - Create `docker-compose` file with some basic information:

    ``` yaml
    image: postgres:16-alpine
    container_name: todo_db
    Some environment configs (user/pass)
    ```

3.  **Verify database connectivity**

    - From host or a psql client container:

      ``` bash
      docker exec -it todo_db psql -U todo_user -d todo_db -c "\dt"
      ```

## Acceptance Criteria

- Evidence `docker`` ``info` shows **Community** (CE).

- Postgres container healthy (`docker`` ``ps`, healthcheck passing).

- Can run `SELECT`` ``1` against the DB.

## Deliverables

- Screenshot/log of `docker`` ``info` indicating CE, container status, and a successful query.

## Rubric (30 pts)

- Docker CE installed & verified

- Postgres container running & healthy

- Connectivity verified
