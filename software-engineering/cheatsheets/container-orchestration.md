# Container Orchestration with Docker Compose

## Introduction

As applications grow in complexity, they often require multiple services working together—a web application, a database, a cache layer, a message queue, and more. Managing these services individually with Docker commands becomes tedious and error-prone. Docker Compose solves this by allowing you to define and run multi-container applications with a single configuration file.

**Why Docker Compose Matters for AI/RAG Projects:**

- **Multi-service RAG stacks**: Combine your API, vector database, Redis cache, and embedding service

- **Development parity**: Replicate production environments locally

- **Consistent deployments**: Version-controlled infrastructure as code

- **Easy scaling**: Run multiple instances of services for testing

------------------------------------------------------------------------

## Docker Compose Basics

### What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services, networks, and volumes. Then, with a single command, you create and start all the services from your configuration.

**Key Concepts:**

- **Service**: A container running from a specific image with defined configuration

- **Project**: A collection of services defined in a compose file (typically named after the directory)

- **Network**: Isolated network for inter-service communication

- **Volume**: Persistent storage shared between containers or with the host

### Compose File Structure

``` yaml
# docker-compose.yml (Compose V2 format)
services:
  # Service definitions
  app:
    image: python:3.11-slim
    build: .
    ports:
      - "8000:8000"
    environment:
      - DEBUG=true
    depends_on:
      - database

  database:
    image: postgres:15
    volumes:
      - db_data:/var/lib/postgresql/data

# Named volumes
volumes:
  db_data:

# Custom networks (optional - default network is created automatically)
networks:
  backend:
    driver: bridge
```

