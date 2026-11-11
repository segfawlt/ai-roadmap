# Containerization with Docker Assignment

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Containerizing a Python Application with Best Practices |
| **Course** | Basic DevOps Essentials for Developer |
| **Project Name** | docker-python-app |
| **Estimated Time** | 120 minutes |
| **Framework** | Docker 24+, Python 3.11+ |

------------------------------------------------------------------------

## Learning Objectives

After completing this assignment, you will be able to:

- **Create** optimized Dockerfiles for Python applications

- **Implement** multi-stage builds to reduce image size

- **Configure** proper layer caching for faster builds

- **Apply** security best practices (non-root users, secrets management)

- **Utilize** .dockerignore to exclude unnecessary files

- **Verify** image security using vulnerability scanning tools

------------------------------------------------------------------------

## Prerequisites

- Docker installed locally (Docker Desktop or Docker Engine)

- Basic understanding of Python application structure

- Familiarity with command line operations

------------------------------------------------------------------------

## Tasks

### Task 1: Create a Basic Dockerfile (20 points)

1.  **Create a simple Python web application** (FastAPI or Flask):

    - Create `app.py` with a basic health endpoint (`/health`)

    - Create `pyproject.toml` or `requirements.txt` with dependencies

2.  **Write an initial Dockerfile** that:

    - Uses `python:3.11-slim` as base image

    - Sets appropriate environment variables (`PYTHONDONTWRITEBYTECODE`, `PYTHONUNBUFFERED`)

    - Copies and installs dependencies

    - Exposes the application port

    - Sets the default command to run the application

3.  **Build and test** the image:

    ``` bash
    docker build -t myapp:v1 .
    docker run -d -p 8000:8000 myapp:v1
    ```

### Task 2: Optimize Layer Caching (20 points)

1.  **Analyze the current Dockerfile** for caching issues

2.  **Refactor to optimize caching**:

    - Copy dependency files before application code

    - Combine RUN commands to reduce layers

    - Order instructions from least to most frequently changed

3.  **Document the changes** and explain why each optimization improves build time

### Task 3: Implement Multi-stage Build (25 points)

1.  **Create a multi-stage Dockerfile** with:

    - **Builder stage**: Install build dependencies and compile/install packages

    - **Runtime stage**: Copy only necessary artifacts from builder

2.  **Compare image sizes**:

    | Stage        | Image Size |
    |--------------|------------|
    | Single-stage | ? MB       |
    | Multi-stage  | ? MB       |

3.  **Implement build targets** for different environments:

    - `development`: Include dev tools (pytest, ruff)

    - `production`: Minimal runtime only

4.  **Build specific targets**:

    ``` bash
    docker build --target development -t myapp:dev .
    docker build --target production -t myapp:prod .
    ```

### Task 4: Implement Security Best Practices (25 points)

1.  **Create a non-root user** and run the container as that user

2.  **Create a `.dockerignore` file** excluding:

    - `.git`, `__pycache__`, `.venv`

    - Test files and documentation

    - Environment files (`.env`)

3.  **Add a HEALTHCHECK instruction** to monitor application health

4.  **Scan the image for vulnerabilities** using Docker Scout or Trivy:

    ``` bash
    docker scout cves myapp:prod
    # or
    trivy image myapp:prod
    ```

5.  **Document any vulnerabilities found** and mitigation strategies

### Task 5: Volume and Port Configuration (10 points)

1.  **Configure volumes** for:

    - Persistent data storage

    - Configuration files (read-only)

2.  **Document port mapping** requirements

3.  **Create a `docker-compose.yml`** for local development that includes:

    - Volume mounts for hot-reload

    - Environment variable configuration

    - Health check configuration

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code for Python application

Optimized multi-stage `Dockerfile`

`.dockerignore` file

`docker-compose.yml` for development

`README.md` with build and run instructions

Screenshots of running container and vulnerability scan results

### Submission Checklist

Docker image builds without errors

Application runs correctly in container

Multi-stage build implemented with measurable size reduction

Non-root user configured

Health check working

Vulnerability scan completed

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                         | Points  |
|----------------------------------|---------|
| Basic Dockerfile creation        | 20      |
| Layer caching optimization       | 20      |
| Multi-stage build implementation | 25      |
| Security best practices          | 25      |
| Volume and port configuration    | 10      |
| **Total**                        | **100** |

------------------------------------------------------------------------

## Hints

- Use `docker`` ``history`` ``<image>` to inspect layer sizes

- Use `docker`` ``images` to compare image sizes before and after optimization

- Start with the official Python slim images for smaller base size

- Always specify exact version tags for reproducibility

- Test your health check with `docker`` ``inspect`` ``--format='{{.State.Health.Status}}'`` ``<container>`

------------------------------------------------------------------------

## References

1.  [Docker Documentation](https://docs.docker.com/)

2.  [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

3.  [Multi-stage Builds](https://docs.docker.com/develop/develop-images/multistage-build/)

4.  [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
