# CI/CD Patterns Quick Reference

## The canonical pipeline

Every modern CI pipeline should have these stages, in order:

1.  **Lint / format check** — cheap, catches ~20% of problems.

2.  **Type check** (if typed language) — catches another 30%.

3.  **Unit tests** — fast feedback, runs on every commit.

4.  **Integration tests** — slower, may need a database/network.

5.  **Build** — compile / bundle / package the artifact.

6.  **Container image build and push** (if deploying containers).

7.  **Security scan** — SAST, dependency audit, container scan.

8.  **Deploy to staging** — automatic on merge to main.

9.  **Smoke tests on staging** — prove the deploy actually works.

10. **Deploy to prod** — manual approval gate.

Stages 1–4 should complete in under 5 minutes. If they don’t, developers stop running them locally and start pushing broken code.

## GitLab CI example

``` yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - deploy

variables:
  PYTHON_VERSION: "3.11"

default:
  image: python:${PYTHON_VERSION}-slim
  cache:
    key:
      files:
        - pyproject.toml
        - uv.lock
    paths:
      - .uv-cache/
  before_script:
    - pip install uv
    - uv sync --frozen

lint:
  stage: lint
  script:
    - uv run ruff check .
    - uv run ruff format --check .

typecheck:
  stage: lint
  script:
    - uv run mypy src/

test:
  stage: test
  services:
    - postgres:16-alpine
  variables:
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://postgres:test@postgres:5432/test
  script:
    - uv run pytest --cov=src --cov-report=term --cov-report=xml
  coverage: '/^TOTAL.+?(\d+\%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest

deploy_staging:
  stage: deploy
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - ./scripts/deploy.sh staging $CI_COMMIT_SHA

deploy_prod:
  stage: deploy
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  environment:
    name: production
    url: https://example.com
  script:
    - ./scripts/deploy.sh prod $CI_COMMIT_SHA
```

## GitHub Actions equivalent

``` yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .

  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --frozen
      - run: uv run pytest --cov=src
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/postgres

  build-and-push:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Secrets management

**Never** commit secrets to Git. Options in order of preference:

1.  **Cloud-native identity** — GitHub Actions OIDC → AWS IAM assume role, no long-lived keys at all.

2.  **Secret manager integration** — AWS Secrets Manager, HashiCorp Vault, Infisical, injected at job start.

3.  **CI-native secrets** — GitLab CI variables, GitHub Actions secrets. Easy, still much better than Git.

4.  **Never** — environment variables committed to the repo.

## Caching tips

- Cache the **dependency lockfile’s output**, not its input.

- Use a content-addressed cache key (hash of `uv.lock`, `pnpm-lock.yaml`, etc.) so cache invalidates automatically on changes.

- Cache build tool output (`ruff`, `mypy`, `tsc` incremental files) for 2–5× speedup on incremental runs.

## Zero-downtime deploys

For containerized services, the deploy step should:

1.  Push the new image with a **unique tag** (commit SHA, never `latest`).

2.  Update the Kubernetes Deployment / ECS service / Cloud Run revision to point at the new tag.

3.  Use a rolling strategy with a readiness probe so old pods drain only when new ones pass `/ready`.

4.  Verify with a smoke test against the new version before marking success.

5.  Keep the previous version’s image available for fast rollback.

## Common mistakes

- **Slow pipelines** — anything over 15 minutes and devs stop waiting, start pushing broken code, and the signal value collapses.

- **Flaky tests** — one flaky test ruins the entire feedback loop. Quarantine or fix aggressively.

- **Everything in one job** — makes failures hard to diagnose. Split stages.

- **No branch protection** — merge buttons that don’t require CI to pass defeat the whole point.

- **Manual steps hidden in runbooks** — if it’s not in the pipeline, it will drift.

## Practice

### 1. Build the canonical pipeline

Take a small Python FastAPI service. Write a `.gitlab-ci.yml` (or `.github/workflows/ci.yml`) that implements the full canonical pipeline from the documentation page: lint → typecheck → test → build → deploy (to a fake staging target).

Target: total pipeline under 5 minutes on cache hit.

### 2. Matrix test against multiple Python versions

Run the test suite against Python 3.11, 3.12, and 3.13 in parallel. Use `parallel:`` ``matrix:` (GitLab) or `strategy.matrix` (GitHub). Verify the pipeline summary shows all three as independent jobs.

### 3. Cache hit rate

Run your pipeline twice. Measure total time and cache hit rate on the second run. If cache hit rate isn’t \>80%, your cache key is wrong — fix it.

### 4. Secret rotation

Add a secret (e.g., a fake API key) to the CI-native secret store. Use it in a job via an environment variable. Rotate it — confirm the next pipeline run uses the new value without any code change.

Bonus: migrate the same secret to a cloud secret manager and inject it via OIDC instead of a static CI variable.

### 5. Manual approval gate

Add a `deploy_prod` job that requires manual approval via GitLab `when:`` ``manual` (or GitHub Actions `environments:` with required reviewers). Confirm the pipeline pauses and waits for a human to click the button before running the deploy step.

### 6. Flake detection

Deliberately introduce a flaky test (random `assert`` ``random.random()`` ``>`` ``0.3`). Run the pipeline 10 times. Configure the pipeline to retry failing tests once and report the flake. Then fix or quarantine it.

## Review Questions

1.  **What is the target wall-clock time for the lint-through-integration-test phase of a CI pipeline?**

    - A. Under 60 minutes

    - B. Under 15 minutes; ideally under 5

    - C. Under 2 hours

    - D. There is no target

2.  **Why is `latest` a dangerous tag for container images in a deploy pipeline?**

    - A. It’s slower to pull

    - B. It’s mutable — you cannot reliably roll back or identify what is running in production

    - C. It’s banned by Docker Hub

    - D. It uses more disk space

3.  **What is the most secure way to give a GitHub Actions workflow access to AWS?**

    - A. Long-lived access keys stored as GitHub Secrets

    - B. OIDC federation with an AWS IAM role and an assume-role trust policy (no long-lived keys)

    - C. Committing keys to a private repo

    - D. Sharing the root account password

4.  **Which stage of a canonical CI pipeline should run first?**

    - A. Integration tests

    - B. Lint and format check (cheap, catches ~20% of problems)

    - C. Build and push Docker image

    - D. Deploy to production

5.  **What makes a cache key effective in CI?**

    - A. Using a fixed string like `"cache"`

    - B. Hashing the dependency lockfile (e.g., `uv.lock`, `pnpm-lock.yaml`) so the cache invalidates automatically on changes

    - C. Using the current timestamp

    - D. Not caching at all

6.  **A flaky test is in your pipeline. What should you do?**

    - A. Ignore it and re-run the pipeline until it passes

    - B. Quarantine (skip) or fix it — one flake destroys the signal value of the whole pipeline

    - C. Delete the test

    - D. Mark the whole suite as optional

7.  **How should the deploy step tag a container image?**

    - A. With `latest`

    - B. With an immutable unique tag like the commit SHA

    - C. With a random UUID generated at deploy time

    - D. It shouldn’t tag at all

8.  **Why should the production deploy require a manual approval gate?**

    - A. To give someone credit

    - B. To add a human checkpoint for high-blast-radius changes, even after automated tests pass

    - C. It’s required by law

    - D. It’s free extra compute time

9.  **What does “zero-downtime deploy” typically require?**

    - A. Taking the service offline during deploys

    - B. A rolling update strategy with health/readiness probes, so old instances drain only when new ones are healthy

    - C. Recompiling the kernel

    - D. Running two separate clusters

10. **Why should manual steps be avoided in the deploy pipeline?**

    - A. Manual steps are slower

    - B. They drift from documentation, are unauditable, and can’t be reproduced — anything not in the pipeline eventually breaks

    - C. They use more electricity

    - D. Manual steps are illegal

**View Answer Key**

1.  **B** — Under 15 minutes; ideally under 5 for fast feedback.

2.  **B** — Mutable tags break rollback and auditability.

3.  **B** — OIDC federation is the modern, keyless approach.

4.  **B** — Cheap checks first; they catch a large fraction of problems at minimal cost.

5.  **B** — Content-addressed keys (hashing lockfiles) give automatic invalidation.

6.  **B** — Quarantine or fix; never just re-run.

7.  **B** — Immutable, unique tags (commit SHA) for rollback and traceability.

8.  **B** — Manual approval is a human checkpoint for high-blast-radius changes.

9.  **B** — Rolling updates with readiness probes are the standard zero-downtime pattern.

10. **B** — Manual steps drift and become un-reproducible; automate everything.