Docker Compose V2 (the current standard) no longer requires the `version` key at the top of the file. The `compose.yaml` file now follows the open **[Compose Specification](https://compose-spec.io/)**, which is supported by other tools beyond Docker (like Podman Compose).

### Essential Docker Compose Commands

``` bash
# Start all services (detached mode)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v

# View running services
docker compose ps

# View logs for all services
docker compose logs

# Follow logs for specific service
docker compose logs -f app

# Execute command in running service
docker compose exec app bash

# Run one-off command
docker compose run --rm app python manage.py migrate

# Scale a service
docker compose up -d --scale worker=3

# View resource usage
docker compose top
```

------------------------------------------------------------------------

## Multi-service Architecture

### Typical Application Stack

A modern application typically consists of multiple services working together:

``` mermaid
        flowchart TD
    LB["Load Balancer<br>(nginx / traefik)"]
    LB --> A1["App 1 — FastAPI"]
    LB --> A2["App 2 — FastAPI"]
    LB --> A3["App 3 — FastAPI"]
    A1 & A2 & A3 --> PG["PostgreSQL<br>(Primary)"]
    A1 & A2 & A3 --> RD["Redis<br>(Cache)"]
    A1 & A2 & A3 --> QD["Qdrant /<br>Chroma DB"]
```

### Complete RAG Application Example

``` yaml
# docker-compose.yml
services:
  # ===================
  # Application Service
  # ===================
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: rag-api
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/ragdb
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_URL=http://qdrant:6333
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
      qdrant:
        condition: service_started
    volumes:
      - ./app:/app # Development: hot reload
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ===================
  # Database Service
  # ===================
  postgres:
    image: postgres:15-alpine
    container_name: rag-postgres
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=ragdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ===================
  # Cache Service
  # ===================
  redis:
    image: redis:7-alpine
    container_name: rag-redis
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ===================
  # Vector Database
  # ===================
  qdrant:
    image: qdrant/qdrant:latest
    container_name: rag-qdrant
    volumes:
      - qdrant_data:/qdrant/storage
    ports:
      - "6333:6333"
      - "6334:6334" # gRPC
    restart: unless-stopped

# ===================
# Persistent Volumes
# ===================
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  qdrant_data:
    driver: local
```

### Service Dependencies

Docker Compose supports dependency ordering with conditions:

``` yaml
services:
  app:
    depends_on:
      database:
        condition: service_healthy # Wait until healthy
      redis:
        condition: service_started # Just wait for start
      migrations:
        condition: service_completed_successfully # Wait for completion
```

**Dependency Conditions:**

| Condition | Behavior |
|----|----|
| `service_started` | Default; waits for container to start |
| `service_healthy` | Waits for healthcheck to pass |
| `service_completed_successfully` | Waits for container to complete with exit code 0 |

`depends_on` conditions only control **container startup order**. They do **not** guarantee the application inside the container is fully ready to accept connections. Your application must implement its own retry/backoff logic for connecting to dependencies.

``` python
# Example: Retry logic for database connection
import time
from sqlalchemy import create_engine
from sqlalchemy.exc import OperationalError

def connect_with_retry(url, max_retries=5):
    for attempt in range(max_retries):
        try:
            engine = create_engine(url)
            engine.connect()
            return engine
        except OperationalError:
            time.sleep(2 ** attempt)  # Exponential backoff
    raise Exception("Could not connect to database")
```

------------------------------------------------------------------------

## Networking

### Default Network Behavior

Docker Compose automatically creates a default network for your project. All services can communicate using their service names as hostnames.

``` yaml
services:
  app:
    # Can connect to postgres using hostname "postgres"
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/db

  postgres:
    image: postgres:15
    # Accessible at hostname "postgres" within the network
```

### Custom Networks

For more complex architectures, define custom networks:

``` yaml
services:
  # Frontend services
  web:
    networks:
      - frontend
      - backend

  # API services
  api:
    networks:
      - backend
      - database

  # Database - isolated from frontend
  postgres:
    networks:
      - database

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  database:
    driver: bridge
    internal: true # No external access
```

### Exposing Ports

``` yaml
services:
  app:
    ports:
      # HOST:CONTAINER
      - "8000:8000" # Expose on all interfaces
      - "127.0.0.1:8000:8000" # Expose only on localhost
      - "8001-8010:8001-8010" # Port range
    expose:
      - "9000" # Expose only to linked services, not host
```

------------------------------------------------------------------------

## Volumes and Data Persistence

### Volume Types

``` yaml
services:
  app:
    volumes:
      # Named volume (managed by Docker)
      - app_data:/app/data

      # Bind mount (host directory)
      - ./src:/app/src

      # Bind mount with read-only flag
      - ./config:/app/config:ro

      # Anonymous volume (not recommended)
      - /app/temp

volumes:
  app_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /path/on/host
```

### Data Initialization

Initialize databases with SQL scripts:

``` yaml
services:
  postgres:
    image: postgres:15
    volumes:
      # Scripts in this directory run on first startup
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      - postgres_data:/var/lib/postgresql/data
```

``` sql
-- init-scripts/01-schema.sql
CREATE TABLE IF NOT EXISTS documents (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding VECTOR(384),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops);
```

------------------------------------------------------------------------

## Environment Management

### Environment Variables

**Method 1: Inline in compose file**

``` yaml
services:
  app:
    environment:
      - DEBUG=true
      - LOG_LEVEL=info
```

**Method 2: Environment file**

``` yaml
services:
  app:
    env_file:
      - .env
      - .env.local # Overrides .env
```

``` bash
# .env
DATABASE_URL=postgresql://postgres:postgres@postgres:5432/db
REDIS_URL=redis://redis:6379/0
DEBUG=false
```

**Method 3: Variable substitution**

``` yaml
services:
  app:
    image: myapp:${VERSION:-latest}
    environment:
      - API_KEY=${API_KEY} # Must be set in shell or .env
```

### Secrets Management

For sensitive data, use Docker secrets (Swarm mode) or mount secret files:

``` yaml
services:
  app:
    environment:
      - DATABASE_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - ./secrets/api_key.txt:/run/secrets/api_key:ro

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

For production, avoid storing secrets in compose files or local files. Use:

- **Secrets managers**: HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager

- **Kubernetes secrets**: When deploying to K8s

- **CI/CD injection**: Inject at runtime via environment variables

The file-based approach above is suitable for local development only.

### Multiple Environments

Use override files for different environments:

``` bash
# Base configuration
docker-compose.yml

# Development overrides
docker-compose.override.yml  # Automatically loaded

# Production overrides
docker-compose.prod.yml
```

``` yaml
# docker-compose.yml (base)
services:
  app:
    image: myapp:latest
    environment:
      - NODE_ENV=production

# docker-compose.override.yml (development - auto-loaded)
services:
  app:
    build: .
    volumes:
      - ./src:/app/src  # Hot reload
    environment:
      - NODE_ENV=development
      - DEBUG=true

# docker-compose.prod.yml (production - explicit)
services:
  app:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

``` bash
# Development (uses docker-compose.yml + docker-compose.override.yml)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

------------------------------------------------------------------------

## Health Checks and Restart Policies

### Health Checks

``` yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s # How often to check
      timeout: 10s # Max time for check
      retries: 3 # Failures before unhealthy
      start_period: 40s # Grace period for startup

  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Restart Policies

``` yaml
services:
  app:
    restart: unless-stopped # Recommended for most services

  worker:
    restart: on-failure # Only restart on failure

  one-time-job:
    restart: "no" # Never restart
```

| Policy           | Behavior                                 |
|------------------|------------------------------------------|
| `no`             | Never restart (default)                  |
| `always`         | Always restart                           |
| `on-failure`     | Restart only on non-zero exit            |
| `unless-stopped` | Always restart unless explicitly stopped |

------------------------------------------------------------------------

## Development Workflow

### Hot Reload Setup

``` yaml
services:
  app:
    build: .
    volumes:
      - ./app:/app # Mount source code
      - /app/__pycache__ # Exclude pycache (anonymous volume)
    command: uvicorn main:app --host 0.0.0.0 --reload
    environment:
      - PYTHONDONTWRITEBYTECODE=1
```

### Running Migrations

``` yaml
services:
  migrations:
    build: .
    command: alembic upgrade head
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/db
    depends_on:
      postgres:
        condition: service_healthy

  app:
    depends_on:
      migrations:
        condition: service_completed_successfully
```

### Debugging

``` yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
      - "5678:5678" # Debugger port
    command: python -m debugpy --listen 0.0.0.0:5678 -m uvicorn main:app --host 0.0.0.0
```

### Live File Sync with Watch

The `watch` command provides automatic file synchronization without bind mounts, offering better performance especially on macOS and Windows:

``` yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    develop:
      watch:
        # Sync source code changes instantly
        - action: sync
          path: ./src
          target: /app/src
          ignore:
            - __pycache__/

        # Sync+restart on config changes
        - action: sync+restart
          path: ./config
          target: /app/config

        # Rebuild container on dependency changes
        - action: rebuild
          path: pyproject.toml
```

``` bash
# Start with watch mode
docker compose watch

# Or run in background
docker compose up -d && docker compose watch
```

**Watch Actions:**

| Action         | Behavior                                    |
|----------------|---------------------------------------------|
| `sync`         | Copies files into container without restart |
| `sync+restart` | Copies files and restarts the service       |
| `rebuild`      | Rebuilds and replaces the container         |

`watch` is faster than bind mounts on macOS/Windows because it uses efficient file transfer instead of volume mounting.

------------------------------------------------------------------------

## Production Considerations

### Resource Limits

``` yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 512M
```

The `deploy` section (including `replicas`, `resources`, `restart_policy`) is **only fully applied with Docker Swarm** (`docker`` ``stack`` ``deploy`).

When using standalone `docker`` ``compose`` ``up`, many `deploy` settings are **ignored**. For standalone Compose, use service-level settings:

``` yaml
services:
  app:
    # Standalone Compose resource limits (works without Swarm)
    cpus: "1.0"
    mem_limit: 1G
    mem_reservation: 512M
```

### Logging Configuration

``` yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # Or use external logging
  app-syslog:
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://192.168.0.42:123"
```

### Production Compose File

``` yaml
# docker-compose.prod.yml
services:
  app:
    image: registry.example.com/myapp:${VERSION}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "5"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
```

------------------------------------------------------------------------

## Compose Profiles

Use profiles to selectively start services:

``` yaml
services:
  app:
    # Always starts (no profile)
    build: .

  postgres:
    # Always starts
    image: postgres:15

  redis:
    # Only with specific profile
    image: redis:7
    profiles:
      - cache

  debug-tools:
    image: alpine
    profiles:
      - debug
    command: sleep infinity
```

``` bash
# Start only default services
docker compose up

# Start with cache profile
docker compose --profile cache up

# Start with multiple profiles
docker compose --profile cache --profile debug up
```

------------------------------------------------------------------------

## Summary

**Key Takeaways:**

1.  **Docker Compose Basics**

    - Define multi-container apps in a single YAML file

    - Services communicate via service names as hostnames

    - Use `docker`` ``compose`` ``up`` ``-d` to start all services

2.  **Service Configuration**

    - Use `depends_on` with conditions for proper startup order

    - Configure health checks for reliable service discovery

    - Set appropriate restart policies for production

3.  **Networking**

    - Default network allows all services to communicate

    - Use custom networks to isolate service groups

    - Expose ports explicitly for external access

4.  **Data Management**

    - Use named volumes for persistent data

    - Use bind mounts for development (hot reload)

    - Initialize databases with scripts in entrypoint directories

5.  **Environment Management**

    - Use `.env` files for configuration

    - Use override files for different environments

    - Never commit secrets to version control

6.  **Production Readiness**

    - Set resource limits

    - Configure logging with rotation

    - Use health checks and restart policies

    - Run containers as non-root users (see Docker security docs)

    - Use secrets managers for sensitive configuration

------------------------------------------------------------------------

## References

1.  [Docker Compose Documentation](https://docs.docker.com/compose/)

2.  [Compose File Reference](https://docs.docker.com/compose/compose-file/)

3.  [Compose Specification](https://compose-spec.io/)

4.  [Compose Networking](https://docs.docker.com/compose/networking/)

5.  [Compose in Production](https://docs.docker.com/compose/production/)

6.  [Docker Compose Best Practices](https://docs.docker.com/develop/dev-best-practices/)

7.  [12-Factor App Methodology](https://12factor.net/)

## Practice

This practice guide contains hands-on exercises to reinforce your understanding of Docker Compose. Complete each exercise in order to build your skills progressively.

- Docker and Docker Compose installed

- Completed Docker Fundamentals exercises

- Basic understanding of networking concepts

------------------------------------------------------------------------

### Exercise 1: Docker Compose Basics

**Objective**: Create your first multi-container application with Docker Compose.

**Skills Practiced**:

- Writing docker-compose.yml

- Starting and stopping services

- Viewing logs and status

### Steps

``` bash
# 1. Create project directory
mkdir compose-basics
cd compose-basics

# 2. Create a simple FastAPI web app
cat > app.py << 'EOF'
from fastapi import FastAPI
import os
import socket

app = FastAPI()

@app.get("/")
def hello():
    return {
        "message": "Hello from Docker Compose!",
        "hostname": socket.gethostname(),
        "environment": os.getenv("APP_ENV", "development")
    }

@app.get("/health")
def health():
    return {"status": "healthy"}
EOF

# 3. Create pyproject.toml
cat > pyproject.toml << 'EOF'
fastapi==0.109.0
uvicorn[standard]==0.27.0
EOF

# 4. Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY pyproject.toml .
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync --frozen
COPY app.py .
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
EOF

# 5. Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - APP_ENV=docker-compose
EOF

# 6. Build and start services
docker compose up -d --build

# 7. Check running services
docker compose ps

# 8. View logs
docker compose logs web

# 9. Test the application
curl http://localhost:8000
curl http://localhost:8000/health

# 10. Stop services
docker compose down
```

### Verification Checklist

Service starts successfully

Application responds on port 8000

Environment variable is passed correctly

Logs are visible with `docker`` ``compose`` ``logs`

------------------------------------------------------------------------

### Exercise 2: Multi-service Application

**Objective**: Build an application with multiple interconnected services.

**Skills Practiced**:

- Service dependencies

- Inter-service communication

- Named volumes

### Steps

``` bash
# 1. Create project directory
mkdir multi-service-app
cd multi-service-app

# 2. Create the FastAPI application with Redis counter
cat > app.py << 'EOF'
from fastapi import FastAPI
from contextlib import asynccontextmanager
import redis.asyncio as redis
import os

# Redis connection
redis_client = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global redis_client
    redis_client = redis.from_url(
        os.getenv("REDIS_URL", "redis://redis:6379/0"),
        decode_responses=True
    )
    yield
    await redis_client.close()

app = FastAPI(lifespan=lifespan)

@app.get("/")
async def index():
    count = await redis_client.incr("visits")
    return {
        "message": "Hello from FastAPI + Redis!",
        "visits": count
    }

@app.get("/health")
async def health():
    try:
        await redis_client.ping()
        return {"status": "healthy", "redis": "connected"}
    except:
        return {"status": "unhealthy", "redis": "disconnected"}

@app.post("/reset")
async def reset():
    await redis_client.set("visits", 0)
    return {"message": "Counter reset", "visits": 0}
EOF

# 3. Create pyproject.toml
cat > pyproject.toml << 'EOF'
fastapi==0.109.0
uvicorn[standard]==0.27.0
redis==5.0.0
EOF

# 4. Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
COPY pyproject.toml .
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync --frozen
COPY app.py .
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
EOF

# 5. Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

volumes:
  redis_data:
EOF

# 6. Start all services
docker compose up -d --build

# 7. Wait for health checks
sleep 10
docker compose ps

# 8. Test the application multiple times
echo "=== Testing visit counter ==="
for i in {1..5}; do
    curl -s http://localhost:8000 | jq .
    sleep 1
done

# 9. Check health endpoint
curl http://localhost:8000/health | jq .

# 10. View Redis data persistence
docker compose exec redis redis-cli GET visits

# 11. Restart services and check persistence
docker compose restart web
sleep 5
curl http://localhost:8000 | jq .
# Counter should continue from previous value

# 12. Clean up
docker compose down
# Note: Volume preserved, counter will persist

# 13. Start again and verify persistence
docker compose up -d
sleep 5
curl http://localhost:8000 | jq .
# Should continue counting

# 14. Full cleanup including volumes
docker compose down -v
```

### Expected Output

``` json
{
  "message": "Hello from FastAPI + Redis!",
  "visits": 5
}
```

------------------------------------------------------------------------

### Exercise 3: Complete RAG Stack

**Objective**: Deploy a complete RAG application stack with API, database, cache, and vector store.

**Skills Practiced**:

- Complex multi-service orchestration

- Database initialization

- Environment configuration

- Service health dependencies

### Steps

``` bash
# 1. Create project directory
mkdir rag-stack
cd rag-stack

# 2. Create the RAG API application
cat > app.py << 'EOF'
import os
from typing import Optional
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import redis.asyncio as redis
import asyncpg

# Configuration from environment
DATABASE_URL = os.getenv("DATABASE_URL")
REDIS_URL = os.getenv("REDIS_URL", "redis://redis:6379/0")

# Connection pools
db_pool = None
redis_client = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global db_pool, redis_client
    # Initialize connections
    db_pool = await asyncpg.create_pool(DATABASE_URL)
    redis_client = redis.from_url(REDIS_URL, decode_responses=True)
    yield
    # Cleanup
    await db_pool.close()
    await redis_client.close()

app = FastAPI(title="RAG API", version="1.0.0", lifespan=lifespan)

class Document(BaseModel):
    content: str
    metadata: Optional[dict] = {}

class Query(BaseModel):
    question: str
    top_k: int = 5

@app.get("/")
def root():
    return {"service": "RAG API", "status": "running"}

@app.get("/health")
async def health():
    status = {"api": "healthy"}

    # Check Redis
    try:
        await redis_client.ping()
        status["redis"] = "connected"
    except:
        status["redis"] = "disconnected"

    # Check PostgreSQL
    try:
        async with db_pool.acquire() as conn:
            await conn.fetchval("SELECT 1")
        status["postgres"] = "connected"
    except Exception as e:
        status["postgres"] = f"error: {str(e)}"

    return status

@app.post("/documents")
async def create_document(doc: Document):
    # Check cache first
    cache_key = f"doc:{hash(doc.content)}"
    cached = await redis_client.get(cache_key)
    if cached:
        return {"message": "Document already exists", "cached": True}

    # Insert into database
    async with db_pool.acquire() as conn:
        doc_id = await conn.fetchval(
            "INSERT INTO documents (content, metadata) VALUES ($1, $2) RETURNING id",
            doc.content, str(doc.metadata)
        )

    # Cache the result
    await redis_client.setex(cache_key, 3600, str(doc_id))

    return {"id": doc_id, "message": "Document created"}

@app.get("/documents")
async def list_documents():
    # Try cache first
    cached = await redis_client.get("documents:list")
    if cached:
        return {"documents": eval(cached), "cached": True}

    async with db_pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT id, content, metadata FROM documents ORDER BY id DESC LIMIT 100"
        )
        docs = [dict(row) for row in rows]

    # Cache for 60 seconds
    await redis_client.setex("documents:list", 60, str(docs))

    return {"documents": docs, "cached": False}

@app.post("/query")
async def query_documents(q: Query):
    async with db_pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT id, content FROM documents WHERE content ILIKE $1 LIMIT $2",
            f"%{q.question}%", q.top_k
        )
        results = [dict(row) for row in rows]

    return {"query": q.question, "results": results}

@app.delete("/cache")
async def clear_cache():
    await redis_client.flushdb()
    return {"message": "Cache cleared"}
EOF

# 3. Create pyproject.toml
cat > pyproject.toml << 'EOF'
fastapi==0.109.0
uvicorn[standard]==0.27.0
redis==5.0.0
asyncpg==0.29.0
pydantic==2.5.0
EOF

# 4. Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# Install curl for healthcheck
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync --frozen

COPY app.py .

RUN useradd -m appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
EOF

# 5. Create database initialization script
mkdir -p init-scripts
cat > init-scripts/01-schema.sql << 'EOF'
-- Create documents table
CREATE TABLE IF NOT EXISTS documents (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    metadata TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create index for text search
CREATE INDEX IF NOT EXISTS idx_documents_content ON documents USING gin(to_tsvector('english', content));

-- Insert sample data
INSERT INTO documents (content, metadata) VALUES
    ('Docker is a containerization platform that packages applications with dependencies.', '{"source": "docker-docs"}'),
    ('Kubernetes is a container orchestration platform for managing containerized workloads.', '{"source": "k8s-docs"}'),
    ('Redis is an in-memory data structure store used as cache and message broker.', '{"source": "redis-docs"}'),
    ('PostgreSQL is a powerful open-source relational database system.', '{"source": "postgres-docs"}');
EOF

# 6. Create .env file
cat > .env << 'EOF'
POSTGRES_USER=raguser
POSTGRES_PASSWORD=ragpassword
POSTGRES_DB=ragdb
DATABASE_URL=postgresql://raguser:ragpassword@postgres:5432/ragdb
REDIS_URL=redis://redis:6379/0
EOF

# 7. Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  # ===================
  # API Service
  # ===================
  api:
    build: .
    container_name: rag-api
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    restart: unless-stopped

  # ===================
  # PostgreSQL Database
  # ===================
  postgres:
    image: postgres:16-alpine
    container_name: rag-postgres
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ===================
  # Redis Cache
  # ===================
  redis:
    image: redis:7-alpine
    container_name: rag-redis
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ===================
  # Adminer (DB UI)
  # ===================
  adminer:
    image: adminer
    container_name: rag-adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    profiles:
      - tools

volumes:
  postgres_data:
  redis_data:
EOF

# 8. Start all services
docker compose up -d --build

# 9. Wait for services to be healthy
echo "Waiting for services to start..."
sleep 15

# 10. Check service status
docker compose ps

# 11. Check health endpoint
echo "=== Health Check ==="
curl -s http://localhost:8000/health | jq .

# 12. List pre-loaded documents
echo "=== Pre-loaded Documents ==="
curl -s http://localhost:8000/documents | jq .

# 13. Add a new document
echo "=== Adding Document ==="
curl -s -X POST http://localhost:8000/documents \
    -H "Content-Type: application/json" \
    -d '{"content": "FastAPI is a modern Python web framework for building APIs.", "metadata": {"source": "fastapi-docs"}}' | jq .

# 14. Query documents
echo "=== Querying Documents ==="
curl -s -X POST http://localhost:8000/query \
    -H "Content-Type: application/json" \
    -d '{"question": "container", "top_k": 3}' | jq .

# 15. Start optional tools
echo "=== Starting Adminer (DB UI) ==="
docker compose --profile tools up -d adminer
echo "Adminer available at http://localhost:8080"

# 16. View logs
docker compose logs api --tail 20

# 17. Clean up
# docker compose down -v
```

### Verification Checklist

All services start and become healthy

API responds with connected status for Redis and PostgreSQL

Pre-loaded documents are visible

New documents can be added

Query returns relevant results

Cache is working (repeat requests are faster)

------------------------------------------------------------------------

### Exercise 4: Development vs Production Configuration

**Objective**: Set up different configurations for development and production environments.

**Skills Practiced**:

- Override files

- Environment-specific configuration

- Development workflow optimization

### Steps

``` bash
# 1. Create project directory
mkdir env-configs
cd env-configs

# 2. Create FastAPI application
cat > app.py << 'EOF'
from fastapi import FastAPI
import os

app = FastAPI()

@app.get("/")
def index():
    return {
        "environment": os.getenv("APP_ENV", "production"),
        "debug": os.getenv("DEBUG", "false"),
        "database": os.getenv("DATABASE_URL", "not configured"),
        "features": {
            "hot_reload": os.getenv("HOT_RELOAD", "false"),
            "debug_toolbar": os.getenv("DEBUG_TOOLBAR", "false")
        }
    }

@app.get("/health")
def health():
    return {"status": "healthy"}
EOF

cat > pyproject.toml << 'EOF'
fastapi==0.109.0
uvicorn[standard]==0.27.0
EOF

# 3. Create base Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
COPY pyproject.toml .
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync --frozen
COPY . .
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
EOF

# 4. Create base docker-compose.yml
cat > docker-compose.yml << 'EOF'
# Base configuration - shared across environments
services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - APP_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/prod
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=prod
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
EOF

# 5. Create development override (auto-loaded)
cat > docker-compose.override.yml << 'EOF'
# Development overrides - automatically loaded with docker-compose.yml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./app.py:/app/app.py  # Hot reload
    environment:
      - APP_ENV=development
      - DEBUG=true
      - DATABASE_URL=postgresql://user:pass@db:5432/dev
      - HOT_RELOAD=true
      - DEBUG_TOOLBAR=true
    command: uvicorn app:app --host 0.0.0.0 --port 8000 --reload

  db:
    environment:
      - POSTGRES_DB=dev
    ports:
      - "5432:5432"  # Expose for local tools
EOF

# 6. Create production configuration
cat > docker-compose.prod.yml << 'EOF'
# Production configuration
services:
  app:
    image: myapp:${VERSION:-latest}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    environment:
      - APP_ENV=production
      - DEBUG=false
      - DATABASE_URL=${DATABASE_URL}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  db:
    # In production, typically use managed database
    # This is just for demonstration
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
EOF

# 7. Create environment files
cat > .env.dev << 'EOF'
APP_ENV=development
DEBUG=true
DATABASE_URL=postgresql://user:pass@db:5432/dev
EOF

cat > .env.prod << 'EOF'
APP_ENV=production
DEBUG=false
DATABASE_URL=postgresql://user:prodpass@db:5432/prod
VERSION=1.0.0
EOF

# 8. Run development environment (default)
echo "=== Starting Development Environment ==="
docker compose up -d --build

sleep 5
echo "Development config:"
curl -s http://localhost:8000 | jq .

docker compose down

# 9. Run production environment
echo "=== Starting Production Environment ==="
docker compose -f docker-compose.yml -f docker-compose.prod.yml --env-file .env.prod up -d --build

sleep 5
echo "Production config:"
curl -s http://localhost:8000 | jq .

# 10. Clean up
docker compose -f docker-compose.yml -f docker-compose.prod.yml down -v
```

### Expected Output

**Development:**

``` json
{
  "environment": "development",
  "debug": "true",
  "features": {
    "hot_reload": "true",
    "debug_toolbar": "true"
  }
}
```

**Production:**

``` json
{
  "environment": "production",
  "debug": "false",
  "features": {
    "hot_reload": "false",
    "debug_toolbar": "false"
  }
}
```

------------------------------------------------------------------------

### Exercise 5: Scaling and Load Balancing

**Objective**: Scale services and implement basic load balancing with nginx.

**Skills Practiced**:

- Service scaling

- Load balancer configuration

- Round-robin distribution

### Steps

``` bash
# 1. Create project directory
mkdir scaling-demo
cd scaling-demo

# 2. Create FastAPI app that shows instance ID
cat > app.py << 'EOF'
from fastapi import FastAPI
import socket

app = FastAPI()
instance_id = socket.gethostname()
request_count = 0

@app.get("/")
def index():
    global request_count
    request_count += 1
    return {
        "instance": instance_id,
        "requests_handled": request_count
    }

@app.get("/health")
def health():
    return {"status": "healthy", "instance": instance_id}
EOF

cat > pyproject.toml << 'EOF'
fastapi==0.109.0
uvicorn[standard]==0.27.0
EOF

cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY pyproject.toml .
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync --frozen
COPY app.py .
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
EOF

# 3. Create nginx configuration
cat > nginx.conf << 'EOF'
upstream backend {
    server app:8000;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
        access_log off;
        proxy_pass http://backend/health;
    }
}
EOF

# 4. Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app

  app:
    build: .
    expose:
      - "8000"
    deploy:
      replicas: 3

EOF

# 5. Start with multiple replicas
docker compose up -d --build --scale app=3

# 6. Wait for startup
sleep 10

# 7. Check running instances
docker compose ps

# 8. Test load balancing
echo "=== Testing Load Balancing ==="
for i in {1..10}; do
    curl -s http://localhost/ | jq -r '.instance'
    sleep 0.5
done

# 9. View all instance responses
echo ""
echo "=== Full responses ==="
for i in {1..6}; do
    curl -s http://localhost/ | jq .
done

# 10. Scale up dynamically
echo ""
echo "=== Scaling to 5 replicas ==="
docker compose up -d --scale app=5
sleep 5
docker compose ps

# 11. Scale down
echo ""
echo "=== Scaling to 2 replicas ==="
docker compose up -d --scale app=2
docker compose ps

# 12. Clean up
docker compose down
```

### Expected Output

```
=== Testing Load Balancing ===
abc123def456
xyz789ghi012
abc123def456
xyz789ghi012
...
```

The instance IDs rotate, showing requests are distributed across containers.

------------------------------------------------------------------------

### Exercise 6: Complete Development Workflow

**Objective**: Set up a complete development workflow with logs, debugging, and database management.

**Skills Practiced**:

- Development tools integration

- Log aggregation

- Database management UI

- Debugging workflow

### Steps

``` bash
# 1. Create project directory
mkdir dev-workflow
cd dev-workflow

# 2. Create FastAPI application
cat > app.py << 'EOF'
from fastapi import FastAPI
from contextlib import asynccontextmanager
import redis.asyncio as redis
import asyncpg
import os

db_pool = None
redis_client = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global db_pool, redis_client
    db_pool = await asyncpg.create_pool(os.getenv("DATABASE_URL"))
    redis_client = redis.from_url(os.getenv("REDIS_URL"), decode_responses=True)
    yield
    await db_pool.close()
    await redis_client.close()

app = FastAPI(title="Dev Workflow Demo", lifespan=lifespan)

@app.get("/")
async def root():
    return {"message": "Development workflow demo"}

@app.get("/health")
async def health():
    try:
        await redis_client.ping()
        async with db_pool.acquire() as conn:
            await conn.fetchval("SELECT 1")
        return {"status": "healthy", "postgres": "ok", "redis": "ok"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
EOF

cat > pyproject.toml << 'EOF'
fastapi==0.109.0
uvicorn[standard]==0.27.0
redis==5.0.0
asyncpg==0.29.0
EOF

cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
COPY pyproject.toml .
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync --frozen
COPY app.py .
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
EOF

# 3. Create comprehensive docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  # Main application
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://dev:devpass@postgres:5432/devdb
      - REDIS_URL=redis://redis:6379/0
    volumes:
      - ./app.py:/app/app.py  # Hot reload
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=dev
      - POSTGRES_PASSWORD=devpass
      - POSTGRES_DB=devdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d devdb"]
      interval: 5s
      timeout: 3s
      retries: 5

  # Redis
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # Database Admin UI
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    profiles:
      - tools

  # Redis Commander (Redis UI)
  redis-commander:
    image: rediscommander/redis-commander:latest
    environment:
      - REDIS_HOSTS=local:redis:6379
    ports:
      - "8081:8081"
    depends_on:
      - redis
    profiles:
      - tools

volumes:
  postgres_data:
  redis_data:
EOF

# 4. Start main services
docker compose up -d --build

# 5. Start development tools
docker compose --profile tools up -d

# 6. Display access information
echo "
=== Development Environment Ready ===

Services:
  - API:             http://localhost:8000
  - Health:          http://localhost:8000/health
  - Docs (Swagger):  http://localhost:8000/docs
  - ReDoc:           http://localhost:8000/redoc

Database Tools:
  - Adminer:         http://localhost:8080
    System:          PostgreSQL
    Server:          postgres
    Username:        dev
    Password:        devpass
    Database:        devdb

  - Redis Commander: http://localhost:8081

Commands:
  - View logs:           docker compose logs -f app
  - Enter app container: docker compose exec app bash
  - Enter postgres:      docker compose exec postgres psql -U dev -d devdb
  - Enter redis:         docker compose exec redis redis-cli

"

# 7. Useful development commands
echo "=== Testing API ==="
curl -s http://localhost:8000 | jq .
curl -s http://localhost:8000/health | jq .

# 8. Database operations
echo ""
echo "=== Direct Database Access ==="
docker compose exec postgres psql -U dev -d devdb -c "\dt"

# 9. Redis operations
echo ""
echo "=== Direct Redis Access ==="
docker compose exec redis redis-cli KEYS "*"
docker compose exec redis redis-cli INFO memory | head -5

# 10. Tail logs
echo ""
echo "=== Application Logs (Ctrl+C to exit) ==="
docker compose logs -f app --tail 10

# Cleanup command (run manually)
# docker compose --profile tools down -v
```

### Verification Checklist

All services running and healthy

FastAPI Swagger docs accessible at /docs

Adminer accessible and can connect to PostgreSQL

Redis Commander showing Redis data

Hot reload working (change app.py and test)

Direct database/redis access working

------------------------------------------------------------------------

### Summary Checklist

After completing all exercises, verify you can:

Write docker-compose.yml from scratch

Configure multi-service applications with FastAPI

Set up service dependencies with health checks

Configure volumes for data persistence

Use environment files and overrides

Scale services horizontally

Configure nginx as load balancer

Set up development tools (Adminer, Redis Commander)

------------------------------------------------------------------------

### Additional Challenges

1.  **Add monitoring**: Integrate Prometheus and Grafana for metrics

2.  **Log aggregation**: Add Loki for centralized logging

3.  **CI/CD integration**: Create GitHub Actions workflow to build and test

4.  **Kubernetes migration**: Convert docker-compose.yml to Kubernetes manifests

## Review Questions

Test your understanding of Docker Compose concepts with these review questions.

------------------------------------------------------------------------

## Multiple Choice

### 1. How do services communicate with each other in Docker Compose by default?

- A\) Via IP addresses only

- B\) Using service names as hostnames

- C\) Through localhost

- D\) They cannot communicate by default

Answer

**B) Using service names as hostnames**

Docker Compose creates a default network where services can reach each other using their service names as DNS hostnames. For example, a service named `postgres` can be reached at hostname `postgres`.

------------------------------------------------------------------------

### 2. What does the `depends_on` directive do?

- A\) Ensures services are on the same network

- B\) Controls the order in which services start

- C\) Shares volumes between services

