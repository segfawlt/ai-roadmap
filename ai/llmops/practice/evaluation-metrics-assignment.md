# Assignment: RAGAS Evaluation Metrics

## Assignment Metadata

| Field               | Description                                    |
|---------------------|------------------------------------------------|
| **Assignment Name** | RAGAS Evaluation Metrics for RAG Systems       |
| **Course**          | LLMOps and Evaluation                          |
| **Project Name**    | `ragas-evaluation-lab`                         |
| **Estimated Time**  | 90 minutes                                     |
| **Framework**       | Python 3.11+, RAGAS, LangChain 1.x, OpenAI API |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Implement** RAGAS evaluation metrics for RAG systems

- **Calculate** Faithfulness scores by decomposing answers into verifiable statements

- **Measure** Answer Relevancy using reverse-engineered questions and embedding similarity

- **Evaluate** Context Precision and Context Recall for retrieval quality assessment

- **Analyze** the relationship between different metrics and overall RAG performance

------------------------------------------------------------------------

## Problem Description

You are tasked with building an evaluation pipeline for a Q&A RAG system. The system retrieves documents and generates answers, but you need to measure its quality across multiple dimensions:

1.  **Faithfulness**: Are generated answers grounded in the retrieved context?

2.  **Answer Relevancy**: Do answers actually address the user’s questions?

3.  **Context Precision**: Are relevant documents ranked higher in retrieval?

4.  **Context Recall**: Does retrieval capture all necessary information?

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `ragas` \>= 0.2

  - `langchain[openai]` \>= 1.0

  - `datasets` (HuggingFace)

### Dataset

Create or use a Q&A dataset with:

- At least 20 question-answer pairs

- Each item containing: question, ground_truth answer, retrieved contexts, generated answer

------------------------------------------------------------------------

## Tasks

### Task 1: Faithfulness Evaluation (25 points)

1.  **Implement Faithfulness scoring** that:

    - Decomposes generated answers into individual claims/statements

    - Verifies each claim against the retrieved context

    - Calculates the ratio of supported claims

2.  **Create test cases** demonstrating:

    - High faithfulness (score \> 0.9): All claims supported by context

    - Medium faithfulness (0.5-0.9): Partial support

    - Low faithfulness (\< 0.5): Hallucinated content

3.  **Document** at least 3 examples with detailed analysis of claim decomposition

### Task 2: Answer Relevancy Evaluation (25 points)

1.  **Implement Answer Relevancy scoring** that:

    - Generates N hypothetical questions from the answer

    - Computes embedding similarity with the original question

    - Returns average cosine similarity score

2.  **Test with examples** showing:

    - Complete answers (high relevancy)

    - Partial answers (medium relevancy)

    - Off-topic answers (low relevancy)

3.  **Analyze** how answer completeness affects the relevancy score

### Task 3: Context Precision & Recall (25 points)

1.  **Implement Context Precision** that:

    - Evaluates relevance of each retrieved chunk

    - Calculates Precision@k at each position

    - Computes weighted average for final score

2.  **Implement Context Recall** that:

    - Decomposes reference answer into claims

    - Checks attribution to retrieved contexts

    - Calculates coverage ratio

3.  **Create a retrieval analysis** with:

    - At least 5 queries with varying retrieval quality

    - Precision/Recall scores for each

    - Recommendations for improvement

### Task 4: End-to-End Evaluation Pipeline (25 points)

1.  **Build a complete evaluation pipeline** using RAGAS:

``` python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision, context_recall

# Your evaluation code here
```

1.  **Evaluate your RAG system** on the full dataset

2.  **Generate a report** with:

    - Summary statistics (mean, std, min, max for each metric)

    - Correlation analysis between metrics

    - Identified failure cases and root causes

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code (Jupyter notebook or Python scripts)

`README.md` with setup and usage instructions

Evaluation results table with all four metrics

Analysis report with examples and insights

Screenshots of RAGAS evaluation outputs

### Submission Checklist

All code runs without errors

All four RAGAS metrics are implemented correctly

Test cases cover edge cases and failure modes

Analysis includes actionable recommendations

Documentation is complete and clear

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                               | Points  |
|----------------------------------------|---------|
| Faithfulness implementation & analysis | 25      |
| Answer Relevancy implementation        | 25      |
| Context Precision & Recall             | 25      |
| End-to-end pipeline & reporting        | 15      |
| Code quality and documentation         | 10      |
| **Total**                              | **100** |

------------------------------------------------------------------------

## Hints

- Use the companion notebook `10_RAG_Evaluation_with_Ragas.ipynb` as a reference

- Start with small examples to understand each metric before scaling up

- For Faithfulness, pick a strong judge model (e.g., Claude Opus 4.6 or GPT-5.2) for more accurate claim verification

- When testing Answer Relevancy, vary the completeness of answers systematically

- Compare your manual calculations with RAGAS automated scores to validate understanding
