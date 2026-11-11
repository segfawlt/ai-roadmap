# Docker Fundamentals

## Introduction

Docker packages applications with all their dependencies into **containers** — lightweight, portable units that run consistently across any environment. Whether you’re developing on macOS, testing in CI, or deploying to production on Linux, the container behaves identically.

**Why Docker Matters:**

- **Consistency**: “Works on my machine” becomes “works everywhere”

- **Isolation**: Each container has its own filesystem, network, and process space

- **Reproducibility**: Pin exact versions of OS, language runtime, and dependencies

- **Fast onboarding**: New developers run `docker`` ``compose`` ``up` instead of installing dozens of tools

------------------------------------------------------------------------

## Core Concepts

``` mermaid
        graph TD
    A[Dockerfile] -->|docker build| B[Image]
    B -->|docker run| C[Container]
    B -->|docker push| D[Registry<br>Docker Hub / GitLab Registry]
    D -->|docker pull| B
```

| Concept | Description | Analogy |
|----|----|----|
| **Dockerfile** | Recipe for building an image | A cooking recipe |
| **Image** | Read-only template with app + dependencies | A frozen meal |
| **Container** | Running instance of an image | The meal, heated and served |
| **Registry** | Storage for images | A grocery store |
| **Volume** | Persistent storage outside the container | A USB drive |
| **Network** | Communication between containers | A LAN cable |

------------------------------------------------------------------------

## Dockerfile Best Practices

### Basic Dockerfile (Python/FastAPI)

``` dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies first (leverages Docker layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/

# Run as non-root user (security best practice)
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Multi-Stage Build (Smaller Images)

Multi-stage builds produce smaller production images by separating the build environment from the runtime:

``` dockerfile
# Stage 1: Build
FROM python:3.12 AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt

# Stage 2: Runtime (smaller base image)
FROM python:3.12-slim

WORKDIR /app

# Copy only the installed packages from the builder stage
COPY --from=builder /app/deps /usr/local/lib/python3.12/site-packages/
COPY src/ ./src/

RUN useradd --create-home appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Layer Caching Rules

Docker caches each layer. If a layer changes, all subsequent layers are rebuilt. Order your instructions from **least frequently changed** to **most frequently changed**:

``` dockerfile
# GOOD order (cache-friendly):
FROM python:3.12-slim          # Rarely changes
WORKDIR /app                   # Never changes
COPY requirements.txt .        # Changes when deps change
RUN pip install -r req...      # Cached unless requirements.txt changed
COPY src/ ./src/               # Changes on every code change (last!)

# BAD order (breaks cache on every change):
FROM python:3.12-slim
COPY . .                       # Everything changes → pip install runs every time
RUN pip install -r requirements.txt
```

### Dockerfile Checklist

Use specific image tags (`python:3.12-slim`), never `latest`

Copy dependency files before source code (layer caching)

Use `--no-cache-dir` with pip (smaller image)

Run as non-root user (`USER`` ``appuser`)

Use `.dockerignore` to exclude unnecessary files

Use multi-stage builds for production images

Set `EXPOSE` for documentation (does not publish ports)

### .dockerignore

```
.git
.venv
__pycache__
*.pyc
.env
.env.*
node_modules
.pytest_cache
.coverage
*.egg-info
dist/
build/
```

------------------------------------------------------------------------

## Docker Compose for Development

Docker Compose defines multi-container applications in a single `docker-compose.yml` file.

### FastAPI + PostgreSQL + Redis

``` yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src # Hot reload: code changes reflect immediately
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata: # Named volume persists data across container restarts
```

### Essential Commands

``` bash
# Start all services (foreground)
docker compose up

# Start in background
docker compose up -d

# Rebuild images after Dockerfile changes
docker compose up --build

# Stop all services
docker compose down

# Stop and remove volumes (fresh database)
docker compose down -v

# View logs
docker compose logs -f app

# Execute command in running container
docker compose exec app bash

# Run a one-off command
docker compose run --rm app pytest tests/
```

### Volume Types

| Type | Use Case | Example |
|----|----|----|
| **Named volume** | Persistent data (databases) | `pgdata:/var/lib/postgresql/data` |
| **Bind mount** | Development hot-reload | `./src:/app/src` |
| **tmpfs** | Temporary data (secrets, caches) | `tmpfs:`` ``/tmp` |

