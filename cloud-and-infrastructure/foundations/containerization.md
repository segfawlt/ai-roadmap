# Docker Fundamentals & Best Practices

## Introduction

Docker has become an essential tool for modern software development, enabling developers to package applications with all their dependencies into standardized units called containers. For AI/ML applications and cloud-native development, understanding Docker is crucial for creating reproducible environments, simplifying deployment, and ensuring consistency across development, testing, and production.

**Why Docker Matters for AI/RAG Projects:**

- **Reproducibility**: Ensure your ML models run identically across all environments

- **Dependency Management**: Package complex Python dependencies, CUDA drivers, and system libraries together

- **Scalability**: Deploy containerized applications seamlessly to Kubernetes or cloud platforms

- **Isolation**: Run multiple versions of frameworks or conflicting dependencies simultaneously

------------------------------------------------------------------------

## Docker Basics

### Containers vs Virtual Machines

Understanding the difference between containers and VMs is fundamental to leveraging Docker effectively.

**Virtual Machines:**

- Run complete operating systems on virtualized hardware

- Include full OS kernel, system libraries, and applications

- Heavy resource consumption (gigabytes of memory per VM)

- Slower startup time (minutes)

- Stronger isolation through hardware-level virtualization

**Containers:**

- Share the host OS kernel

- Package only application and its dependencies

- Lightweight (megabytes instead of gigabytes)

- Fast startup (seconds or less)

- Process-level isolation using Linux namespaces and cgroups

``` mermaid
        block-beta
    columns 2
    block:vm:1
        columns 1
        VA["Application"]
        VL["Libraries"]
        VG["Guest OS"]
        VH["Hypervisor"]
        VO["Host OS"]
        VW["Hardware"]
    end
    block:ct:1
        columns 1
        CA["Application"]
        CL["Libraries"]
        CE["Container Engine"]
        CO["Host OS"]
        CW["Hardware"]
    end
    vm --- L1["Virtual Machine"]
    ct --- L2["Container"]
```

**When to Use Each:**

| Use Case                                     | Recommended      |
|----------------------------------------------|------------------|
| Running multiple apps with same dependencies | Containers       |
| Strong security isolation required           | Virtual Machines |
| Rapid scaling and deployment                 | Containers       |
| Running different operating systems          | Virtual Machines |
| Microservices architecture                   | Containers       |
| Legacy applications requiring full OS        | Virtual Machines |

### Core Docker Concepts

**Image**: A read-only template containing instructions for creating a Docker container. Think of it as a snapshot or blueprint.

**Container**: A runnable instance of an image. You can create, start, stop, and delete containers.

**Dockerfile**: A text file containing instructions to build a Docker image.

**Registry**: A repository for storing and distributing Docker images (Docker Hub, GitHub Container Registry, AWS ECR).

**Layer**: Each instruction in a Dockerfile creates a layer. Layers are cached and reused to speed up builds.

### Essential Docker Commands

``` bash
# Image Management
docker images                       # List all images
docker pull python:3.11-slim        # Download image from registry
docker build -t myapp:v1 .          # Build image from Dockerfile
docker rmi myapp:v1                 # Remove an image
docker tag myapp:v1 myapp:latest    # Tag an image

# Container Operations
docker run -d --name myapp myimage  # Run container in background
docker ps                           # List running containers
docker ps -a                        # List all containers (including stopped)
docker stop myapp                   # Stop a container
docker start myapp                  # Start a stopped container
docker rm myapp                     # Remove a container
docker logs myapp                   # View container logs
docker logs -f myapp                # Follow logs in real-time

# Interactive Commands
docker run -it python:3.11 bash     # Run with interactive terminal
docker exec -it myapp bash          # Execute command in running container

# Inspection & Debugging
docker inspect myapp                # View container details
docker stats                        # Real-time resource usage
docker top myapp                    # View running processes
docker diff myapp                   # View filesystem changes

# Cleanup
docker system prune                 # Remove unused data
docker image prune -a               # Remove all unused images
docker container prune              # Remove all stopped containers
```

### Port Mapping and Volumes

**Port Mapping**: Expose container ports to the host system.

``` bash
# Map host port 8080 to container port 80
docker run -d -p 8080:80 nginx

# Map to specific interface
docker run -d -p 127.0.0.1:8080:80 nginx

# Map multiple ports
docker run -d -p 8080:80 -p 8443:443 nginx
```

**Volumes**: Persist data beyond container lifecycle.

``` bash
# Named volume (managed by Docker)
docker run -v mydata:/app/data myimage

# Bind mount (host directory)
docker run -v /host/path:/container/path myimage

# Read-only mount
docker run -v /config:/app/config:ro myimage
```

------------------------------------------------------------------------

## Dockerfile Best Practices

### Basic Dockerfile Structure

A Dockerfile contains instructions that Docker executes sequentially to build an image.