- D\) Links environment variables between services

Answer

**B) Controls the order in which services start**

`depends_on` specifies that a service should wait for another service to start before starting itself. With conditions like `service_healthy`, it can also wait for health checks to pass.

------------------------------------------------------------------------

### 3. What is the difference between `ports` and `expose` in docker-compose.yml?

- A\) There is no difference

- B\) `ports` publishes to host, `expose` only exposes to other services in the network

- C\) `expose` publishes to host, `ports` only exposes to other services

- D\) `ports` is for TCP, `expose` is for UDP

Answer

**B) `ports` publishes to host, `expose` only exposes to other services in the network**

- `ports` maps container ports to host ports, making them accessible from outside

- `expose` only documents the port and makes it accessible to linked services, not the host

------------------------------------------------------------------------

### 4. What happens to named volumes when you run `docker`` ``compose`` ``down`?

- A\) They are always deleted

- B\) They are preserved unless you use `-v` flag

- C\) They are moved to a backup location

- D\) They are converted to bind mounts

Answer

**B) They are preserved unless you use `-v` flag**

Named volumes persist data even when containers are removed. To delete volumes along with containers, use `docker`` ``compose`` ``down`` ``-v`.

------------------------------------------------------------------------

### 5. What file is automatically loaded with docker-compose.yml?

