# MLOps Principles

This is the one lesson in this track kept at full depth rather than compressed toward keywords. Every other lesson here assumes you understand the ideas in this page — they are the reasoning behind the tools, not tool-specific knowledge that ages out in a year.

## Learning Objectives

- State the reproducibility contract a model run must satisfy

- Describe the structure of a run manifest and why each field exists

- Explain dataset lineage as distinct from dataset versioning for data engineering purposes

- Explain why a promoted model version is sealed, and what “seal” actually means operationally

## 1. The Reproducibility Contract

A model result is reproducible if, given the same code, the same data, the same hyperparameters, and the same environment, you get the same metric.

``` mermaid
        graph LR
    A[Code] --> E[Reproducibility contract]
    B[Data] --> E
    C[Hyperparameters] --> E
    D[Environment] --> E
    E --> F[Metric]
```

Every one of the four inputs can silently change without anyone noticing, and each has a specific failure it causes:

- **Code** changes — you fixed a bug in preprocessing after training, and now nobody can tell which version of the code produced the reported metric.

- **Data** changes — the training set gets re-pulled from a source table that has since been updated, and “the same run” now trains on different rows.

- **Hyperparameters** are lost — someone remembers “we used a learning rate around 1e-3” but not the exact value, and the exact run can never be recreated.

- **Environment** drifts — a library version bump changes numerical behaviour in ways too subtle to show up as an error, only as a slightly different number six weeks later.

None of these are hypothetical. They are the default outcome unless something actively records all four. That something is experiment tracking, and the unit it produces is the run manifest.

## 2. The Run Manifest

A **run manifest** is the record of one training run — everything needed to answer “what exactly produced this number?” without asking anyone to remember. At minimum it captures:

| Field | What it answers |
|----|----|
| `run_id` | Which run is this, unambiguously? |
| `artifact_uri` | Where is the actual trained model / output stored? |
| Parameters | What hyperparameters were used? |
| Metrics | What did the run achieve? |
| Tags | Free-form context — who ran it, what experiment it belongs to |
| Source | What code (commit, script) produced it? |

This is not a set of logging calls you sprinkle in — it is a structure. If any field is missing, the run is not fully reproducible even if the others are complete. [MLflow Basics](mlflow-basics.md) is the concrete tool that implements this structure in practice; this lesson is about the structure itself, which would still matter even if you used a different tool.

## 3. Dataset Lineage vs. Dataset Versioning

These sound like the same idea and are not. Data and model/code version independently, on two separate axes — a run is defined by a specific point on **both axes at once**, not by either alone:

``` mermaid
        graph TD
    subgraph "Data axis"
        D1[Dataset v1] --> D2[Dataset v2] --> D3[Dataset v3]
    end
    subgraph "Model axis"
        M1[Model v1] --> M2[Model v2]
    end
    D2 -.->|used by| R1["Run A<br/>data v2 × model v1"]
    M1 -.->|used by| R1
    D2 -.->|used by| R2["Run B<br/>data v2 × model v2"]
    M2 -.->|used by| R2
    D3 -.->|used by| R3["Run C<br/>data v3 × model v2"]
    M2 -.->|used by| R3
```

Run A and Run B share the same dataset version but differ in model version — comparing their metrics tells you whether the model change helped. Run B and Run C share the same model version but differ in data — comparing those tells you whether the data change mattered. Losing track of which axis moved between two runs is the single most common way a team draws the wrong conclusion from an experiment.

- **Dataset versioning** (a data engineering concern) is about making a dataset itself reproducible and shareable — “version 3 of the training set exists as an immutable artifact anyone can pull.”

- **Dataset lineage** (an MLOps concern, and the one this lesson cares about) is about recording *which dataset version a specific run used* — “run `a3f9c1` trained on dataset version 3.” Lineage is the link between a run manifest and a dataset version, not the dataset versioning mechanism itself.

You can have perfect dataset versioning and still have no lineage, if nothing records which version each run consumed. [Baseline Integration](practice/baseline-integration.md) is where you implement both halves together for your own project.

## 4. Backend Integration: What Belongs Where

When a model registry sits behind an application backend, one rule matters more than any specific schema: **`artifact_uri` is the single source of truth for the model or dataset binary.** The application database (commonly Postgres) stores only the link — `run_id`, `artifact_uri` — never the blob itself. Databases are bad at storing large binaries and worse at versioning them; object storage is built for exactly this.

If your team is designing the schema for these links, that is standard relational modelling — the same CRUD and schema skills from **Data Persistence** apply directly. This lesson does not teach that schema design; it only states the constraint the schema has to respect.

## 5. Sealed Versions

Once a model version is **promoted to production**, it is treated as **sealed** — immutable. You do not edit a production model version in place. Any change, however small, creates a *new* version; the old one remains exactly as it was, available for rollback.

This is not a technical limitation, it is a deliberate policy, and the reason is auditability: if a production model’s behaviour changes, you need to be able to say precisely which version was live at any point in time, and be certain that “version 7” means the same artifact today that it meant last month. A registry that allows silent overwrites cannot make that guarantee.

## Explore Further

- **Registry mechanics** — `Model`` ``Registry`, `stage`` ``transitions` (Staging → Production → Archived), `model`` ``card`, `A/B`` ``testing`` ``between`` ``model`` ``versions`

- **Governance** — `audit`` ``trail`, `model`` ``rollback`, `champion/challenger`

Curated starting points:

- [MLflow Model Registry documentation](https://mlflow.org/docs/latest/model-registry.html) — the concrete implementation of everything in §2 and §5

- [Google Cloud’s “MLOps: Continuous delivery and automation pipelines” whitepaper](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning) — vendor-neutral in its concepts despite the URL, a good second pass on this whole lesson

## Review Questions

Discussion prompts for the weekly review — no single right answer, be ready to defend yours. The graded short quiz is at [Quiz](assessment/quiz.md).

1.  A teammate says “I re-ran the same script and got a different accuracy.” Walk through the four parts of the reproducibility contract and identify which one most likely changed.

2.  Why does dataset lineage matter even if your dataset versioning is already perfect?

3.  Someone proposes patching a bug directly on the production model version instead of publishing a new one. What goes wrong with that, specifically?

4.  Design (in words, not code) what a Postgres table linking your application to MLflow runs would need to store, and what it must never store.
