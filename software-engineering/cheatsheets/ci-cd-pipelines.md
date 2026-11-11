# CI/CD Pipelines

## Introduction

Continuous Integration (CI) and Continuous Deployment (CD) automate the process of building, testing, and deploying code. A well-designed pipeline catches bugs early, enforces quality standards, and delivers changes to production safely and quickly.

**Why CI/CD Matters:**

- **Catch bugs early**: Automated tests run on every commit, not just before release

- **Consistent builds**: Every deployment follows the same steps, eliminating “works on my machine” issues

- **Faster feedback**: Developers know within minutes if their change breaks something

- **Safe deployments**: Automated checks prevent broken code from reaching production

- **Team velocity**: Less time on manual testing and deployment = more time building features

------------------------------------------------------------------------

## Core Concepts

### CI vs CD vs CD

| Term | Full Name | What It Does |
|----|----|----|
| **CI** | Continuous Integration | Automatically build and test code on every commit/merge |
| **CD** | Continuous Delivery | Automatically prepare releases that *can* be deployed (manual trigger) |
| **CD** | Continuous Deployment | Automatically deploy every change that passes all checks (no manual step) |

``` mermaid
        flowchart LR
    A[Developer pushes code] --> B[CI: Build & Test]
    B -->|Tests pass| C[CD: Stage & Review]
    C -->|Approved| D[CD: Deploy to Production]
    B -->|Tests fail| E[Notify developer]
```

### Pipeline Stages

A typical CI/CD pipeline has these stages, executed in order:

``` yaml
stages:
  - lint # Code quality checks (fast, fail early)
  - test # Unit and integration tests
  - build # Compile/package the application
  - security # Vulnerability scanning
  - deploy # Ship to staging/production
```

------------------------------------------------------------------------

## GitLab CI Fundamentals

GitLab CI uses a `.gitlab-ci.yml` file at the repository root to define pipelines.

### Basic Pipeline

``` yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

# Lint stage: Check code quality
lint:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install ruff
    - ruff check src/
    - ruff format --check src/
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# Test stage: Run automated tests
test:
  stage: test
  image: python:3.12-slim
  script:
    - pip install -r requirements.txt
    - pytest tests/ --cov=src --cov-report=term-missing --cov-fail-under=80
  coverage: '/TOTAL.*\s+(\d+%)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

# Build stage: Build the artifact
build:
  stage: build
  image: python:3.12-slim
  script:
    - pip install build
    - python -m build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week

# Deploy stage: Ship to production
deploy-production:
  stage: deploy
  script:
    - echo "Deploying to production..."
  environment:
    name: production
    url: https://yourapp.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  when: manual # Require manual approval for production
```

### Key GitLab CI Concepts

| Concept | Purpose | Example |
|----|----|----|
| **`stages`** | Define the order of pipeline phases | `[lint,`` ``test,`` ``build,`` ``deploy]` |
| **`image`** | Docker image for the job runner | `python:3.12-slim` |
| **`script`** | Commands to execute | `pytest`` ``tests/` |
| **`artifacts`** | Files to pass between stages or download | Test reports, build output |
| **`cache`** | Persist files between pipeline runs (speed up) | `pip` cache, `node_modules` |
| **`rules`** | Control when jobs run | Only on MR, only on main branch |
| **`environment`** | Track deployments | `production`, `staging`, `preview/*` |
| **`needs`** | Run jobs out of stage order (DAG) | `needs:`` ``[build]` |

------------------------------------------------------------------------

## Caching Strategies

Caching is the single biggest lever for pipeline speed. Without caching, every job re-downloads dependencies from scratch.

### Python Project Caching

``` yaml
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

default:
  cache:
    key:
      files:
        - requirements.txt # Cache key changes when deps change
    paths:
      - .cache/pip
      - .venv/
```

### Node.js Project Caching

``` yaml
default:
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
```

### Cache Best Practices

| Practice | Why |
|----|----|
| Use file-based cache keys (`requirements.txt`, `package-lock.json`) | Cache invalidates automatically when dependencies change |
| Cache only dependency directories, not build output | Build output should be artifacts, not cache |
| Use `policy:`` ``pull` on jobs that only read the cache | Prevents race conditions on parallel jobs |
| Set `fallback_keys` for branch caches | Fall back to main branch cache on new branches |

------------------------------------------------------------------------

## Pipeline Design Patterns

### Pattern 1: Rules-Based Triggering

Only run relevant jobs based on what changed:

``` yaml
.content-paths: &content-paths
  - src/**/*
  - requirements.txt

lint:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes: *content-paths
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# Skip heavy jobs for docs-only changes
test:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - src/**/*.py
        - tests/**/*.py
        - requirements.txt
```

### Pattern 2: MR Preview Environments

Deploy merge request branches to temporary preview URLs:

``` yaml
deploy-preview:
  stage: deploy
  image: node:22-bookworm-slim
  needs:
    - job: build
      artifacts: true
  script:
    - npx wrangler@4 pages deploy ${BUILD_DIR}
      --project-name=${CLOUDFLARE_PROJECT_NAME}
      --commit-hash=${CI_COMMIT_SHA}
      --branch=${CI_COMMIT_REF_NAME}
  environment:
    name: preview/$CI_COMMIT_REF_SLUG
    url: https://${CI_COMMIT_REF_SLUG}.${CLOUDFLARE_PROJECT_NAME}.pages.dev
    auto_stop_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

### Pattern 3: Parallel Test Splitting

Split a large test suite across multiple runners:

``` yaml
test:
  stage: test
  parallel: 4
  script:
    - pytest tests/ --splits 4 --group $CI_NODE_INDEX