- A\) docker-compose.local.yml

- B\) docker-compose.override.yml

- C\) docker-compose.dev.yml

- D\) docker-compose.default.yml

Answer

**B) docker-compose.override.yml**

Docker Compose automatically loads `docker-compose.override.yml` if it exists alongside `docker-compose.yml`. This is useful for development-specific configurations.

------------------------------------------------------------------------

### 6. Which dependency condition waits for a service’s health check to pass?

- A\) `service_started`

- B\) `service_running`

- C\) `service_healthy`

- D\) `service_ready`

Answer

**C) `service_healthy`**

``` yaml
depends_on:
  database:
    condition: service_healthy
```

This waits for the database service’s health check to pass before starting the dependent service.

------------------------------------------------------------------------

### 7. How do you scale a service to 5 instances?

- A\) `docker`` ``compose`` ``scale`` ``service=5`

- B\) `docker`` ``compose`` ``up`` ``-d`` ``--scale`` ``service=5`

- C\) `docker`` ``compose`` ``replicas`` ``service`` ``5`

- D\) `docker`` ``compose`` ``start`` ``service`` ``-n`` ``5`

Answer

**B) `docker`` ``compose`` ``up`` ``-d`` ``--scale`` ``service=5`**

The `--scale` flag allows you to run multiple instances of a service. Note that when scaling, you should use `expose` instead of fixed `ports` to avoid port conflicts.

