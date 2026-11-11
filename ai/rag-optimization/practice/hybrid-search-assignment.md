# Assignment: Hybrid Search

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Hybrid Search with BM25 and Reciprocal Rank Fusion |
| **Course** | RAG and Optimization |
| **Project Name** | `hybrid-search-rag` |
| **Estimated Time** | 90 minutes |
| **Framework** | Python 3.11+, LangChain 1.x, rank-bm25, Sentence-Transformers, ChromaDB |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Implement** BM25 keyword search alongside vector-based semantic search

- **Apply** Reciprocal Rank Fusion (RRF) to merge results from multiple retrievers

- **Compare** the effectiveness of Vector Search, BM25, and Hybrid Search

- **Configure** the fusion parameters to optimize retrieval quality

- **Analyze** scenarios where Hybrid Search outperforms single-method approaches

------------------------------------------------------------------------

## Problem Description

Your RAG system currently relies solely on Vector Search for retrieval. While this works well for semantic queries, users report poor results when searching for:

- Specific error codes (e.g., “Error 503 Service Unavailable”)

- Product SKUs and model numbers

- Technical terms and acronyms

- Proper names and exact phrases

Your task is to implement a **Hybrid Search** system that combines BM25 keyword matching with Vector Search, using **RRF** to merge the results.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain` \>= 1.0

  - `rank-bm25` \>= 0.2.2

  - `sentence-transformers` \>= 2.2.0

  - `chromadb` \>= 0.4.0

  - `nltk` \>= 3.8.0 (for tokenization)

### Dataset

Prepare a dataset that includes documents with:

- Technical specifications with codes/numbers

- Natural language descriptions

- Mixed content (code snippets, prose, tables)

- At least 100 documents for meaningful comparison

------------------------------------------------------------------------

## Tasks

### Task 1: Implement BM25 Retriever (25 points)

1.  **Build a BM25 retriever** that:

    - Tokenizes documents properly (handle punctuation, case normalization)

    - Indexes all documents in your corpus

    - Returns top-K documents with BM25 scores

2.  **Test with keyword-heavy queries**:

    - Create at least 5 queries containing specific codes, numbers, or technical terms

    - Verify that BM25 correctly retrieves documents with exact keyword matches

### Task 2: Implement Hybrid Search with RRF (35 points)

1.  **Create a Hybrid Retriever** that:

    - Executes both BM25 and Vector Search in parallel

    - Implements RRF score calculation: `RRF(d)`` ``=`` ``Σ`` ``1/(k`` ``+`` ``rank(d))`

    - Uses configurable `k` constant (default: 60)

    - Returns merged and re-ranked results

2.  **Handle edge cases**:

    - Documents appearing in only one result list

    - Ties in RRF scores

    - Empty results from one retriever

### Task 3: Comparative Evaluation (40 points)

1.  **Create a test set** with 20 queries categorized as:

    - **Keyword queries** (5): Exact matches, codes, identifiers

    - **Semantic queries** (5): Conceptual questions, synonyms

    - **Hybrid queries** (10): Mix of keywords and semantic intent

2.  **Evaluate each retrieval method** (Vector, BM25, Hybrid):

    - Precision@5: Proportion of relevant documents in top 5

    - Recall@10: Proportion of all relevant documents retrieved in top 10

    - Mean Reciprocal Rank (MRR): Average of 1/rank of first relevant result

3.  **Create a comparison table** showing:

| Query Type | Method | Precision@5 | Recall@10 | MRR |
|------------|--------|-------------|-----------|-----|
| Keyword    | Vector |             |           |     |
| Keyword    | BM25   |             |           |     |
| Keyword    | Hybrid |             |           |     |
| Semantic   | Vector |             |           |     |
| Semantic   | BM25   |             |           |     |
| Semantic   | Hybrid |             |           |     |
| Hybrid     | Vector |             |           |     |
| Hybrid     | BM25   |             |           |     |
| Hybrid     | Hybrid |             |           |     |

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code (Jupyter notebook or Python scripts)

`README.md` with setup and usage instructions

Evaluation results table (as shown above)

Analysis document explaining when each method excels

Screenshots showing example queries and retrieved documents

### Submission Checklist

BM25 retriever correctly matches keywords

RRF fusion produces valid merged rankings

Evaluation covers all three query types

Code is well-documented with comments

Analysis includes specific examples

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                            | Points  |
|-------------------------------------|---------|
| BM25 implementation correctness     | 15      |
| Tokenization and preprocessing      | 10      |
| RRF implementation accuracy         | 25      |
| Hybrid retriever edge case handling | 10      |
| Evaluation methodology              | 15      |
| Comparative analysis quality        | 15      |
| Code quality and documentation      | 10      |
| **Total**                           | **100** |

------------------------------------------------------------------------

## Hints

- The `rank-bm25` library provides easy BM25 implementation

- Use `nltk.word_tokenize()` for consistent tokenization

- Test RRF with small examples first to verify your formula

- Consider using the companion notebook `02-hybrid-search-rag.ipynb` as reference

- For the evaluation, manually label at least the top 10 results per query as relevant/not relevant
