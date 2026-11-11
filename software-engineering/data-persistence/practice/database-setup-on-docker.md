# \~~Install Docker CE (On Mac)\~~

- Need to install a lightweight Linux environment such as Colima \>\> Not OK

- Change to install `Docker`` ``CE` to Docker Desktop for **Personal Usage**

## Install Docker Desktop on MacOS

- <https://www.docker.com/products/docker-desktop/>

## Install Docker CE on WSL (Window): Recommended

Refer to company guide

## Setup Docker on WSL Windows Machine

- **Step 1**: Pull `postgres` image from DockerHub

``` sh
docker pull postgres
```

- **Step 2**: Create `docker-compose.yml`

  - Then using docker compose-up or VS Code Extension Here is sample

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
  adminer:
    image: dockette/adminer
    ports:
      - 8080:80
  pgadmin:
    image: dpage/pgadmin4
    ports:
      - 5050:80
    environment:
      PGADMIN_DEFAULT_EMAIL: "[email protected]"
      PGADMIN_DEFAULT_PASSWORD: "abc@123"
```

Change `POSTGRES_PASSWORD`, `POSTGRES_USER`, `POSTGRES_USER`… for something if you want