------------------------------------------------------------------------

### 8. What is the purpose of profiles in Docker Compose?

- A\) To define security settings

- B\) To selectively start certain services

- C\) To configure logging

- D\) To set resource limits

Answer

**B) To selectively start certain services**

Profiles allow you to group services and only start them when that profile is activated:

``` yaml
services:
  debug-tools:
    profiles:
      - debug
```

Start with: `docker`` ``compose`` ``--profile`` ``debug`` ``up`

------------------------------------------------------------------------

## Scenario Questions

### 9. Database Connection Refused

Your application container keeps failing with “connection refused” when trying to connect to PostgreSQL, even though `depends_on:`` ``[postgres]` is configured.

**What’s the likely issue and how do you fix it?**

Answer

**Issue**: `depends_on:`` ``[postgres]` only waits for the container to start, not for PostgreSQL to be ready to accept connections.

**Solution**: Use health check condition:

``` yaml
services:
  app:
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
```

This ensures the app only starts after PostgreSQL is actually ready to accept connections.

------------------------------------------------------------------------

### 10. Port Already in Use

You try to start your compose stack but get an error saying port 5432 is already in use.

**What are your options?**

Answer

**Options:**

1.  **Change the host port**:

    ``` yaml
    ports:
      - "5433:5432" # Use 5433 on host, 5432 in container
    ```

