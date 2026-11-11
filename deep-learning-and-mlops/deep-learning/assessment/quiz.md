# Quiz

## Deep Learning Core Concepts

**Question 1:** What does a `DataLoader` do that a classical ML pipeline typically does not need?

- **A.** Encode categorical columns

- **B.** Yield shuffled batches of raw input on the fly

- **C.** Scale numeric features

- **D.** Compute a correlation matrix

**Answer:** B

**Question 2:** In one training iteration, what happens right after the forward pass?

- **A.** The optimizer updates the weights

- **B.** The loss is computed by comparing predictions to labels

- **C.** The next batch is loaded

- **D.** The model is evaluated on the test set

**Answer:** B

**Question 3:** Training loss keeps falling while validation loss starts rising after several epochs. What is this?

- **A.** Underfitting

- **B.** A learning rate that is too low

- **C.** Overfitting — the same concept as in classical ML, showing up across epochs

- **D.** A sign the batch size is too large

**Answer:** C

**Question 4:** Which architecture family is the standard first choice for image data, and why?

- **A.** RNN, because images have sequential structure

- **B.** CNN, because convolution shares weights across spatial positions

- **C.** Linear regression, because images are just large feature vectors

- **D.** Decision trees, because they handle high-dimensional data well

**Answer:** B

**Question 5:** You have 300 labelled images of a rare defect. What is the more reasonable starting point?

- **A.** Train a CNN from scratch — more control

- **B.** Fine-tune a model pretrained on a large image dataset

- **C.** Skip deep learning and only use rule-based detection

- **D.** Collect 300,000 more images before starting

**Answer:** B

**Question 6:** Loss suddenly becomes `NaN` partway through training. What is the most likely cause?

- **A.** The dataset is too large

- **B.** The learning rate is too high, or gradients are exploding

- **C.** The validation set is too small

- **D.** The model has too few layers

**Answer:** B
