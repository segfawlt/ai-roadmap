# CI/CD Automation Pipelines

## Introduction

Continuous Integration and Continuous Deployment (CI/CD) are foundational practices in modern software development. They enable teams to deliver code changes more frequently and reliably by automating the build, test, and deployment processes. For AI/ML applications and cloud-native development, robust CI/CD pipelines are essential for maintaining code quality and enabling rapid iteration.

**Why CI/CD Matters for AI/RAG Projects:**

- **Rapid Experimentation**: Quickly test new models and features in production-like environments

- **Quality Assurance**: Automated tests catch regressions before they reach production

- **Reproducibility**: Version-controlled pipelines ensure consistent deployments

- **Collaboration**: Team members can merge changes confidently with automated checks

- **Model Validation**: Catch accuracy regressions before they impact users

------------------------------------------------------------------------

## CI/CD Fundamentals

### Continuous Integration (CI)

Continuous Integration is the practice of frequently merging code changes into a shared repository, where automated builds and tests verify each integration.

**Key Principles:**

- Developers commit code at least daily

- Each commit triggers an automated build

- Automated tests run on every build

- Failed builds are fixed immediately

**CI Pipeline Stages:**

``` mermaid
        flowchart LR
    A["Lint<br>(Code Quality)"] --> B["Build<br>(Compile / Package)"]
    B --> C["Test<br>(Unit / Integration)"]
    C --> D["Report<br>(Coverage / Results)"]
```

### Continuous Delivery vs Continuous Deployment

| Aspect      | Continuous Delivery        | Continuous Deployment              |
|-------------|----------------------------|------------------------------------|
| Definition  | Code is always deployable  | Every change deploys automatically |
| Manual Step | Approval before production | No manual intervention             |
| Risk        | Lower (human verification) | Requires robust testing            |
| Speed       | Fast with gates            | Fastest possible                   |

------------------------------------------------------------------------

## GitHub Actions

GitHub Actions is a powerful CI/CD platform integrated directly into GitHub. It allows you to automate workflows triggered by repository events.

### Core Concepts

**Workflow**: An automated process defined in a YAML file in `.github/workflows/`

**Job**: A set of steps that execute on the same runner

**Step**: An individual task that can run commands or actions

**Action**: A reusable unit of code (from GitHub Marketplace or custom)

**Runner**: A server that executes your workflows (GitHub-hosted or self-hosted)

### Basic Workflow Structure

``` yaml
# .github/workflows/ci.yml
name: CI Pipeline

# Trigger conditions
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# Environment variables (available to all jobs)
env:
  PYTHON_VERSION: "3.11"

jobs:
  # First job: Lint and format check
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install linters
        run: uv pip install --system ruff black mypy

      - name: Run Ruff (linter)
        run: ruff check .

      - name: Check formatting with Black
        run: black --check .

  # Second job: Run tests
  test:
    runs-on: ubuntu-latest
    needs: lint # Wait for lint to pass

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: |
          uv sync
          uv pip install --system pytest pytest-cov

      - name: Run tests with coverage
        run: pytest --cov=src --cov-report=xml

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          file: coverage.xml
```

### Workflow Triggers

``` yaml
on:
  # Push to specific branches
  push:
    branches: [main, develop]
    paths:
      - "src/**"
      - "tests/**"
    paths-ignore:
      - "**.md"
      - "docs/**"

  # Pull request events
  pull_request:
    types: [opened, synchronize, reopened]

  # Scheduled runs (cron syntax)
  schedule:
    - cron: "0 0 * * *" # Daily at midnight

  # Manual trigger
  workflow_dispatch:
    inputs:
      environment:
        description: "Deployment environment"
        required: true
        default: "staging"
        type: choice
        options:
          - staging
          - production
```

### Matrix Builds

Test across multiple configurations simultaneously:

``` yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        python-version: ["3.10", "3.11", "3.12"]
        exclude:
          - os: windows-latest
            python-version: "3.10"
      fail-fast: false # Continue other jobs if one fails

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pytest
```

### Secrets and Environment Variables

``` yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production # Use environment-specific secrets

    steps:
      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          echo "Deploying with secure credentials"
          ./deploy.sh
```

### Caching Dependencies

``` yaml
steps:
  - uses: actions/checkout@v4

  - name: Set up Python
    uses: actions/setup-python@v5
    with:
      python-version: "3.11"
      cache: "pip" # Built-in pip caching

  # Or custom caching
  - name: Cache dependencies
    uses: actions/cache@v4
    with:
      path: ~/.cache/pip
      key: ${{ runner.os }}-pip-${{ hashFiles('**/pyproject.toml') }}
      restore-keys: |
        ${{ runner.os }}-pip-
```

------------------------------------------------------------------------

## GitLab CI/CD

GitLab CI/CD is another powerful platform, especially popular in enterprise environments.

### Basic Pipeline Structure

``` yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  PYTHON_VERSION: "3.11"
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

# Cache configuration
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .cache/pip
    - venv/

# Lint job
lint:
  stage: lint
  image: python:${PYTHON_VERSION}-slim
  script:
    - uv pip install --system ruff black
    - ruff check .
    - black --check .

# Test job
test:
  stage: test
  image: python:${PYTHON_VERSION}-slim
  services:
    - postgres:16-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://test:test@postgres:5432/testdb
  script:
    - uv sync
    - pytest --cov=src --junitxml=report.xml
  coverage: '/TOTAL.*\s+(\d+%)/'
  artifacts:
    reports:
      junit: report.xml

# Build Docker image
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

# Deploy to staging
deploy_staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

# Deploy to production (manual)
deploy_production:
  stage: deploy
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
```

### GitLab vs GitHub Actions Comparison

| Feature     | GitHub Actions               | GitLab CI                    |
|-------------|------------------------------|------------------------------|
| Config File | `.github/workflows/*.yml`    | `.gitlab-ci.yml`             |
| Runners     | GitHub-hosted or self-hosted | GitLab-hosted or self-hosted |
| Marketplace | GitHub Marketplace           | GitLab CI templates          |
| Services    | Docker Compose in workflow   | Built-in `services` keyword  |
| Artifacts   | `actions/upload-artifact`    | Native `artifacts`           |
| Variables   | `env` and `secrets`          | `variables`                  |

------------------------------------------------------------------------

## Automated Testing in Pipelines

### Test Pyramid

``` mermaid
        block-beta
    columns 1
    block:e2e:1
        E["E2E Tests — Slow, Expensive (10%)"]
    end
    block:integ:1
        I["Integration Tests — Medium Speed (20%)"]
    end
    block:unit:1
        U["Unit Tests — Fast, Cheap (70%)"]
    end
```

### Complete Test Pipeline

``` yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - name: Install dependencies
        run: uv sync

      - name: Run unit tests
        run: pytest tests/unit -v --cov=src --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - name: Install dependencies
        run: uv sync

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/postgres
          REDIS_URL: redis://localhost:6379/0
        run: pytest tests/integration -v
```

------------------------------------------------------------------------

## Deployment Strategies

### Blue-Green Deployment

Zero-downtime deployment with instant rollback capability:

``` yaml
name: Blue-Green Deploy

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to inactive environment
        run: |
          # Determine current active environment
          CURRENT=$(curl -s $API_URL/health | jq -r '.environment')
          if [ "$CURRENT" == "blue" ]; then
            TARGET="green"
          else
            TARGET="blue"
          fi
          echo "Deploying to $TARGET environment"
          ./deploy.sh $TARGET ${{ github.sha }}

      - name: Health check
        run: |
          for i in {1..30}; do
            # Comprehensive health check - verify app AND dependencies
            HEALTH=$(curl -sf "$TARGET_URL/health" | jq -e '
              .status == "healthy" and
              .database == "connected" and
              .cache == "connected"')
            if [ "$HEALTH" == "true" ]; then
              echo "Health check passed - all dependencies verified"
              exit 0
            fi
            echo "Waiting for healthy status... (attempt $i/30)"
            sleep 10
          done
          echo "Health check failed after 30 attempts"
          exit 1

      - name: Switch traffic
        run: ./switch-traffic.sh $TARGET
```

### GitOps Deployment

GitOps uses Git as the single source of truth for infrastructure and application state. Tools like **ArgoCD** or **Flux** continuously reconcile the cluster state with the desired state in Git.

``` mermaid
        flowchart LR
    Dev["Developer<br>Push Code"] --> Git["Git Repo<br>(GitOps)"]
    Git <-- Sync --> Argo["ArgoCD"]
    Argo --> K8s["Kubernetes<br>Cluster"]
```

**GitOps Workflow:**

``` yaml
# .github/workflows/gitops.yml
name: GitOps Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  update-manifests:
    runs-on: ubuntu-latest
    needs: build-and-push

    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: my-org/gitops-manifests
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Update image tag
        run: |
          # Update Kubernetes manifest with new image
          yq eval '.spec.template.spec.containers[0].image = "ghcr.io/${{ github.repository }}:${{ github.sha }}"' \
            -i apps/myapp/deployment.yaml

      - name: Commit and push
        run: |
          git config user.name "github-actions"
          git config user.email "[email protected]"
          git add .
          git commit -m "Deploy: ${{ github.sha }}"
          git push
      # ArgoCD automatically syncs from this repo
```

Why GitOps?

- **Audit Trail**: Every change is a Git commit

- **Rollback**: `git`` ``revert` instantly rolls back deployments

- **Security**: No direct cluster access from CI

- **Consistency**: Git is the single source of truth

### Canary Deployment

Gradual rollout with monitoring:

``` yaml
name: Canary Deploy

jobs:
  deploy-canary:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy canary (10% traffic)
        run: |
          kubectl set image deployment/app app=$IMAGE:${{ github.sha }}
          kubectl patch deployment app -p '{"spec":{"replicas":1}}'

      - name: Monitor metrics (5 minutes)
        run: |
          for i in {1..10}; do
            ERROR_RATE=$(curl -s $METRICS_URL | jq '.error_rate')
            if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
              echo "Error rate too high: $ERROR_RATE"
              exit 1
            fi
            sleep 30
          done

      - name: Promote to full deployment
        run: kubectl scale deployment app --replicas=5

  rollback:
    runs-on: ubuntu-latest
    needs: deploy-canary
    if: failure()

    steps:
      - name: Rollback canary
        run: kubectl rollout undo deployment/app
```

