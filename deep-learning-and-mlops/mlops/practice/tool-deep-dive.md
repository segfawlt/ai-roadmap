# Tool Deep Dive

**One group per area.** Each of the three groups is assigned exactly one of orchestration, dataset versioning, or drift monitoring (see [Deep Learning & MLOps Training Overview](../../overview.md) for which group has which). Within your assigned area, you choose the specific tool yourselves from [MLOps Tool Landscape](../tool-landscape.md) — nothing here is pre-selected for you.

## Minimum Deliverable

Take the pipeline your group already built for [Baseline Integration](baseline-integration.md) — version data → train → track in MLflow → check drift — and automate or deepen exactly the piece that matches your assigned area, using the tool you chose:

- **Orchestration** — Write a DAG (or your chosen tool’s equivalent) that runs your baseline pipeline’s steps in order: validate data → train → track → check drift. Three to five tasks is enough; this is about demonstrating dependency and scheduling, not building a production pipeline.

- **Dataset versioning** — Replace the manual hash/snapshot approach from [Baseline Integration](baseline-integration.md) with your chosen tool’s real versioning mechanism, and show you can retrieve a specific past version on demand.

- **Drift monitoring** — Replace the minimal PSI/KS-test script from [Baseline Integration](baseline-integration.md) with your chosen tool’s report or dashboard, run against two batches of your project’s data (for example, your original training set vs. a deliberately shifted sample).

## Deliverable Format

A live demo at the final review, run against your group’s actual project data — not a toy example unconnected to [Baseline Integration](baseline-integration.md). Be ready to explain why you chose this specific tool over the other options listed in [MLOps Tool Landscape](../tool-landscape.md).

## Questions to Expect From the Other Groups

- What does this tool give you that the minimal version in `baseline-integration` didn’t?

- What would break first if your data or project scaled up 100x?

- Why this tool and not \[name a specific alternative from the tool landscape\]?

## Hints

- Do not start this until [Baseline Integration](baseline-integration.md) works — the deep dive extends it, it does not replace the reasoning behind it.

- Read your chosen tool’s own quickstart guide before anything else; a first working example usually exists in that tool’s documentation already.

## References

- [MLOps Tool Landscape](../tool-landscape.md)

- [Baseline Integration](baseline-integration.md)

- [Orchestration Overview](../orchestration-overview.md)