``` dockerfile
# Use official Python runtime as base image
FROM python:3.11-slim

# Set metadata
LABEL maintainer="[email protected]"
LABEL version="1.0"

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Install uv for fast package management
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Set working directory
WORKDIR /app

# Copy dependency file first (for better caching)
COPY pyproject.toml .

# Install dependencies with uv (10-100x faster than pip)
RUN uv sync --frozen

# Copy application code
COPY . .

# Expose port
EXPOSE 8000

# Define health check
HEALTHCHECK --interval=30s --timeout=10s \
    CMD curl -f http://localhost:8000/health || exit 1

# Set default command
CMD ["python", "app.py"]
```

### Layer Optimization

Each `RUN`, `COPY`, and `ADD` instruction creates a new layer. Optimize layers to reduce image size and build time.

**❌ Bad Practice - Multiple RUN commands:**

``` dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN rm -rf /var/lib/apt/lists/*
```

**✅ Good Practice - Combined commands:**

``` dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*
```

### Dependency Caching

Docker caches layers and reuses them if the instruction and context haven’t changed. Order your Dockerfile to maximize cache hits.

**❌ Bad - Full rebuild on any code change:**

``` dockerfile
FROM python:3.11-slim
COPY . /app
WORKDIR /app
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync
CMD ["python", "app.py"]
```

**✅ Good - Dependencies cached unless pyproject.toml/uv.lock changes:**

``` dockerfile
# syntax=docker/dockerfile:1
FROM python:3.11-slim
WORKDIR /app

# Install uv for fast package management
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Copy dependency files (both for reproducible builds)
COPY pyproject.toml uv.lock ./

# Use BuildKit cache mount for even faster rebuilds
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen

# Then copy application code
COPY . .
CMD ["python", "app.py"]
```

Always commit `uv.lock` to version control. It ensures reproducible builds by locking exact dependency versions. Generate it with `uv`` ``lock`.

### Choosing Base Images

Select appropriate base images for your use case:

| Base Image                  | Size   | Use Case                    |
|-----------------------------|--------|-----------------------------|
| `python:3.13`               | ~900MB | Development with full tools |
| `python:3.13-slim`          | ~150MB | Production without extras   |
| `python:3.13-alpine`        | ~50MB  | Minimal size, musl libc     |
| `gcr.io/distroless/python3` | ~50MB  | Maximum security, no shell  |
| `nvidia/cuda:12.4-runtime`  | ~3GB   | GPU workloads               |

**Recommendations:**

- Start with `-slim` variants for production

- Avoid `alpine` for Python (pip compile issues, slower builds)

- Use specific version tags, not `latest`

- Consider distroless images for maximum security

For maximum reproducibility and security in production, use **fully versioned tags**:

``` dockerfile
# ❌ Avoid floating tags (can change unexpectedly)
FROM python:3.13-slim

# ✅ Use fully pinned tags for production
FROM python:3.13.2-slim-bookworm
```

The fully versioned tag ensures your builds are deterministic and won’t break due to upstream changes.

### .dockerignore File

Create a `.dockerignore` file to exclude unnecessary files from the build context.

```
# Version control
.git
.gitignore

# Python artifacts
__pycache__
*.pyc
*.pyo
*.pyd
.Python
*.so

# Virtual environments
venv/
.venv/
ENV/

# IDE and editors
.vscode/
.idea/
*.swp

# Testing and coverage
.pytest_cache/
.coverage
htmlcov/

# Build artifacts
build/
dist/
*.egg-info/

# Environment files
.env
.env.local
*.local

# Documentation
docs/
*.md
!README.md

# Docker files (avoid recursion)
Dockerfile*
docker-compose*.yml
```

------------------------------------------------------------------------

## Multi-stage Builds

Multi-stage builds allow you to use multiple `FROM` statements in a Dockerfile, each beginning a new stage. You can selectively copy artifacts from one stage to another, leaving behind everything you don’t need.

### Why Multi-stage Builds?

**Benefits:**

- **Smaller final images**: Only runtime dependencies included

- **Separation of concerns**: Build tools stay in build stage

- **Security**: Fewer packages means smaller attack surface

- **Faster deployments**: Smaller images transfer faster

### Basic Multi-stage Pattern

``` dockerfile
# =============
# Build Stage
# =============
FROM python:3.11 AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install Python dependencies
COPY pyproject.toml .
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync --frozen

# =============
# Runtime Stage
# =============
FROM python:3.11-slim

WORKDIR /app

# Install only runtime dependencies
RUN apt-get update && apt-get install -y \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Copy virtual environment from builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy application code
COPY . .

# Run as non-root user
RUN useradd -m appuser
USER appuser

EXPOSE 8000
CMD ["python", "app.py"]
```

### CUDA Multi-stage Build (AI/ML Workloads)

For GPU-based AI applications, separate CUDA dependencies from runtime:

