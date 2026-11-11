# Assignment: Post-Retrieval Processing

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Re-ranking with Cross-Encoder and Maximal Marginal Relevance |
| **Course** | RAG and Optimization |
| **Project Name** | `post-retrieval-rag` |
| **Estimated Time** | 90 minutes |
| **Framework** | Python 3.11+, LangChain 1.x, Sentence-Transformers, Cross-Encoder models |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Implement** Cross-Encoder re-ranking to improve retrieval precision

- **Apply** Maximal Marginal Relevance (MMR) to ensure result diversity

- **Compare** Bi-Encoder and Cross-Encoder architectures for re-ranking

- **Configure** the funnel strategy: retrieve many, re-rank few

- **Evaluate** the trade-offs between relevance and diversity in retrieval

------------------------------------------------------------------------

## Problem Description

Your RAG system retrieves the top-K documents using vector similarity. However, users report two issues:

1.  **Precision problems**: Sometimes highly relevant documents are ranked lower than less relevant ones

2.  **Redundancy problems**: Retrieved documents often contain duplicate or overlapping information

Your task is to implement **Cross-Encoder re-ranking** and **MMR** as post-retrieval processing steps.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain` \>= 1.0

  - `sentence-transformers` \>= 2.2.0

  - `chromadb` \>= 0.4.0

  - `numpy` \>= 1.24.0

### Models

- Bi-Encoder: `sentence-transformers/all-MiniLM-L6-v2`

- Cross-Encoder: `cross-encoder/ms-marco-MiniLM-L-6-v2`

------------------------------------------------------------------------

## Tasks

### Task 1: Implement Cross-Encoder Re-ranking (35 points)

1.  **Build a re-ranking pipeline** that:

    - Takes top-50 results from Bi-Encoder retrieval

    - Scores each (query, document) pair using Cross-Encoder

    - Returns re-ranked top-K documents

2.  **Implement the funnel strategy**:

    - Stage 1: Retrieve top-50 with Bi-Encoder (fast)

    - Stage 2: Re-rank to top-5 with Cross-Encoder (accurate)

3.  **Measure performance**:

    - Re-ranking latency per query

    - Memory usage comparison (Bi-Encoder vs Cross-Encoder)

### Task 2: Implement MMR (35 points)

1.  **Implement the MMR algorithm**:

    ```
    MMR = argmax[λ * sim(doc, query) - (1-λ) * max(sim(doc, selected_docs))]
    ```

    - Start with the most relevant document

    - Iteratively select documents balancing relevance and diversity

    - Use configurable λ parameter (default: 0.5)

2.  **Test with different λ values**:

    - λ = 1.0 (pure relevance, no diversity)

    - λ = 0.5 (balanced)

    - λ = 0.3 (prioritize diversity)

3.  **Create demonstration examples** showing:

    - Without MMR: redundant information in top-5

    - With MMR: diverse information coverage

### Task 3: Combined Pipeline and Evaluation (30 points)

1.  **Build a combined post-retrieval pipeline**:

    - Option A: Cross-Encoder first, then MMR

    - Option B: MMR first, then Cross-Encoder

    - Compare which order produces better results

2.  **Create a test set** with 10 queries including:

    - Queries prone to redundant results (biographical, product features)

    - Queries requiring precise matching (technical, factual)

3.  **Evaluation metrics**:

| Query ID | Baseline nDCG@5 | Cross-Encoder nDCG@5 | MMR Diversity Score | Combined nDCG@5 |
|----|----|----|----|----|
| Q1 |  |  |  |  |
| Q2 |  |  |  |  |
| … |  |  |  |  |

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code (Jupyter notebook or Python scripts)

`README.md` with setup and usage instructions

Performance benchmarks (latency, memory)

Evaluation results table

Example outputs showing before/after re-ranking and MMR

### Submission Checklist

Cross-Encoder re-ranking improves precision

MMR produces diverse result sets

Combined pipeline is properly implemented

Performance trade-offs are documented

Code includes clear comments and documentation

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                       | Points  |
|--------------------------------|---------|
| Cross-Encoder implementation   | 20      |
| Funnel strategy implementation | 15      |
| MMR algorithm correctness      | 20      |
| λ parameter experimentation    | 10      |
| Combined pipeline design       | 15      |
| Evaluation quality             | 10      |
| Code quality and documentation | 10      |
| **Total**                      | **100** |

------------------------------------------------------------------------

## Hints

- Use `sentence_transformers.CrossEncoder` for easy re-ranking implementation

- For MMR, cache document-document similarities to avoid recomputation

- Consider batch processing for Cross-Encoder to improve throughput

- Test your MMR implementation with a small set first (5-10 documents)

- The diversity score can be computed as the average pairwise distance between selected documents