2.  **Stop the conflicting service**:

    ``` bash
    # Find what's using the port
    sudo lsof -i :5432
    # Stop local PostgreSQL
    sudo systemctl stop postgresql
    ```

3.  **Don’t expose to host** (if only needed internally):

    ``` yaml
    # Remove ports section, use expose instead
    expose:
      - "5432"
    ```

4.  **Use a different project name** to isolate networks:

    ``` bash
    docker compose -p myproject up
    ```

------------------------------------------------------------------------

### 11. Data Loss on Rebuild

Every time you rebuild your containers, all database data is lost.

**How do you fix this?**

Answer

**Issue**: Data is stored inside the container instead of a persistent volume.

**Solution**: Use named volumes for database data:

``` yaml
services:
  postgres:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data: # Named volume - persists across rebuilds
```

**Important notes:**

- Named volumes persist unless explicitly removed with `docker`` ``compose`` ``down`` ``-v`

- Use `docker`` ``volume`` ``ls` to see existing volumes

- Use `docker`` ``volume`` ``inspect`` ``volume_name` to see where data is stored

------------------------------------------------------------------------

### 12. Environment Variables Not Working

You set environment variables in `.env` file but they’re not being picked up by your application.

**What could be wrong?**

Answer

**Possible issues and solutions:**

