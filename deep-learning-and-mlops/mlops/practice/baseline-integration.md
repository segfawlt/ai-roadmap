# Baseline Integration

**Every group does this**, inside their own deep learning project — this is not a separate exercise, it’s the piece that connects [MLOps Principles](../principles.md) to code you already have running.

## Minimum Deliverable

Three things, added to whichever project your group is already building (Computer Vision, OCR & Document AI, or Speech AI):

1.  **Version your dataset.** Take the dataset (or a defined split of it) you’re training on and give it an identifiable version — at minimum, a content hash or a clearly named snapshot (`train_v1`, `train_v2`) that you can point back to.

2.  **Log an MLflow run that records that version.** Using `mlflow.log_input` as shown in [MLflow Basics](../mlflow-basics.md) §4, make sure at least one training run in your project logs which dataset version it used.

3.  **Add a minimal drift check.** Pick one feature or statistic from your data (average pixel brightness for images, average clip duration for audio, average field-count for documents — whatever’s simplest for your project) and write a short script that flags if a new batch of data differs from your training distribution by more than a threshold. A basic PSI (Population Stability Index) or KS-test comparison between two distributions is enough — you are not building a monitoring system, you are proving you understand what one would check.

## Why This Order

This mirrors the three-stage argument in [MLOps Principles](../principles.md): version the data, record which version a run used (lineage), then check whether new data has moved away from that version. Doing all three inside your own project — instead of as an abstract exercise — is what makes the reproducibility contract concrete rather than theoretical.

## Deliverable Format

No separate write-up required. Show this working as part of your group’s regular checkpoint and final demo — the reviewer will ask to see the MLflow run with its logged dataset input, and the drift check script run against two different batches of data.

## Hints

- Do this early, not at the end — retrofitting dataset versioning onto a project you’ve already trained five models on means those five runs have no lineage.

- Your drift check does not need a dedicated tool. A simple population-mean comparison with a printed warning is a legitimate minimum; if your group’s deep-dive area (see [Tool Deep Dive](tool-deep-dive.md)) happens to be drift monitoring, you’ll extend this with a real tool there instead of building two versions.

## References

- [MLOps Principles](../principles.md)

- [MLflow Basics](../mlflow-basics.md)
