# Assignment: RAG Architecture Experiment Comparison

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | RAG Architecture Experiment Comparison |
| **Course** | LLMOps and Evaluation |
| **Project Name** | `rag-experiment-comparison` |
| **Estimated Time** | 180 minutes |
| **Framework** | Python 3.11+, LangChain 1.x, ChromaDB/Qdrant, Neo4j (optional), RAGAS, OpenAI |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Design** a rigorous experimental framework for evaluating RAG systems

- **Implement** at least two RAG architectures (Naive and Advanced)

- **Evaluate** systems using RAGAS metrics and custom benchmarks

- **Analyze** trade-offs between quality, cost, and latency

- **Derive** actionable recommendations for architecture selection

------------------------------------------------------------------------

## Problem Description

Your team needs to choose the right RAG architecture for a new product. You must conduct a scientific comparison to answer:

1.  **Quality**: Which architecture produces the most accurate and faithful answers?

2.  **Cost**: What is the cost per query for each architecture?

3.  **Latency**: How does response time vary across architectures?

4.  **Use Cases**: Which architecture fits which query types?

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain[openai]` \>= 1.0

  - `chromadb` \>= 0.4.0 OR `qdrant-client` \>= 1.7.0

  - `ragas` \>= 0.2

  - `sentence-transformers` \>= 2.2.0

  - (Optional) `neo4j` \>= 5.0 for GraphRAG

### Dataset Requirements

- At least 10 documents (minimum 50,000 characters total)

- At least 30 test questions categorized by type:

  - Factual (40%): Simple fact lookup

  - Relational (30%): Relationship between concepts

  - Multi-hop (20%): Requires connecting multiple facts

  - Analytical (10%): Summarization or trend analysis

------------------------------------------------------------------------

## Tasks

### Task 1: Implement RAG Architectures (40 points)

1.  **Naive RAG (Required)**:

    - Fixed-size chunking (500-1000 characters)

    - Standard embedding model (e.g., `text-embedding-3-small`)

    - Top-K retrieval (k=5)

    - Direct LLM generation

2.  **Advanced RAG (Required)**:

    - Semantic or recursive chunking

    - Hybrid search (Vector + BM25)

    - Query transformation (HyDE or similar)

    - Re-ranking with Cross-Encoder

3.  **GraphRAG (Bonus - 10 extra points)**:

    - Entity and relationship extraction

    - Knowledge graph construction

    - Graph-based retrieval

4.  **Document your implementations** with architecture diagrams

### Task 2: Create Evaluation Dataset (15 points)

1.  **Prepare documents**:

    - Select or create documents with clear topics

    - Ensure coverage of different complexity levels

2.  **Generate test questions**:

    - Create questions for each category (Factual, Relational, Multi-hop, Analytical)

    - Provide ground truth answers

    - Tag questions with expected difficulty

3.  **Validate dataset**:

    - Ensure questions are unambiguous

    - Verify ground truth accuracy

    - Document any assumptions

### Task 3: Run Experiments (25 points)

1.  **Execute evaluation** for each architecture:

    - Run all test questions through each system

    - Capture responses, contexts, and metadata

    - Record latency for each query

2.  **Calculate metrics** using RAGAS:

    - Faithfulness

    - Answer Relevancy

    - Context Precision

    - Context Recall

3.  **Track costs**:

    - Embedding API calls

    - LLM API calls

    - Total cost per query

4.  **Compile results table**:

| System | Faithfulness | Answer Rel. | Context Prec. | Context Rec. | Latency (s) | Cost (\$) |
|----|----|----|----|----|----|----|
| Naive |  |  |  |  |  |  |
| Advanced |  |  |  |  |  |  |

### Task 4: Analysis and Recommendations (20 points)

1.  **Performance by query type**:

    - Break down metrics by question category

    - Identify strengths and weaknesses of each architecture

2.  **Trade-off analysis**:

    - Quality vs. Cost

    - Quality vs. Latency

    - Create visualizations (charts/graphs)

3.  **Write recommendations** (300-500 words):

    - When to use each architecture

    - Optimization priorities for each

    - Your recommended default choice with justification

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code for all implemented architectures

`README.md` with setup and execution instructions

Test dataset with questions and ground truth

Results table with all metrics

Analysis report with visualizations

Architecture decision recommendation

### Submission Checklist

At least 2 RAG architectures are implemented

Test dataset has 30+ categorized questions

All RAGAS metrics are calculated

Cost and latency are tracked

Analysis includes actionable insights

Code is well-documented

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                           | Points  |
|------------------------------------|---------|
| RAG architecture implementations   | 40      |
| Evaluation dataset quality         | 15      |
| Experiment execution & metrics     | 25      |
| Analysis and recommendations       | 15      |
| Code quality and documentation     | 5       |
| **Total**                          | **100** |
| **Bonus: GraphRAG implementation** | **+10** |

------------------------------------------------------------------------

## Hints

- Start with Naive RAG to establish a baseline, then add complexity

- Use the same embedding model across architectures for fair comparison

- For Advanced RAG, consider LangChain’s built-in query transformers

- When analyzing results, look for patterns in failure cases

- Consider statistical significance when comparing small differences

- The companion notebooks can help with RAGAS evaluation setup
