# MLflow Basics

Every group in this track uses MLflow, regardless of which MLOps area they specialise in — it is the baseline, not one group’s tool. This lesson covers exactly enough to log your own project’s runs properly. Deeper MLflow features (advanced autologging, custom `pyfunc` model flavours) are left to [MLOps tools quick reference](../cheatsheets/mlops-tools-quick-reference.md) and the *Explore Further* list.

## Learning Objectives

- Log parameters, metrics, and artifacts for a training run

- Read and compare runs in the MLflow UI

- Explain what the Model Registry’s stages mean

- Record dataset lineage for a run — connecting it back to [MLOps Principles](principles.md)

## 1. Logging a Run

``` python
import mlflow

with mlflow.start_run():
    mlflow.log_param("learning_rate", 1e-3)
    mlflow.log_param("model", "resnet18")
    mlflow.log_metric("val_accuracy", 0.91)
    mlflow.log_artifact("confusion_matrix.png")
```

Each `mlflow.start_run()` block produces one run manifest — this is the concrete implementation of the structure described in [MLOps Principles](principles.md). `log_param` records a hyperparameter, `log_metric` records a result, `log_artifact` attaches a file (a plot, a model checkpoint, a report). Nothing here is exotic; the discipline is calling these consistently for every run you care about, not just the ones that turned out well.

## 2. Reading the UI

Start it locally:

``` bash
mlflow ui
```

This opens a local web UI (default `http://localhost:5000`) listing every run, sortable and filterable by parameter or metric. The habit worth building now: before starting a new experiment, glance at the UI to see what you already tried — it is far cheaper than re-running a configuration you forgot you already tested.

## 3. Model Registry, Briefly

The registry sits on top of runs and tracks named model versions through stages:

``` mermaid
        graph LR
    A[Staging] --> B[Production]
    B --> C[Archived]
```

`Staging` is for a candidate under evaluation. `Production` is the version currently serving traffic — once here, per [MLOps Principles](principles.md) §5, it is sealed. `Archived` is a retired version, kept for rollback and audit, not deleted. You will not run a real promotion workflow in this track’s timeframe; know that this exists so the vocabulary is not new when you encounter a real registry later.

## 4. Recording Dataset Lineage

This is the one MLflow feature this lesson insists on, because it is what [Baseline Integration](practice/baseline-integration.md) requires every group to do:

``` python
dataset = mlflow.data.from_pandas(df, source="data/train_v2.csv")

with mlflow.start_run():
    mlflow.log_input(dataset, context="training")
    # ... your normal training and logging
```

`log_input` attaches a record to the run of exactly which dataset (and version) it consumed. This is dataset lineage from [MLOps Principles](principles.md) §3, made concrete: the run manifest now answers “which data produced this number?” without anyone having to remember or guess.

## Explore Further

- **Beyond this lesson’s scope** — `mlflow.autolog()` (automatic logging for PyTorch and scikit-learn, less manual than the calls above), `mlflow.pyfunc` (a unified packaging format for models regardless of training framework)

- **Alternative to MLflow if your team is fully on AWS** — `SageMaker`` ``Experiments` — not taught here, one line is enough: it exists, and it plays a similar role if you are already committed to the SageMaker ecosystem.

Curated starting points:

- [MLflow Tracking Quickstart](https://mlflow.org/docs/latest/getting-started/intro-quickstart/) — the fastest path from this lesson to a working local setup

- [MLflow autologging documentation](https://mlflow.org/docs/latest/tracking/autolog.html) — worth reading once your manual logging works and starts feeling repetitive

## Review Questions

Discussion prompts for the weekly review — no single right answer, be ready to defend yours. The graded short quiz for this module is at [Quiz](assessment/quiz.md) (covers both this page and [MLOps Principles](principles.md)).

1.  What is the difference between what `log_param` and `log_metric` are each meant to store? Give an example that would be wrong to log with the other call.

2.  Why does `log_input` matter even if you can remember which CSV file you used?

3.  Your group has been re-running the same training script and eyeballing results in the terminal. What does moving to MLflow actually buy you, concretely, this week — not in the abstract?
