# Quiz

## Machine Learning Core Concepts

**Question 1:** A defect-detection dataset is 2% defective. Which metric is most likely to mislead you here?

- **A.** Precision

- **B.** Recall

- **C.** Accuracy

- **D.** PR-AUC

**Answer:** C

**Question 2:** What is the key difference between a loss function and an evaluation metric?

- **A.** They are the same thing, just different names

- **B.** The loss function is what training optimises; the evaluation metric is what you actually judge the model by, and the two can disagree

- **C.** Loss functions only apply to regression

- **D.** Evaluation metrics must be differentiable and loss functions do not

**Answer:** B

**Question 3:** You have sensor readings from 50 machines, one row per minute. Why is a random train/test split risky here?

- **A.** Random splits are always invalid

- **B.** Readings from the same machine are correlated, so a random split lets information leak from train into test

- **C.** Random splits only work for classification

- **D.** It is not risky — random splits are always safest

**Answer:** B

**Question 4:** Which of these is a sign of label leakage?

- **A.** The model’s validation accuracy is far below training accuracy

- **B.** A feature is only available after the outcome you’re predicting has already happened

- **C.** The model trains slowly

- **D.** The loss function does not converge

**Answer:** B

**Question 5:** On a tabular dataset with 5,000 rows, which is the more reasonable first model to try?

- **A.** A deep neural network trained from scratch

- **B.** Gradient boosting (XGBoost or LightGBM)

- **C.** A Vision Transformer

- **D.** Whatever the most recent paper recommends

**Answer:** B

**Question 6:** What does it mean for classical ML to rely on “feature engineering” while deep learning relies on “representation learning”?

- **A.** They mean the same thing

- **B.** In classical ML a human designs the input features; in deep learning the model learns its own features from raw data

- **C.** Representation learning is a synonym for regularization

- **D.** Feature engineering only applies to unsupervised learning

**Answer:** B
