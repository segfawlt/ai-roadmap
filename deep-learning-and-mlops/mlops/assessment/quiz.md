# Quiz

## MLOps Principles & MLflow Basics

**Question 1:** A teammate reruns a training script and gets a different metric than last time, even though they didn’t touch the code. Which part of the reproducibility contract most likely changed?

- **A.** Code

- **B.** Data, hyperparameters, or environment — code was ruled out

- **C.** The metric itself

- **D.** Nothing changed; results should always vary

**Answer:** B

**Question 2:** What does `artifact_uri` represent in a run manifest?

- **A.** The run’s unique identifier

- **B.** The location of the actual trained model or output binary — the single source of truth for it

- **C.** A tag describing who ran the experiment

- **D.** The hyperparameters used for training

**Answer:** B

**Question 3:** What is the difference between dataset versioning and dataset lineage?

- **A.** They are the same concept with different names

- **B.** Versioning makes a dataset itself reproducible; lineage records which version a specific run consumed

- **C.** Lineage only applies to images; versioning only applies to tabular data

- **D.** Versioning is done in MLflow; lineage is done in the application database

**Answer:** B

**Question 4:** Why is a production model version “sealed”?

- **A.** To save storage space

- **B.** So the model can no longer be evaluated

- **C.** So that “version 7” always refers to the exact same artifact, enabling reliable audit and rollback

- **D.** Sealing is only a convention, not something registries enforce

**Answer:** C

**Question 5:** In `mlflow.log_input(dataset,`` ``context="training")`, what is being recorded?

- **A.** The trained model’s weights

- **B.** Which dataset (and version) this run consumed — dataset lineage

- **C.** The evaluation metric for the run

- **D.** A backup copy of the dataset

**Answer:** B

**Question 6:** Where should a production database (e.g. Postgres) store a trained model’s binary weights?

- **A.** Directly in a BLOB column

- **B.** It shouldn’t — it stores only the link (`run_id`, `artifact_uri`); the binary lives in object storage

- **C.** In the commit message of the training script

- **D.** Split across multiple rows, one per weight

**Answer:** B