**Named volumes** persist across container restarts and are managed by Docker. **Bind mounts** map host directories into the container and are essential for development workflows where you want live code reloading.

------------------------------------------------------------------------

## Networking

Containers in the same `docker-compose.yml` can communicate using service names as hostnames:

``` python
# Inside the 'app' container, connect to PostgreSQL:
DATABASE_URL = "postgresql://postgres:postgres@db:5432/myapp"
#                                                 ^^
#                                  service name = hostname
```

### Common Networking Patterns

| Pattern | When to Use |
|----|----|
| `ports:`` ``"8000:8000"` | Expose to host machine (development, external access) |
| No `ports` | Service only accessible to other containers (internal database) |
| Custom network | Isolate groups of services from each other |

------------------------------------------------------------------------

## Security

### Image Security

``` bash
# Scan image for vulnerabilities
docker scout cves myapp:latest

# Alternative: Trivy (open source)
trivy image myapp:latest
```

### Runtime Security Checklist

Run as non-root user (`USER`` ``appuser` in Dockerfile)

Use read-only filesystem where possible (`read_only:`` ``true` in compose)

Never store secrets in images (use environment variables or Docker secrets)

Pin image versions (`postgres:16-alpine`, not `postgres:latest`)

Scan images in CI before deployment

Use minimal base images (`-slim`, `-alpine`)

Drop unnecessary Linux capabilities

------------------------------------------------------------------------

## Debugging Containers

``` bash
# Check running containers
docker ps

# View container logs
docker logs <container_id> --tail 50

# Shell into a running container
docker exec -it <container_id> bash

# Inspect container configuration
docker inspect <container_id>

# Check resource usage
docker stats

# View container filesystem changes
docker diff <container_id>
```

------------------------------------------------------------------------

## Summary

| Topic | Key Takeaway |
|----|----|
| **Images** | Use specific tags, multi-stage builds, non-root user |
| **Compose** | Define multi-container dev environment in one file |
| **Volumes** | Named volumes for data persistence, bind mounts for development |
| **Networking** | Service names are hostnames within Compose |
| **Security** | Scan images, run non-root, never embed secrets |
| **Caching** | Order Dockerfile layers from least to most frequently changed |

------------------------------------------------------------------------

## References

1.  [Docker Documentation](https://docs.docker.com/)

2.  [Dockerfile Best Practices](https://docs.docker.com/build/building/best-practices/)

3.  [Docker Compose Documentation](https://docs.docker.com/compose/)

4.  [Docker Security Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

## Practice

### Exercise 1: Containerize a FastAPI App

Given a FastAPI project with this structure:

```
myapp/
├── src/
│   ├── main.py
│   └── config.py
├── tests/
│   └── test_main.py
├── requirements.txt
└── README.md
```

**Tasks:**

1.  Write a `Dockerfile` following all best practices from this guide

2.  Write a `docker-compose.yml` with the app + PostgreSQL

3.  Verify the app starts and connects to the database

4.  Add a health check for the database service

### Exercise 2: Optimize Image Size

Start with this unoptimized Dockerfile:

``` dockerfile
FROM python:3.12
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0"]
```

**Tasks:**

1.  Measure the image size (`docker`` ``images`)

2.  Apply optimizations: slim base, multi-stage build, .dockerignore, non-root user

3.  Measure the new image size

4.  Target: reduce image size by at least 50%

### Exercise 3: Debug a Container

A container starts but the API returns 500 errors. Walk through these debugging steps:

1.  Check container logs (`docker`` ``logs`)

2.  Shell into the container and verify environment variables

3.  Test database connectivity from inside the container

4.  Check if the correct port is exposed

## Review Questions

1.  **What is the difference between a Docker image and a container?**

    - *Hint*: Think of the relationship between a class and an object in OOP.

2.  **Why should you copy `requirements.txt` before copying source code in a Dockerfile?**

    - *Hint*: Think about Docker layer caching and what triggers a rebuild.

3.  **What is a multi-stage build and when would you use one?**

    - *Hint*: Consider what tools you need at build time vs runtime.

4.  **What is the difference between a named volume and a bind mount? When would you use each?**

    - *Hint*: One is for persistent data, the other is for development convenience.

5.  **Why should containers run as a non-root user?**

    - *Hint*: Think about what happens if an attacker exploits a vulnerability in your app.

6.  **How do containers in the same Docker Compose file communicate with each other?**

    - *Hint*: Docker creates a network and uses service names as DNS hostnames.