``` dockerfile
# syntax=docker/dockerfile:1
# =============
# Build Stage (with full CUDA toolkit)
# =============
FROM nvidia/cuda:12.4.0-devel-ubuntu22.04 AS builder

WORKDIR /app

# Install Python and build tools
RUN apt-get update && apt-get install -y \
    python3.12 python3.12-venv python3-pip \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Create venv and install dependencies
RUN python3.12 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Compile any CUDA extensions
COPY . .
RUN python setup.py build_ext --inplace

# =============
# Runtime Stage (minimal CUDA runtime only)
# =============
FROM nvidia/cuda:12.4.0-runtime-ubuntu22.04

WORKDIR /app

# Install only Python runtime (no dev tools)
RUN apt-get update && apt-get install -y \
    python3.12 \
    && rm -rf /var/lib/apt/lists/*

# Copy venv and compiled extensions from builder
COPY --from=builder /opt/venv /opt/venv
COPY --from=builder /app .
ENV PATH="/opt/venv/bin:$PATH"

# Run as non-root
RUN useradd -m appuser
USER appuser

CMD ["python", "main.py"]
```

**Result**: Build stage ~8GB → Runtime stage ~4GB (50% reduction)

### Build Targets

You can build specific stages for different purposes:

``` dockerfile
FROM python:3.11-slim AS base
WORKDIR /app
COPY pyproject.toml .
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv sync
COPY . .

FROM base AS development
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
RUN uv pip install --system pytest pytest-cov black ruff
CMD ["python", "-m", "pytest"]

FROM base AS production
RUN useradd -m appuser
USER appuser
CMD ["python", "app.py"]
```

``` bash
# Build development image
docker build --target development -t myapp:dev .

# Build production image
docker build --target production -t myapp:prod .
```

------------------------------------------------------------------------

## Security Best Practices

### Non-root Users

Running containers as root is a security risk. If an attacker exploits a vulnerability, they gain root access.

**❌ Bad - Running as root:**

``` dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
CMD ["python", "app.py"]  # Runs as root!
```

**✅ Good - Non-root user:**

``` dockerfile
FROM python:3.11-slim

WORKDIR /app

# Create non-root user
RUN groupadd -r appgroup && \
    useradd -r -g appgroup -d /app -s /sbin/nologin appuser

# Copy files and set ownership
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

CMD ["python", "app.py"]
```

### Image Scanning

Scan images for known vulnerabilities before deployment.

``` bash
# Docker Scout (built into Docker Desktop)
docker scout cves myimage:latest

# Trivy (open source)
trivy image myimage:latest

# Snyk
snyk container test myimage:latest
```

### Secrets Management

**❌ Never do this - Secrets in Dockerfile:**

``` dockerfile
# NEVER store secrets in Dockerfile!
ENV DATABASE_PASSWORD=mysecretpassword
```

**✅ Use runtime injection:**

``` bash
# Pass at runtime via environment variables
docker run -e DATABASE_PASSWORD="${DB_PASS}" myimage

# Or use Docker secrets (Swarm) / Kubernetes secrets
docker secret create db_password password.txt
```

### Container Security Options

``` bash
# Prevent privilege escalation
docker run --security-opt=no-new-privileges:true myimage

# Combined security hardening (production-grade)
docker run \
  --read-only \
  --security-opt=no-new-privileges:true \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --tmpfs /tmp \
  --memory=512m \
  --cpus=1 \
  myimage:prod
```

Always start with `--cap-drop=ALL` and explicitly add back only the capabilities your application requires:

| Capability          | Use Case              |
|---------------------|-----------------------|
| `NET_BIND_SERVICE`  | Bind to ports \< 1024 |
| `CHOWN`             | Change file ownership |
| `SETUID` / `SETGID` | Change user/group IDs |

Most applications need **zero** additional capabilities.

------------------------------------------------------------------------

## Summary

**Key Takeaways:**

1.  **Docker Basics**

    - Containers share the host kernel, making them lightweight and fast

    - Images are immutable templates; containers are runnable instances

    - Use volumes for persistent data and port mapping for network access

2.  **Dockerfile Best Practices**

    - Order instructions to maximize layer caching

    - Use `.dockerignore` to reduce build context

    - Combine RUN commands to minimize layers

    - Choose appropriate base images (prefer `-slim` variants)

3.  **Multi-stage Builds**

    - Separate build and runtime environments

    - Dramatically reduce final image size

    - Use named stages for clarity and build targets

4.  **Security**

    - Always run as non-root user

    - Scan images for vulnerabilities

    - Never store secrets in images

    - Minimize installed packages

------------------------------------------------------------------------

## References

1.  [Docker Documentation](https://docs.docker.com/)

2.  [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

3.  [Multi-stage Builds](https://docs.docker.com/develop/develop-images/multistage-build/)

4.  [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)

5.  [Trivy Container Scanner](https://aquasecurity.github.io/trivy/)
