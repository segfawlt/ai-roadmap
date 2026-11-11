# MLOps Tool Landscape

Three MLOps areas beyond the shared MLflow baseline: orchestration, dataset versioning, and drift monitoring. **No tool is mandated for any of the three.** Each group is assigned one area (see [Deep Learning & MLOps Training Overview](../overview.md)) and chooses its own tool from the relevant list below — popularity is one input to that choice, not the deciding one. Justify your pick at the final demo; “it was the first result on Google” is not a justification, “it doesn’t need a server and our project is small” is.

## Orchestration

Read [Orchestration Overview](orchestration-overview.md) first for what orchestration is and when it earns its complexity.

| Tool | Notes |
|----|----|
| **Apache Airflow** | The most widely deployed orchestrator in production MLOps today; DAGs in Python (`PythonOperator`, `BashOperator`); the default choice if you have no reason to pick otherwise |
| **Prefect** | Python-native, less boilerplate than Airflow for small pipelines; growing adoption, smaller ecosystem |
| **Dagster** | Built around data assets rather than tasks — a different mental model, worth it if your pipeline is more about producing and tracking datasets than running steps |

For this track: run whichever tool locally (`airflow`` ``standalone`, or the equivalent quickstart for Prefect/Dagster) — do not set up a managed orchestration service; that is a hosting decision covered in **What Is Cloud & Infrastructure?**, not this track.

## Dataset Versioning

Recording *which version* of a dataset exists — the versioning half of the lineage discussed in [MLOps Principles](principles.md) §3.

| Tool | Notes |
|----|----|
| **DVC** | Git-like versioning for data files, no server required; the most common starting point for a small team |
| **lakeFS** | A versioning layer on top of object storage (S3-compatible); more infrastructure-facing than DVC, worth it if you’re already centred on object storage |
| **Weights & Biases Artifacts** | Convenient, but overlaps with MLflow’s role if you use both — pick one experiment-tracking-adjacent tool, don’t run two in parallel for the same purpose |

## Drift Monitoring

Detecting that the data a model sees in production has shifted from what it was trained on — see [MLOps Principles](principles.md) for why this matters and [Baseline Integration](practice/baseline-integration.md) for the minimal version every group implements regardless of assignment.

| Tool | Notes |
|----|----|
| **Evidently** | Generates drift reports and dashboards out of the box; the most common starting point |
| **NannyML** | Estimates model performance *without* needing ground-truth labels yet — a genuinely different and more advanced capability than the others here |
| **whylogs** | Lightweight profiling library, usually paired with another tool rather than used standalone |
| **Deepchecks** | Broader scope — checks both data and model, not drift alone |

## How to Choose

A reasonable process, not a formula: read each tool’s own quickstart for fifteen minutes before picking. Prefer the tool that runs locally without a server for a project this size. If two tools look equivalent, the one with better documentation for your specific use case wins — you will be reading that documentation all week.

## Explore Further

This page already names the tools; go straight to their own documentation rather than a secondary link:

- [DVC documentation](https://dvc.org/doc)

- [Evidently documentation](https://docs.evidentlyai.com/)

- [Apache Airflow documentation](https://airflow.apache.org/docs/)