1.  **Not referencing .env file**:

    ``` yaml
    services:
      app:
        env_file:
          - .env # Explicitly reference
    ```

2.  **Variable substitution vs passing**:

    ``` yaml
    # This substitutes at compose time (from shell or .env)
    environment:
      - DATABASE_URL=${DATABASE_URL}

    # This passes the file contents to the container
    env_file:
      - .env
    ```

3.  **Wrong file name**: Ensure it’s exactly `.env` not `.env.txt`

4.  **Rebuild required**:

    ``` bash
    docker compose up -d --build
    ```

5.  **Check variable is set**:

    ``` bash
    docker compose exec app env | grep DATABASE_URL
    ```

------------------------------------------------------------------------

## Command Identification

### 13. Match the command to its purpose:

| Command                                                  | Purpose |
|----------------------------------------------------------|---------|
| `docker`` ``compose`` ``logs`` ``-f`` ``app`             | ?       |
| `docker`` ``compose`` ``exec`` ``app`` ``bash`           | ?       |
| `docker`` ``compose`` ``run`` ``--rm`` ``app`` ``pytest` | ?       |
| `docker`` ``compose`` ``config`                          | ?       |

Answer

| Command | Purpose |
|----|----|
| `docker`` ``compose`` ``logs`` ``-f`` ``app` | Follow (stream) logs for the app service |
| `docker`` ``compose`` ``exec`` ``app`` ``bash` | Open interactive shell in running app container |
| `docker`` ``compose`` ``run`` ``--rm`` ``app`` ``pytest` | Run one-off command (pytest) and remove container after |
| `docker`` ``compose`` ``config` | Validate and view the merged compose configuration |

