# Computer Vision Overview

Everyone reads this page — only Group A goes deep on it. It exists so the other two groups can follow Group A’s demo and ask a sharp question, not so they can build a detector themselves.

## Three Different Problems, Not One

``` mermaid
        graph LR
    A[Input image] --> B["Classification:<br/>what is in this image?"]
    A --> C["Detection:<br/>where are the objects,<br/>with boxes?"]
    A --> D["Segmentation:<br/>which pixels belong<br/>to which object?"]
```

- **Classification** answers “what is this image, overall?” — one label per image. Cheapest to label, cheapest to train, and the right starting point for almost any new vision problem.

- **Detection** answers “where are the objects, and what are they?” — a bounding box plus a label per object. More expensive to label (drawing boxes) and to evaluate.

- **Segmentation** answers “which pixels belong to which object?” — a mask, not a box. The most expensive to label and the most precise output.

Pick the cheapest one that answers your actual business question. A defect line that only needs a pass/fail signal does not need segmentation; a line that needs to show an inspector exactly where the defect is might.

## Why Images Need Heavier Augmentation

[Deep Learning Core Concepts](../deep-learning/core-concepts.md) introduced augmentation as one of the things that becomes central once the input is raw. For images specifically, the reason is that a CNN with limited training data will latch onto incidental correlations — the lighting in one factory, the exact camera angle — unless it is forced to see the same object varied along dimensions that should not matter: rotated, cropped, brightness-shifted, occasionally cut out (`cutout`/`mixup`). A dataset of a few hundred defect images with no augmentation is a dataset the model will memorise, not learn from.

## Explore Further

- **Detection** — `YOLO`, `Faster`` ``R-CNN`, `IoU` (Intersection over Union — how well two boxes overlap), `mAP` (mean Average Precision — the standard detection metric), `Non-Maximum`` ``Suppression`` ``(NMS)`

- **Segmentation** — `U-Net`, `Mask`` ``R-CNN`, `Dice`` ``loss`

- **Practical workflow** — `transfer`` ``learning`` ``for`` ``vision` (start from an ImageNet-pretrained backbone, almost always), `test-time`` ``augmentation`, `error`` ``analysis` (looking at the images the model got wrong to find the actual failure pattern)

Curated starting points:

- [Ultralytics YOLO documentation](https://docs.ultralytics.com/) — the most common practical entry point for detection today

- [PyTorch Vision tutorials](https://docs.pytorch.org/tutorials/beginner/finetuning_torchvision_models_tutorial.html) — transfer learning for image classification, working code

- [Roboflow’s computer vision glossary and guides](https://roboflow.com/learn) — good for connecting the theory terms above to what you’ll actually type

## For Group A

Your project brief is at [Project: Detect Product Defects on a Production Line](practice/group-project.md). Read it before your first meeting — it defines the minimum deliverable and what counts as a stretch goal.
