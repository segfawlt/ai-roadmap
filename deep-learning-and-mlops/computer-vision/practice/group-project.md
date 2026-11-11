# Project: Detect Product Defects on a Production Line

## Group Responsible

Group A — Computer Vision.

## Minimum Deliverable

A working image classifier that labels a product photo **pass** or **defect**, trained via transfer learning on a pretrained backbone, with a short write-up of the data and architecture choices. It must run end to end from a fresh checkout — someone other than the group should be able to clone the repo, follow a `README`, and reproduce the demo.

You do not need real production-line images. Use a public dataset that maps onto the same problem shape — a labelled pass/fail visual inspection dataset (search for “surface defect detection dataset” or similar) — and treat it as if it were your own factory’s data.

## Extensions (Not Required)

If the group finishes the minimum deliverable with time to spare, in order of value:

1.  Add object detection to localise *where* the defect is, not just that one exists.

2.  Add a basic error analysis pass: pull out the misclassified images and look for a pattern (lighting, defect type, image quality).

3.  Add test-time augmentation and report whether it changes the demo’s accuracy.

Do not start an extension before the minimum deliverable runs end to end.

## Environment

Train on **Google Colab** or **Kaggle Notebooks** — both give free GPU time. Both also disconnect after a few hours, so mount Google Drive (Colab) or attach a Kaggle dataset (Kaggle) to persist checkpoints across sessions; do not lose two hours of training to a disconnect. See [PyTorch quick reference](../../cheatsheets/pytorch-quick-reference.md) for the minimal training-loop code to adapt.

## Review Schedule

- **Mid-week checkpoint** — show a training run that completes, even if accuracy is not yet good. Be ready to explain your data split and why you chose it.

- **Final demo** — run the classifier live on a held-out image, walk through your architecture and augmentation choices, and connect your work to [Baseline Integration](../../mlops/practice/baseline-integration.md) (your MLflow run should already be logging this project).

## Questions to Expect From the Other Groups

Prepare answers, not slides, for these:

- Why transfer learning instead of training from scratch, given how much data you had?

- How did you split your data, and could that split leak information?

- Your dataset is presumably imbalanced (defects are rare) — what metric did you actually optimise for, and why not accuracy?

- Show us one image your model got wrong. What do you think happened?

## Hints

- Start from a pretrained ResNet or EfficientNet, not from a randomly initialised network — you almost never have enough data to train from scratch in two weeks.

- Freeze most of the backbone at first; only unfreeze more layers if the frozen version underperforms.

- Log every run to MLflow from day one ([MLflow Basics](../../mlops/mlflow-basics.md)) — retrofitting tracking after the fact loses your early experiments.

## References

- [Computer Vision Overview](../overview.md)

- [Deep Learning Core Concepts](../../deep-learning/core-concepts.md)

- [MLflow Basics](../../mlops/mlflow-basics.md)
