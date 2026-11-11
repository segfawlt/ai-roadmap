# Assignment: Advanced Indexing

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Advanced Indexing for RAG Systems |
| **Course** | RAG and Optimization |
| **Project Name** | `advanced-indexing-rag` |
| **Estimated Time** | 120 minutes |
| **Framework** | Python 3.11+, LangChain 1.x, ChromaDB/Qdrant, Sentence-Transformers |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Implement** Semantic Chunking to split text based on meaning rather than fixed character counts

- **Configure** HNSW index parameters (`M`, `ef_construction`, `ef_search`) for optimal performance

- **Compare** chunking strategies (Recursive vs. Semantic) and measure their impact on retrieval quality

- **Analyze** trade-offs between retrieval speed and accuracy when tuning HNSW parameters

- **Validate** the effectiveness of your indexing strategy through retrieval experiments

------------------------------------------------------------------------

## Problem Description

You are building a RAG system for a technical documentation platform. The current system uses fixed-size chunking (500 characters) and brute-force vector search, which causes:

1.  **Semantic fragmentation**: Important concepts are split across multiple chunks

2.  **High latency**: Search becomes slow as the document count grows

Your task is to implement **Semantic Chunking** and configure **HNSW indexing** to solve these problems.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain` \>= 1.0

  - `sentence-transformers` \>= 2.2.0

  - `chromadb` \>= 0.4.0 OR `qdrant-client` \>= 1.7.0

  - `numpy` \>= 1.24.0

### Dataset

Use the provided sample documents or create your own dataset with:

- At least 10 documents

- Each document containing multiple distinct topics/sections

- Total text length of at least 50,000 characters

------------------------------------------------------------------------

## Tasks

### Task 1: Implement Semantic Chunking (40 points)

1.  **Implement a Semantic Chunker** that:

    - Splits documents into sentences using proper sentence boundary detection

    - Calculates cosine similarity between consecutive sentences

    - Creates chunk boundaries when similarity drops below a threshold

    - Handles edge cases (very short/long sentences, code blocks, lists)

2.  **Configuration Parameters**:

    - Similarity threshold (recommended: 0.7-0.85)

    - Minimum chunk size (in sentences)

    - Maximum chunk size (in characters)

3.  **Comparison Experiment**:

    - Process the same documents using both Recursive Chunking and Semantic Chunking

    - Record the number of chunks, average chunk size, and chunking time

    - Analyze at least 3 examples where Semantic Chunking preserves context better

### Task 2: Configure HNSW Index (30 points)

1.  **Set up a vector database** (ChromaDB or Qdrant) with HNSW indexing

2.  **Experiment with HNSW parameters**:

    - Test at least 3 different values for `M` (e.g., 16, 32, 64)

    - Test at least 3 different values for `ef_construction` (e.g., 100, 200, 400)

    - Test at least 3 different values for `ef_search` (e.g., 50, 100, 200)

3.  **Document the results** in a table showing:

    - Parameter configuration

    - Index build time

    - Average query latency

    - Memory usage (if measurable)

    - Recall@10 (compared to brute-force search)

### Task 3: End-to-End RAG Pipeline (30 points)

1.  **Build a complete RAG pipeline** that uses:

    - Your Semantic Chunker for document processing

    - HNSW-indexed vector database for retrieval

    - An LLM for answer generation (can use OpenAI API or local model)

2.  **Create a test set** of at least 10 questions that require:

    - Single-topic answers (should retrieve one complete chunk)

    - Multi-topic answers (should retrieve multiple related chunks)

3.  **Evaluate and compare** retrieval quality between:

    - Baseline: Recursive Chunking + Brute-force search

    - Optimized: Semantic Chunking + HNSW index

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code in a Jupyter notebook or Python scripts

`README.md` with setup instructions and usage examples

Results table comparing chunking strategies

Results table comparing HNSW parameter configurations

Screenshots or logs showing retrieval quality comparison

### Submission Checklist

All code runs without errors

Semantic Chunker correctly preserves topic boundaries

HNSW index is properly configured and benchmarked

End-to-end pipeline produces coherent answers

Documentation is complete with clear explanations

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                         | Points  |
|----------------------------------|---------|
| Semantic Chunking implementation | 25      |
| Chunking comparison analysis     | 15      |
| HNSW parameter experimentation   | 20      |
| Performance benchmarking         | 10      |
| End-to-end RAG pipeline          | 20      |
| Code quality and documentation   | 10      |
| **Total**                        | **100** |

------------------------------------------------------------------------

## Hints

- Start with a small dataset to test your Semantic Chunker before scaling up

- Use `sentence-transformers` models like `all-MiniLM-L6-v2` for efficient similarity calculation

- When tuning HNSW, prioritize `ef_search` for query-time optimization

- Consider using the companion notebook `01-advanced-indexing.ipynb` as a reference
