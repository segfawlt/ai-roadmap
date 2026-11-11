# Orchestration Overview

Everyone reads this page — only the group assigned orchestration (see [Deep Learning & MLOps Training Overview](../overview.md)) goes deep on it, using a tool of their choice from [MLOps Tool Landscape](tool-landscape.md).

## The Problem a Shell Script Eventually Hits

A training pipeline is a sequence of steps with dependencies: validate data, then train, then evaluate, then register the model, then (if it passes) promote it. A shell script that runs these in order works fine — until a step fails at 2am, or two steps could run in parallel and don’t, or someone needs to know which step of last Tuesday’s run actually failed.

**Orchestration** means expressing that pipeline as an explicit graph of tasks and dependencies, rather than an implicit sequence buried in script logic:

``` mermaid
        graph LR
    A[validate_data] --> B[train]
    B --> C[evaluate]
    C --> D[register]
    D --> E[promote]
```

An orchestrator gives you, on top of that graph: scheduling (run this every night), retries (a flaky step gets retried automatically instead of failing the whole run), visibility (which task failed, and its logs, without grepping through one giant script’s output), and dependency correctness (a task cannot start before the ones it depends on finish).

## When It’s Worth Adopting

If your pipeline is one person running one script by hand once a week, you do not need an orchestrator — you need a cron job at most. Orchestration earns its complexity when a pipeline has real dependencies between steps, needs to run on a schedule or in response to new data, or is owned by more than one person who all need visibility into what ran and what failed.

## Explore Further

- **Core vocabulary** — `DAG`` ``(Directed`` ``Acyclic`` ``Graph)`, `task`` ``dependency`, `retry`` ``policy`, `sensor` (a task that waits for a condition, like a new file arriving), `schedule_interval`

- **What this overview does not cover** — running an orchestrator on managed infrastructure (a specific vendor’s hosted service) is a hosting decision, not an orchestration concept; see [Basic AWS Services Essential](../../cloud-and-infrastructure/applied/aws-services.md) if that becomes relevant to your project

Curated starting points:

- [Apache Airflow “Concepts” documentation](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html) — the DAG vocabulary above, explained by the tool most teams reach for first

- [Prefect vs. Airflow comparison](https://docs.prefect.io/latest/getting-started/why-prefect/) — useful if your group is deciding rather than defaulting

## For Whichever Group Takes This Area

Your deep-dive brief is at [Tool Deep Dive](practice/tool-deep-dive.md). It applies regardless of which of the three MLOps areas — orchestration, dataset versioning, or drift monitoring — your group is assigned.
