# Step-by-step: Add Connection in pgAdmin

1.  **Open pgAdmin** You already have it open at `http://localhost:5050` (based on your Docker setup).

2.  **Click “Add New Server”** You’ll find it in the “Quick Links” section or by right-clicking on **Servers → Register → Server**.

3.  **In the dialog:**

    **General Tab**

    - **Name:** `Local`` ``Docker`` ``Postgres` (or any name you like)

    **Connection Tab**

    - **Host name / address:** If pgAdmin and Postgres are in the **same Docker Compose network** (as in your `docker-compose.yml`), use the service name:

      ```
      postgres
      ```

      > Don’t use `localhost` — pgAdmin inside Docker can’t see your host machine’s localhost; it communicates through the Docker network. You can use **service_name** (in docker), for example `postgres`

    - **Port:** `5432`

    - **Maintenance database:** `example_pgdb`

    - **Username:** `root`

    - **Password:** `postgres`

    (These match your `docker-compose` environment variables.)

    Here is sample docker-compose.yml

    ``` yaml
    version: "3.9"
     services:
     postgres:
         image: "postgres"
         ports:
         - "5432:5432"
         environment:
         POSTGRES_PASSWORD: postgres
         POSTGRES_USER: root
         POSTGRES_DB: example_pgdb
    ```

4.  **Save** Click **Save** — pgAdmin should now connect to your Postgres container.
