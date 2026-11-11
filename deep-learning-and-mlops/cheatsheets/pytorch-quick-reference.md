# PyTorch quick reference

Enough PyTorch to get a group project off the ground. This is not a PyTorch course — it is the handful of patterns every group in this track will type in roughly the first hour, so you are not searching for boilerplate during your first working session.

## Minimal training loop

``` python
import torch
from torch import nn, optim

model = MyModel()
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=1e-3)

for epoch in range(num_epochs):
    model.train()
    for inputs, labels in train_loader:
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()

    model.eval()
    with torch.no_grad():
        val_loss = sum(criterion(model(x), y).item() for x, y in val_loader)
    print(f"epoch {epoch}: val_loss={val_loss:.4f}")
```

This is the forward pass → loss → backward pass → optimizer step loop from [Deep Learning Core Concepts](../deep-learning/core-concepts.md) §2, in code. `optimizer.zero_grad()` matters and is easy to forget — without it, gradients accumulate across batches instead of resetting.

## Dataset and DataLoader

``` python
from torch.utils.data import Dataset, DataLoader

class MyDataset(Dataset):
    def __init__(self, paths, labels, transform=None):
        self.paths = paths
        self.labels = labels
        self.transform = transform

    def __len__(self):
        return len(self.paths)

    def __getitem__(self, idx):
        image = load_image(self.paths[idx])
        if self.transform:
            image = self.transform(image)
        return image, self.labels[idx]

train_loader = DataLoader(MyDataset(train_paths, train_labels), batch_size=32, shuffle=True)
val_loader = DataLoader(MyDataset(val_paths, val_labels), batch_size=32, shuffle=False)
```

Shuffle the training loader, never the validation loader — shuffling validation data buys nothing and makes debugging harder when you want to inspect the same failing examples run to run.

## Transfer learning (image classification)

``` python
import torchvision.models as models

model = models.resnet18(weights="IMAGENET1K_V1")

# Freeze the backbone; train only the final layer first
for param in model.parameters():
    param.requires_grad = False

model.fc = nn.Linear(model.fc.in_features, num_classes)
# Only model.fc.parameters() have requires_grad=True now
```

Start frozen. If validation accuracy plateaus below what you need, unfreeze the last few blocks (not the whole network) and retrain with a smaller learning rate — this is the “discriminative fine-tuning” idea from [Deep Learning Core Concepts](../deep-learning/core-concepts.md).

## Common augmentation pipeline (images)

``` python
from torchvision import transforms

train_transform = transforms.Compose([
    transforms.RandomResizedCrop(224),
    transforms.RandomHorizontalFlip(),
    transforms.ColorJitter(brightness=0.2, contrast=0.2),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

val_transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])
```

Augment training data, not validation data — the point of augmentation is to make training harder in a way that generalizes; validation should reflect what you’ll actually see at inference time. The normalization constants above are ImageNet’s channel statistics — keep them if you’re using an ImageNet-pretrained backbone, since the backbone expects inputs on that scale.

## Saving and loading checkpoints

``` python
torch.save({
    "epoch": epoch,
    "model_state_dict": model.state_dict(),
    "optimizer_state_dict": optimizer.state_dict(),
    "val_accuracy": val_accuracy,
}, "checkpoint.pt")

checkpoint = torch.load("checkpoint.pt")
model.load_state_dict(checkpoint["model_state_dict"])
```

Save the optimizer state alongside the model if you might resume training later — Adam’s internal state matters for picking up training cleanly, not just the weights. On Colab or Kaggle, save checkpoints to mounted Drive/attached dataset storage, not the ephemeral session disk — see the group project pages for setup notes.

## Device handling

``` python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model.to(device)
inputs, labels = inputs.to(device), labels.to(device)
```

A model and its inputs must be on the same device or PyTorch raises an error immediately — this is one of the most common early mistakes, and the error message tells you exactly what happened.

## Practice

1.  Adapt the training loop above to your group’s actual model and dataset.

2.  Log training and validation loss for each epoch to MLflow using `mlflow.log_metric`, from [MLflow Basics](../mlops/mlflow-basics.md).

3.  Add the augmentation pipeline above (or an audio/text equivalent) to your `Dataset` class.

4.  Save a checkpoint after your best validation epoch, not just the last one — compare final-epoch checkpointing against best-epoch checkpointing on your own run.

5.  Confirm your model and a batch of inputs are on the same device before your first full training run, not after debugging a cryptic error.

## Review Questions

1.  What happens if you forget `optimizer.zero_grad()` inside the training loop?

2.  Why does the validation transform pipeline skip `RandomHorizontalFlip` and `ColorJitter`?

3.  When freezing a pretrained backbone, which parameters actually receive gradient updates?

4.  Why save the optimizer state, not just `model.state_dict()`, if you plan to resume training later?