------------------------------------------------------------------------

### 14. What commands would you use to:

1.  Rebuild images and start all services

2.  Stop all services but keep volumes

3.  View resource usage of all containers

4.  Run database migrations before starting app

Answer

``` bash
# 1. Rebuild and start
docker compose up -d --build

# 2. Stop without removing volumes
docker compose down  # (default behavior preserves volumes)

# 3. View resource usage
docker compose top
# or
docker stats

# 4. Run migrations before app
docker compose run --rm app alembic upgrade head
docker compose up -d
```

------------------------------------------------------------------------

## Short Answer

### 15. Explain the difference between bind mounts and named volumes

Answer

**Bind Mounts:**

- Map a host directory to container path

- Format: `./host/path:/container/path`

- Managed by user on host filesystem

- Good for: development (hot reload), config files

- Example: `./src:/app/src`

**Named Volumes:**

- Managed by Docker

- Format: `volume_name:/container/path`

- Stored in Docker’s data directory

- Persist until explicitly deleted

- Good for: database data, persistent application data

- Example: `postgres_data:/var/lib/postgresql/data`

``` yaml
volumes:
  - ./app:/app # Bind mount
  - db_data:/var/lib/postgresql/data # Named volume
```

------------------------------------------------------------------------

### 16. How do you configure different environments (dev/staging/prod) with Docker Compose?

Answer

**Method 1: Override files**

``` bash
# docker-compose.yml (base)
# docker-compose.override.yml (dev - auto-loaded)
# docker-compose.prod.yml (production)

# Development (automatic override)
docker compose up

# Production (explicit files)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

**Method 2: Environment files**

``` bash
# .env.dev, .env.prod
docker compose --env-file .env.prod up
```

**Method 3: Profiles**

``` yaml
services:
  debug-tools:
    profiles: [dev]
  monitoring:
    profiles: [prod]
```

``` bash
docker compose --profile dev up
```

------------------------------------------------------------------------

### 17. What happens when you scale a service that has a fixed port mapping?

Answer

**Problem**: Port conflicts occur because you cannot bind the same host port to multiple containers.

``` yaml
# This will fail when scaled
ports:
  - "8000:8000"
```

**Solutions:**

1.  **Use port ranges**:

    ``` yaml
    ports:
      - "8000-8010:8000"
    ```

2.  **Use expose only (internal access)**:

    ``` yaml
    expose:
      - "8000"
    ```

3.  **Use a load balancer**:

    ``` yaml
    nginx:
      ports:
        - "80:80"
    app:
      expose:
        - "8000"
      deploy:
        replicas: 3
    ```

------------------------------------------------------------------------

### 18. What is the purpose of health checks in Docker Compose?

Answer

**Health checks serve multiple purposes:**

1.  **Dependency coordination**: With `condition:`` ``service_healthy`, dependent services wait until the dependency is actually ready, not just started

2.  **Container orchestration**: Docker can restart unhealthy containers (with appropriate restart policy)

3.  **Load balancer integration**: Load balancers can route traffic only to healthy containers

4.  **Monitoring**: Easily identify service health with `docker`` ``compose`` ``ps`

**Example:**

``` yaml
services:
  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

**States**: starting → healthy/unhealthy

------------------------------------------------------------------------

## Self-Assessment Checklist

Rate your confidence (1-5) on each skill:

| Skill                                  | Confidence |
|----------------------------------------|------------|
| Writing docker-compose.yml files       | ⬜         |
| Configuring multi-service applications | ⬜         |
| Setting up service dependencies        | ⬜         |
| Managing volumes and data persistence  | ⬜         |
| Environment configuration              | ⬜         |
| Scaling services                       | ⬜         |
| Debugging compose issues               | ⬜         |
| Production configuration               | ⬜         |

**Scoring**:

- **35-40**: Expert level - ready to orchestrate complex applications

- **25-34**: Proficient - comfortable with multi-service deployments

- **15-24**: Intermediate - practice more complex scenarios

- **Below 15**: Beginner - review documentation and redo exercises