------------------------------------------------------------------------

## Best Practices

### Pipeline Design Principles

1.  **Fail Fast**: Run quick checks (lint, format) first

2.  **Parallel Execution**: Run independent jobs concurrently

3.  **Cache Dependencies**: Speed up builds with caching

4.  **Artifacts**: Share data between jobs efficiently

5.  **Environment Separation**: Use dedicated environments for staging/production

6.  **Secret Management**: Never hardcode secrets; use encrypted secrets

7.  **Idempotency**: Pipelines should be safely re-runnable

### Security Best Practices

``` yaml
jobs:
  secure-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read # Minimal permissions
      packages: write

    steps:
      # Pin action versions with SHA
      - uses: actions/checkout@8ade135a41bc03ea155e62e844d188df1ea18608 # v4.1.1

      # Use OIDC for cloud authentication (no long-lived secrets)
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: us-east-1
```

### Supply Chain Security

Protect your software supply chain with automated scanning and dependency updates:

``` yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "0 6 * * 1" # Weekly on Monday

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          scan-ref: "."
          format: "sarif"
          output: "trivy-results.sarif"
          severity: "CRITICAL,HIGH"

      - name: Upload to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: "trivy-results.sarif"

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan container image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "myapp:${{ github.sha }}"
          format: "table"
          exit-code: "1" # Fail if vulnerabilities found
          severity: "CRITICAL"
```

**Automated Dependency Updates:**

``` yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      python-deps:
        patterns:
          - "*"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

------------------------------------------------------------------------

## CI/CD for AI/ML Projects

AI/ML pipelines have unique requirements beyond traditional software CI/CD.

> **Note**
>
> Note
>
> For the ML-specific decisions behind *when* to retrain and *what* gates a training pipeline needs (as opposed to the CI/CD mechanics below), see [MLOps Principles](../../deep-learning-and-mlops/mlops/principles.md).

### Model Validation Pipeline

``` yaml
# .github/workflows/ml-pipeline.yml
name: ML Pipeline

on:
  push:
    paths:
      - "models/**"
      - "src/**"
      - "tests/**"

jobs:
  data-validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install great-expectations pandas

      - name: Validate data quality
        run: |
          python -c "
          import great_expectations as gx
          context = gx.get_context()
          result = context.run_checkpoint('data_quality_checkpoint')
          assert result.success, 'Data validation failed'
          "

  model-test:
    runs-on: ubuntu-latest
    needs: data-validation

    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true # Pull model artifacts

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install pytest torch transformers

      - name: Run inference tests
        run: |
          pytest tests/model/ -v --tb=short

      - name: Check model performance
        run: |
          python scripts/benchmark.py \
            --model models/latest \
            --threshold 0.85 \
            --metric accuracy

  log-experiment:
    runs-on: ubuntu-latest
    needs: model-test
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Log to experiment tracker
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_URI }}
        run: |
          python scripts/log_experiment.py \
            --commit ${{ github.sha }} \
            --model-path models/latest
```

### Caching Large Model Artifacts

``` yaml
- name: Cache model weights
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/huggingface
      models/
    key: models-${{ hashFiles('models/config.json') }}
    restore-keys: |
      models-

- name: Download model if not cached
  run: |
    if [ ! -f models/pytorch_model.bin ]; then
      python scripts/download_model.py
    fi
```

------------------------------------------------------------------------

## Summary

**Key Takeaways:**

1.  **CI/CD Fundamentals**

    - CI: Automated build and test on every commit

    - CD: Automated deployment to staging/production

    - Fast feedback loop improves code quality

2.  **GitHub Actions**

    - Workflows triggered by repository events

    - Matrix builds for multi-platform testing

    - Reusable workflows for consistency

    - Built-in caching and artifact management

3.  **Testing in Pipelines**

    - Follow the test pyramid (unit → integration → E2E)

    - Use services for database/cache in tests

    - Include security scanning (Trivy, Bandit)

4.  **Deployment Strategies**

    - Blue-Green: Zero-downtime with instant rollback

    - Canary: Gradual rollout with monitoring

    - Environment promotion: dev → staging → production

------------------------------------------------------------------------

## References

1.  [GitHub Actions Documentation](https://docs.github.com/en/actions)

2.  [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)

3.  [Docker Build Push Action](https://github.com/docker/build-push-action)

4.  [Codecov GitHub Action](https://github.com/codecov/codecov-action)

5.  [Continuous Delivery - Martin Fowler](https://martinfowler.com/bliki/ContinuousDelivery.html)

6.  [DORA DevOps Capabilities](https://dora.dev/capabilities/)

7.  [ArgoCD - GitOps for Kubernetes](https://argo-cd.readthedocs.io/)

8.  [Trivy - Container Security Scanner](https://aquasecurity.github.io/trivy/)

9.  [Great Expectations - Data Quality](https://greatexpectations.io/)

10. [MLflow - Experiment Tracking](https://mlflow.org/)
