# Assignment: Query Transformation

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Query Transformation with HyDE and Decomposition |
| **Course** | RAG and Optimization |
| **Project Name** | `query-transformation-rag` |
| **Estimated Time** | 90 minutes |
| **Framework** | Python 3.11+, LangChain 1.x, OpenAI API, Sentence-Transformers |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Implement** Hypothetical Document Embeddings (HyDE) for improved query-document matching

- **Build** Query Decomposition pipelines to handle complex multi-part questions

- **Design** effective prompts for LLM-based query transformation

- **Evaluate** the impact of query transformation on retrieval quality

- **Apply** these techniques to real-world RAG scenarios

------------------------------------------------------------------------

## Problem Description

Users of your RAG system often submit queries that perform poorly in retrieval:

1.  **Short, vague queries**: “remote work” instead of “What are the company’s remote work policies?”

2.  **Question-answer mismatch**: Questions are interrogative while documents are declarative

3.  **Complex multi-part queries**: “Compare the battery life and camera quality of iPhone 15 and Samsung S24”

Your task is to implement **HyDE** and **Query Decomposition** to transform user queries before retrieval.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain[openai]` \>= 1.0

  - `sentence-transformers` \>= 2.2.0

  - `chromadb` \>= 0.4.0

### API Requirements

- OpenAI API key (or compatible LLM endpoint)

- Embedding model (e.g., `text-embedding-3-small` or local `all-MiniLM-L6-v2`)

------------------------------------------------------------------------

## Tasks

### Task 1: Implement HyDE (35 points)

1.  **Build a HyDE pipeline** with three stages:

    - **Generate**: Use LLM to create a hypothetical answer paragraph

    - **Encode**: Convert the hypothetical answer to an embedding

    - **Retrieve**: Search using the hypothetical answer embedding

2.  **Design the generation prompt** that:

    - Instructs the LLM to write in the style of your target documents

    - Includes domain-specific vocabulary guidance

    - Handles different query types (how-to, what-is, troubleshooting)

3.  **Test with at least 5 query types**:

    - Short queries (1-3 words)

    - Technical troubleshooting queries

    - Conceptual/definition queries

    - How-to procedure queries

    - Comparison queries

### Task 2: Implement Query Decomposition (35 points)

1.  **Build a Query Decomposition pipeline**:

    - Use LLM to analyze if a query contains multiple sub-questions

    - Generate independent sub-queries for parallel retrieval

    - Aggregate retrieved documents from all sub-queries

    - Synthesize a final answer using all gathered context

2.  **Design the decomposition prompt** that:

    - Identifies multiple intents within a single question

    - Generates standalone sub-queries (each understandable without context)

    - Preserves important constraints and filters from the original query

3.  **Handle these decomposition scenarios**:

    - Comparison queries (A vs B)

    - Aggregation queries (list all X with property Y)

    - Sequential queries (first do A, then what happens to B?)

### Task 3: Comparative Evaluation (30 points)

1.  **Create a test set** with 15 queries:

    - 5 queries suitable for HyDE (short/vague)

    - 5 queries suitable for Decomposition (complex/multi-part)

    - 5 baseline queries (clear, single-intent)

2.  **Compare retrieval quality** across methods:

| Query ID | Query Type | Baseline Recall@5 | HyDE Recall@5 | Decomposition Recall@5 |
|----------|------------|-------------------|---------------|------------------------|
| Q1       | Short      |                   |               |                        |
| Q2       | Complex    |                   |               |                        |
| …        | …          |                   |               |                        |

1.  **Analyze LLM outputs**:

    - Document 3 example HyDE generations and their effectiveness

    - Document 3 example decompositions and sub-query quality

    - Identify failure cases and suggest improvements

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code (Jupyter notebook or Python scripts)

`README.md` with setup and usage instructions

Prompt templates used for HyDE and Decomposition

Evaluation results table

Analysis document with examples and failure case analysis

### Submission Checklist

HyDE correctly generates hypothetical documents

Query Decomposition produces valid sub-queries

Both pipelines integrate with the retrieval system

Evaluation demonstrates improvement over baseline

Documentation includes prompt design rationale

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                        | Points  |
|---------------------------------|---------|
| HyDE implementation correctness | 20      |
| HyDE prompt design quality      | 15      |
| Decomposition implementation    | 20      |
| Decomposition prompt design     | 15      |
| Evaluation methodology          | 15      |
| Analysis and examples quality   | 10      |
| Code quality and documentation  | 5       |
| **Total**                       | **100** |

------------------------------------------------------------------------

## Hints

- For HyDE, the hypothetical answer doesn’t need to be factually correct—focus on style and vocabulary

- Use few-shot examples in your prompts for more consistent LLM outputs

- Consider caching LLM responses during development to save API costs

- The companion notebook `03-langchain-hyde-demo.ipynb` provides a starting point

- For decomposition, test that each sub-query makes sense in isolation
