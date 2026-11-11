# MLOps tools quick reference

MLflow, Airflow, DVC, and Evidently, each reduced to the minimal pattern you need to get moving. This page assumes you’ve read [MLOps Principles](../mlops/principles.md) and [MLOps Tool Landscape](../mlops/tool-landscape.md) — it is a syntax lookup, not an explanation of why these tools exist.

## MLflow

``` bash
pip install mlflow
mlflow ui  # http://localhost:5000
```

``` python
import mlflow

with mlflow.start_run():
    mlflow.log_param("lr", 1e-3)
    mlflow.log_metric("val_accuracy", 0.91)
    mlflow.log_artifact("model.pt")

    dataset = mlflow.data.from_pandas(df, source="data/train_v2.csv")
    mlflow.log_input(dataset, context="training")
```

Autologging, if you want less manual instrumentation:

``` python
import mlflow.pytorch
mlflow.pytorch.autolog()
```

## Apache Airflow

``` bash
pip install apache-airflow
airflow standalone  # runs scheduler + webserver + a throwaway admin user
```

``` python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG("baseline_pipeline", start_date=datetime(2026, 1, 1), schedule_interval="@daily") as dag:
    validate = PythonOperator(task_id="validate_data", python_callable=validate_data)
    train = PythonOperator(task_id="train", python_callable=train_model)
    evaluate = PythonOperator(task_id="evaluate", python_callable=evaluate_model)

    validate >> train >> evaluate
```

`>>` sets task dependency order — this DAG runs `validate_data`, then `train`, then `evaluate`, and Airflow’s UI shows the graph and per-task logs. Use `airflow`` ``standalone` for local development only; production Airflow needs a real scheduler and metadata database setup, which is out of scope here.

## DVC

``` bash
pip install dvc
dvc init
dvc add data/train.csv          # creates data/train.csv.dvc, a small pointer file
git add data/train.csv.dvc .gitignore
git commit -m "chore: track training data v1 with DVC"
```

``` bash
dvc remote add -d storage s3://my-bucket/dvc-store   # or a local path for this track
dvc push                                              # upload the actual data
dvc pull                                              # fetch it on another machine
```

The `.dvc` pointer file is what you commit to Git; the actual data lives wherever your DVC remote points (S3, a local directory, etc.). This is the versioning half of the lineage distinction in [MLOps Principles](../mlops/principles.md) §3 — DVC versions the dataset, `mlflow.log_input` records which version a run used.

## Evidently

``` bash
pip install evidently
```

``` python
from evidently import Report
from evidently.presets import DataDriftPreset

report = Report(metrics=[DataDriftPreset()])
result = report.run(reference_data=train_df, current_data=new_batch_df)
result.save_html("drift_report.html")
```

`reference_data` is what your model was trained on; `current_data` is the new batch you’re checking. The generated HTML report flags which columns have drifted and by how much — open it in a browser, it’s built to be read directly rather than parsed.

## Which one am I reaching for?

| Situation                                                    | Tool      |
|--------------------------------------------------------------|-----------|
| “What did I log for this run three days ago?”                | MLflow UI |
| “Run these four steps in order, on a schedule, with retries” | Airflow   |
| “Which exact version of this CSV did I train on?”            | DVC       |
| “Has my incoming data drifted from what I trained on?”       | Evidently |

## Practice

1.  Install MLflow and log one real run from your group’s project, including `log_input` for your dataset.

2.  If your group’s assigned MLOps area (from [MLOps Tool Landscape](../mlops/tool-landscape.md)) is orchestration, write a 3-task DAG that mirrors your `baseline-integration` pipeline.

3.  If your area is dataset versioning, run `dvc`` ``add` on your actual training data and confirm you can `dvc`` ``pull` it fresh in a new directory.

4.  If your area is drift monitoring, run `DataDriftPreset` comparing your training data against a deliberately shifted sample and read the generated report.

## Review Questions

1.  What does a `.dvc` file contain, and what does it deliberately not contain?

2.  In the Airflow DAG above, what would happen if `train` were defined before `validate`` ``>>`` ``train`? Would the pipeline still run in the right order?

3.  Evidently’s report needs both `reference_data` and `current_data`. What would happen if you accidentally passed the same DataFrame as both?

4.  Which of these four tools would you reach for to answer “did we already try a learning rate of 1e-4 last week”?
