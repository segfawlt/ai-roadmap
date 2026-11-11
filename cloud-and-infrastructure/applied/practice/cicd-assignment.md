# CI/CD and Deployment Assignment

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Building a Complete CI/CD Pipeline with GitHub Actions |
| **Course** | Basic DevOps Essentials for Developer |
| **Project Name** | cicd-pipeline-demo |
| **Estimated Time** | 150 minutes |
| **Framework** | GitHub Actions, Python 3.11+, Docker |

------------------------------------------------------------------------

## Learning Objectives

After completing this assignment, you will be able to:

- **Create** CI/CD workflows using GitHub Actions

- **Configure** workflow triggers for different events (push, PR, manual)

- **Implement** automated testing and linting in pipelines

- **Build** and push Docker images as part of CI/CD

- **Apply** caching strategies to speed up pipelines

- **Design** deployment strategies (staging/production environments)

- **Utilize** secrets management for secure deployments

------------------------------------------------------------------------

## Prerequisites

- GitHub account with repository access

- Completed Docker assignment (or equivalent Docker knowledge)

- Basic understanding of YAML syntax

- Python application with tests

------------------------------------------------------------------------

## Tasks

### Task 1: Create Basic CI Workflow (20 points)

1.  **Create a GitHub repository** with a Python application including:

    - Source code in `src/` directory

    - Tests in `tests/` directory

    - `pyproject.toml` with dependencies

2.  **Create a basic CI workflow** (`.github/workflows/ci.yml`):

    ``` yaml
    name: CI Pipeline
    on:
      push:
        branches: [main, develop]
      pull_request:
        branches: [main]
    ```

3.  **Implement the following jobs**:

    - **lint**: Run Ruff and Black for code quality

    - **test**: Run pytest with coverage reporting

4.  **Configure job dependencies** so tests only run after linting passes

### Task 2: Implement Matrix Testing (15 points)

1.  **Extend the CI workflow** with matrix builds:

    - Test across Python versions: 3.10, 3.11, 3.12

    - Test on multiple OS: ubuntu-latest, macos-latest

2.  **Configure fail-fast behavior** appropriately

3.  **Document the matrix configuration** and explain when to use `exclude`

### Task 3: Add Caching and Artifacts (20 points)

1.  **Implement dependency caching**:

    - Cache pip dependencies

    - Use cache key based on `pyproject.toml` hash

2.  **Upload test artifacts**:

    - Coverage reports

    - Test results (JUnit XML format)

3.  **Configure Codecov integration** for coverage reporting

4.  **Measure and document** the time saved by caching:

    | Run Type      | Build Time |
    |---------------|------------|
    | Without cache | ? seconds  |
    | With cache    | ? seconds  |

### Task 4: Build and Push Docker Image (20 points)

1.  **Create a Docker build workflow** that:

    - Builds the Docker image on every push to main

    - Tags images with commit SHA and `latest`

    - Pushes to GitHub Container Registry (ghcr.io)

2.  **Configure secrets** for registry authentication

3.  **Implement conditional builds**:

    - Only build Docker image when source code changes

    - Use `paths` filter to skip builds for documentation changes

4.  **Add image scanning** in the pipeline using Trivy or Docker Scout

### Task 5: Implement Deployment Strategy (25 points)

1.  **Create environment-specific deployments**:

    - **staging**: Auto-deploy on push to `develop` branch

    - **production**: Manual approval required, deploy on push to `main`

2.  **Implement a deployment workflow** with:

    - Environment protection rules

    - Deployment status notifications

    - Health check verification after deployment

3.  **Create a simple deployment script** (`deploy.sh`) that:

    - Pulls the new Docker image

    - Performs health check

    - Reports deployment status

4.  **Document the deployment flow** with a diagram:

    ```
    Code Push → CI Tests → Build Image → Deploy Staging → Manual Approval → Deploy Production
    ```

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

GitHub repository URL with all workflows

`.github/workflows/ci.yml` - CI pipeline

`.github/workflows/docker.yml` - Docker build pipeline

`.github/workflows/deploy.yml` - Deployment pipeline

`README.md` with pipeline documentation

Screenshots of successful workflow runs

Screenshots of coverage reports and artifacts

### Submission Checklist

CI workflow runs on push and pull requests

Matrix testing across multiple Python versions

Caching implemented and working

Docker image builds and pushes successfully

Deployment workflow with environment protection

All workflows pass without errors

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                           | Points  |
|------------------------------------|---------|
| Basic CI workflow implementation   | 20      |
| Matrix testing configuration       | 15      |
| Caching and artifacts              | 20      |
| Docker build and push              | 20      |
| Deployment strategy implementation | 25      |
| **Total**                          | **100** |

------------------------------------------------------------------------

## Hints

- Use `workflow_dispatch` to enable manual triggering for testing

- Use `needs` keyword to define job dependencies

- Check workflow runs in the Actions tab of your repository

- Use `${{`` ``secrets.GITHUB_TOKEN`` ``}}` for GHCR authentication

- Test workflows on a branch before merging to main

- Use `if:`` ``github.ref`` ``==`` ``'refs/heads/main'` for conditional steps

------------------------------------------------------------------------

## References

1.  [GitHub Actions Documentation](https://docs.github.com/en/actions)

2.  [Workflow Syntax Reference](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)

3.  [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

4.  [Codecov GitHub Action](https://github.com/codecov/codecov-action)

5.  [Docker Build Push Action](https://github.com/docker/build-push-action)
