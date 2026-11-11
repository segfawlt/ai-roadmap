# Deep Learning & MLOps Training Overview

**Topic Code**: DLO_01 \| **Version**: 1.0 \| **Audience**: Interns who have finished the AI track or have equivalent Python experience

This track runs for two to three weeks. It is knowledge-based and tied to no vendor certification — the goal is that you can reason about a model and its lifecycle, not that you can recite a service catalogue.

The program makes three deliberate trade-offs, and you should understand them before week one:

1.  **The core is short; the rest is keywords.** Every lesson ends with an *Explore Further* list of five to eight terms and two or three curated links. Those lists are not optional reading — they are where most of the surface area lives. A lesson that teaches you six things and points at twenty is doing its job.

2.  **Groups specialise; nobody covers everything.** Three groups form in week one. Each takes one deep learning area and one MLOps area. Everyone reads every overview page, but only one group goes deep on any given area — and then teaches it to the others.

3.  **Live review replaces written exams.** There is no 40-question theory paper. Each week a group presents, and the reviewer *and the other two groups* question them. Cross-questioning is the mechanism: you cannot coast on a slide deck you did not write.

## Schedule

| Week | Focus |
|----|----|
| 1 | Shared core — [Machine Learning Core Concepts](machine-learning/core-concepts.md) and [Deep Learning Core Concepts](deep-learning/core-concepts.md), one reading session and one short quiz each. Then [MLOps Principles](mlops/principles.md) and [MLflow Basics](mlops/mlflow-basics.md). Groups form; deep learning and MLOps areas are assigned. Every group reads all three deep learning overviews and [MLOps Tool Landscape](mlops/tool-landscape.md), not just their own. |
| 2 | Groups build. Deep learning project and MLOps tool deep-dive run in parallel. **Mid-week review checkpoint** — present progress, answer cross-questions. Every group also completes [Baseline Integration](mlops/practice/baseline-integration.md) inside their own project. |
| 3 | Finish the project. **Final demo** — each group presents the deep learning project *and* their MLOps area. Reviewer and the other two groups ask questions live. |

If the program is compressed to two weeks, fold week 3 into the second half of week 2 and keep only the final demo.

## Group Assignments

Three groups, three deep learning areas, three MLOps areas. The deep learning area and the MLOps area are assigned independently — group A is not required to take orchestration just because it is listed first.

| Group | Deep learning area | Case study | MLOps area |
|----|----|----|----|
| A | Computer Vision | Detect product defects on a production line from images | One of: orchestration, dataset versioning, drift monitoring |
| B | OCR & Document AI | Extract fields from invoices and contracts | One of the two remaining areas |
| C | Speech AI | Transcribe call-centre recordings | The last remaining area |

Each MLOps area is taken by exactly one group. No area is left uncovered and none is doubled up. Within an area, **the group chooses its own tool** from [MLOps Tool Landscape](mlops/tool-landscape.md) — the track names candidates and trade-offs, not mandates.

## What Every Group Does Regardless of Assignment

- Reads all three deep learning overview pages and [MLOps Tool Landscape](mlops/tool-landscape.md)

- Learns MLflow from [MLflow Basics](mlops/mlflow-basics.md) — it is the shared baseline, not one group’s specialty

- Completes [Baseline Integration](mlops/practice/baseline-integration.md) inside their own project: version the dataset, log a run that records which dataset version it used, and add a minimal drift check

## Learning Outcomes

By the end of the program you should be able to:

- Explain where classical machine learning ends and deep learning begins, and why that boundary changes what you spend your time on

- Choose an evaluation metric that matches the problem, and say what it hides

- Describe what a training loop does and which architecture family fits a given data structure

- State the reproducibility contract for a model run and identify what breaks it

- Version a dataset, track an experiment, and detect that a data distribution has moved

- Read a tool landscape and justify a choice, rather than adopting whatever the tutorial used

## Assessment

| Component | How it works |
|----|----|
| Short quizzes (3) | Five to eight questions each, on the shared core only: [Machine Learning Core Concepts](machine-learning/core-concepts.md), [Deep Learning Core Concepts](deep-learning/core-concepts.md), and [MLOps Principles](mlops/principles.md) plus [MLflow Basics](mlops/mlflow-basics.md). Self-check, answers shown inline. |
| Mid-week checkpoint | Pass / not yet. Present progress, show something running, answer cross-questions. |
| Final demo | Pass / not yet. Present the deep learning project and the MLOps deep-dive. Both the reviewer and the other groups question you. |

**There is no points total and no GPA.** A group passes by clearing both review sessions. What the reviewer is listening for:

- Can you explain *why* you chose this architecture, this metric, this tool — not just what you ran?

- When the answer is “we did not get to that”, can you say what you would try next and why?

- Does the group answer, or does one person answer for everyone?

- Can you answer a question about a decision you did not personally make?

That last point is the reason for cross-questioning. Specialising by group is efficient; letting one member own all the knowledge is not.

## Prerequisites

- Python 3.11+

- Comfort with the command line and Git

- A Google or Kaggle account for free GPU notebooks (see [Project: Detect Product Defects on a Production Line](computer-vision/practice/group-project.md) for setup notes)

## What This Track Deliberately Omits

Unlike the AI, Software Engineering, and Cloud tracks, this one has **no `exams/` section and no `appendix/`**. Two to three weeks does not accommodate a 40-question theory paper or a separate reference archive; live review replaces the former and the *Explore Further* sections replace the latter. If the program is ever extended to the one-to-two-month rhythm of the other tracks, both should be added back.