```

### Pattern 4: Security Scanning

``` yaml
security-scan:
  stage: security
  image: python:3.12-slim
  script:
    - pip install pip-audit bandit
    - pip-audit # Check dependencies for CVEs
    - bandit -r src/ -ll # Static security analysis
  allow_failure: true # Don't block pipeline initially
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

------------------------------------------------------------------------

## DORA Metrics

DORA (DevOps Research and Assessment) metrics measure team delivery performance. Track these to improve your pipeline:

| Metric | Definition | Elite Target | How to Improve |
|----|----|----|----|
| **Deployment Frequency** | How often you deploy to production | Multiple times per day | Automate deployment, reduce batch size |
| **Lead Time for Changes** | Time from commit to production | Less than 1 hour | Reduce pipeline duration, automate approvals |
| **Change Failure Rate** | % of deployments causing incidents | \< 5% | Better testing, gradual rollouts |
| **Time to Restore Service** | Time to recover from a failure | Less than 1 hour | Automated rollbacks, monitoring alerts |

------------------------------------------------------------------------

## Pipeline Speed Targets

| Pipeline Stage     | Target Duration   | If Exceeding                        |
|--------------------|-------------------|-------------------------------------|
| Lint               | \< 1 minute       | Use faster tools (ruff \> pylint)   |
| Unit tests         | \< 3 minutes      | Parallelize, remove slow tests      |
| Build              | \< 2 minutes      | Multi-stage Docker, better caching  |
| Security scan      | \< 2 minutes      | Run async, allow failure initially  |
| **Total pipeline** | **\< 10 minutes** | Split test suites, optimize caching |

If your pipeline takes more than 10 minutes, developers will stop waiting for it and push without checking results. Keep it fast.

------------------------------------------------------------------------

## Summary

| Topic | Key Takeaway |
|----|----|
| **CI** | Automate lint + test on every push. Fail fast, fail early. |
| **CD** | Automate deployment with manual approval for production. |
| **Caching** | Use file-based cache keys. Cache dependencies, not build output. |
| **Rules** | Only run jobs affected by the changes. Use `rules:changes`. |
| **Speed** | Target \< 10 minutes total. Parallelize tests. |
| **DORA** | Measure deployment frequency, lead time, failure rate, recovery time. |

------------------------------------------------------------------------

## References

1.  [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)

2.  [GitLab CI/CD Best Practices](https://docs.gitlab.com/ee/ci/pipelines/pipeline_efficiency.html)

3.  [DORA Metrics — Google Cloud](https://dora.dev/)

4.  [Conventional Commits](https://www.conventionalcommits.org/)

5.  [P99Soft — CI/CD Best Practices 2026](https://p99soft.com)

## Practice

### Exercise 1: Build a Basic Pipeline

Create a `.gitlab-ci.yml` for a Python FastAPI project with these stages:

1.  **lint**: Run `ruff`` ``check` and `ruff`` ``format`` ``--check`

2.  **test**: Run `pytest` with 80% coverage requirement

3.  **build**: Build a Docker image

4.  **deploy**: Deploy to a staging environment (echo the commands)

**Requirements:**

- Use caching for pip dependencies

- Only run on merge requests and the main branch

- Test job should produce a coverage report artifact

### Exercise 2: Optimize a Slow Pipeline

Given this pipeline that takes 25 minutes:

``` yaml
stages: [test, build, deploy]

test:
  script:
    - pip install -r requirements.txt
    - pytest tests/ -v
  # No caching, no parallelization

build:
  script:
    - docker build -t myapp .
  # Builds from scratch every time

deploy:
  script:
    - docker push myapp
  when: manual
```

**Tasks:**

1.  Add caching to reduce dependency installation time

2.  Parallelize the test suite across 3 runners

3.  Add `rules:changes` to skip the build when only docs change

4.  Add a lint stage that runs before tests

5.  Target: Get the pipeline under 10 minutes

## Review Questions

1.  **What is the difference between Continuous Delivery and Continuous Deployment?**

    - *Hint*: One requires a manual step, the other does not.

2.  **Why should the lint stage come before the test stage in a pipeline?**

    - *Hint*: Think about fail-fast principles and how long each stage takes.

3.  **Explain how `rules:changes` improves pipeline efficiency. Give an example.**

    - *Hint*: Not every file change requires every job to run.

4.  **What are the four DORA metrics? Why are they important for measuring team performance?**

    - *Hint*: They measure speed AND stability, not just one or the other.

5.  **A developer complains their pipeline takes 20 minutes. What are three concrete steps you would take to reduce it?**

    - *Hint*: Caching, parallelization, and selective job triggering.

6.  **What is the difference between `cache` and `artifacts` in GitLab CI?**

    - *Hint*: One persists between pipeline runs (speed), the other passes data between jobs/stages within a single run.
