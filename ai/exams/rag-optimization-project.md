# Final Exam: Enterprise RAG System

## overview

| Field             | Value                 |
|-------------------|-----------------------|
| **Course**        | RAG and Optimization  |
| **Duration**      | 240 minutes (4 hours) |
| **Passing Score** | 70%                   |
| **Total Points**  | 100                   |

------------------------------------------------------------------------

## Description

You have been hired as an **AI Engineer** at **TechDocs Inc.**, a company that provides enterprise documentation solutions. Your task is to build a production-ready **Enterprise RAG System** that can answer complex questions about technical documentation, company policies, and product specifications.

The current basic RAG system has several limitations:

- Poor retrieval quality due to fixed-size chunking

- Slow search performance with growing document collections

- Inability to handle keyword-specific queries (error codes, product IDs)

- Redundant and irrelevant results in retrieved documents

- Missing relationship information between entities (policies, stakeholders, regulations)

You must apply **all five optimization techniques** learned in this module to build a comprehensive, production-grade RAG system.

------------------------------------------------------------------------

## Objectives

By completing this exam, you will demonstrate mastery of:

- Implementing **Semantic Chunking** for intelligent document segmentation

- Configuring **HNSW Index** for high-performance vector search

- Building **Hybrid Search** combining BM25 and Vector Search with RRF fusion

- Applying **Query Transformation** techniques (HyDE and Query Decomposition)

- Implementing **Post-Retrieval Processing** with Cross-Encoder and MMR

- Designing a **GraphRAG** architecture for relationship-aware retrieval

------------------------------------------------------------------------

## Problem Description

Build an **Enterprise RAG System** named `enterprise-rag-system` that processes a collection of technical documents and provides accurate, contextual answers to user queries. The system must handle:

1.  **Technical documentation** with code snippets, error codes, and specifications

2.  **Policy documents** with stakeholder relationships and regulatory references

3.  **Product catalogs** with model numbers, features, and comparisons

The system should intelligently route queries to the appropriate retrieval strategy and provide high-quality, diverse, and accurate results.

------------------------------------------------------------------------

## Assumptions

- You have access to sample documents (technical docs, policies, product specs) or will use provided sample data

- OpenAI API key or compatible LLM endpoint is available

- Neo4j database is available (local Docker or cloud instance)

- Python 3.11+ environment with necessary packages installed

- Basic understanding of all five RAG optimization techniques

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain[anthropic]` \>= 1.0 \# or `langchain[openai]`

  - `langchain-neo4j` \>= 0.1.0

  - `sentence-transformers` \>= 2.2.0

  - `chromadb` \>= 0.4.0 OR `qdrant-client` \>= 1.7.0

  - `rank-bm25` \>= 0.2.2

  - `pydantic` \>= 2.0.0

  - `neo4j` \>= 5.0.0

### Infrastructure

- Vector Database: ChromaDB or Qdrant with HNSW indexing

- Graph Database: Neo4j (Docker recommended)

- Embedding Model: `text-embedding-3-small` or `all-MiniLM-L6-v2`

- Cross-Encoder: `cross-encoder/ms-marco-MiniLM-L-6-v2`

- LLM: A frontier model such as Claude Sonnet 4.6, GPT-5.2, or Gemini 3

------------------------------------------------------------------------

## Tasks

### Task 1: Advanced Indexing Pipeline (20 points)

**Time Allocation**: 45 minutes

Implement an intelligent document indexing pipeline that preserves semantic coherence.

#### Requirements:

1.  **Semantic Chunking Implementation**

    - Build a chunker that splits documents based on semantic similarity between sentences

    - Configure similarity threshold (0.7-0.85) and chunk size limits

    - Handle edge cases: code blocks, tables, lists, short documents

2.  **HNSW Index Configuration**

    - Set up vector database with HNSW indexing

    - Configure optimal parameters: `M=32`, `ef_construction=200`, `ef_search=100`

    - Document the trade-offs for your chosen configuration

3.  **Indexing Pipeline**

    - Process at least 20 documents through the pipeline

    - Store metadata (source, chunk_id, document_type) with each vector

    - Implement batch processing for efficiency

#### Deliverables:

- `indexing/semantic_chunker.py`

- `indexing/vector_store.py`

- Indexed document collection with metadata

------------------------------------------------------------------------

### Task 2: Hybrid Search Implementation (20 points)

**Time Allocation**: 45 minutes

Build a hybrid retrieval system that combines keyword and semantic search.

#### Requirements:

1.  **BM25 Retriever**

    - Implement BM25 indexing for all document chunks

    - Proper tokenization with case normalization and punctuation handling

    - Return top-K results with BM25 scores

2.  **Hybrid Search with RRF**

    - Execute both BM25 and Vector Search in parallel

    - Implement RRF fusion: `RRF(d)`` ``=`` ``Σ`` ``1/(60`` ``+`` ``rank(d))`

    - Handle documents appearing in only one result list

3.  **Query Router**

    - Analyze query to determine optimal search strategy

    - Route keyword-heavy queries to prioritize BM25

    - Route semantic queries to prioritize Vector Search

    - Use Hybrid Search as default

#### Deliverables:

- `retrieval/bm25_retriever.py`

- `retrieval/hybrid_search.py`

- `retrieval/query_router.py`

------------------------------------------------------------------------

### Task 3: Query Transformation Layer (15 points)

**Time Allocation**: 35 minutes

Implement query transformation to handle vague and complex queries.

#### Requirements:

1.  **HyDE Implementation**

    - Generate hypothetical answer paragraphs using LLM

    - Use hypothetical answer embedding for retrieval

    - Design domain-appropriate generation prompts

2.  **Query Decomposition**

    - Detect multi-part questions requiring information from multiple sources

    - Generate independent sub-queries for parallel retrieval

    - Aggregate results from all sub-queries

3.  **Transformation Router**

    - Classify queries: simple, vague (use HyDE), complex (use Decomposition)

    - Apply appropriate transformation before retrieval

#### Deliverables:

- `transformation/hyde.py`

- `transformation/query_decomposition.py`

- `transformation/transformation_router.py`

------------------------------------------------------------------------

### Task 4: Post-Retrieval Processing (15 points)

**Time Allocation**: 35 minutes

Implement re-ranking and diversity optimization for retrieved results.

#### Requirements:

1.  **Cross-Encoder Re-ranking**

    - Retrieve top-50 candidates with Bi-Encoder

    - Re-rank using Cross-Encoder (`cross-encoder/ms-marco-MiniLM-L-6-v2`)

    - Return top-10 re-ranked results

2.  **MMR for Diversity**

    - Implement MMR algorithm with configurable λ parameter

    - Default λ=0.5 for balanced relevance/diversity

    - Ensure diverse information coverage in final results

3.  **Configurable Pipeline**

    - Support both: Cross-Encoder → MMR and MMR → Cross-Encoder orders

    - Allow configuration of k values at each stage

#### Deliverables:

- `post_retrieval/cross_encoder_reranker.py`

- `post_retrieval/mmr.py`

- `post_retrieval/post_retrieval_pipeline.py`

------------------------------------------------------------------------

### Task 5: GraphRAG Integration (20 points)

**Time Allocation**: 50 minutes

Build a knowledge graph for relationship-aware retrieval.

#### Requirements:

1.  **Entity Extraction**

    - Define Pydantic models for domain entities (Policy, Stakeholder, Product, Regulation, etc.)

    - Extract entities and relationships using LLM with structured output

    - Validate extracted data against schema

2.  **Knowledge Graph Construction**

    - Populate Neo4j with extracted entities and relationships

    - Use MERGE to prevent duplicates

    - Create appropriate indexes for query performance

3.  **Graph-Aware Retrieval**

    - Implement natural language to Cypher translation

    - Support relationship traversal queries

    - Combine graph results with vector search results

#### Deliverables:

- `graph/entity_models.py`

- `graph/entity_extractor.py`

- `graph/knowledge_graph.py`

- `graph/graph_retriever.py`

------------------------------------------------------------------------

### Task 6: Integration and Orchestration (10 points)

**Time Allocation**: 30 minutes

Integrate all components into a unified RAG system.

#### Requirements:

1.  **Unified Query Pipeline**

    - Accept user query as input

    - Apply query classification and routing

    - Execute appropriate retrieval strategy

    - Apply post-retrieval processing

    - Generate final answer using LLM

2.  **Configuration Management**

    - Externalize all configurable parameters

    - Support different modes: fast (less accurate), accurate (slower), balanced

3.  **Error Handling and Logging**

    - Graceful degradation if a component fails

    - Structured logging for debugging and monitoring

#### Deliverables:

- `main.py` or `enterprise_rag.py`

- `config.py` or `config.yaml`

- `README.md` with setup and usage instructions

------------------------------------------------------------------------

## Questions to Answer

Include written answers to these questions in your `README.md` or a separate `ANSWERS.md` file:

1.  **Architecture Decision**: Explain why you chose your specific HNSW parameters and how they balance speed vs. accuracy for this use case.

2.  **Hybrid Search Trade-offs**: Describe a scenario where Hybrid Search significantly outperforms pure Vector Search, and explain why.

3.  **Query Transformation Selection**: How does your system decide when to use HyDE vs. Query Decomposition? What signals does it look for?

4.  **Re-ranking Strategy**: Why did you choose your specific order of Cross-Encoder and MMR? What would change if the use case prioritized diversity over precision?

5.  **GraphRAG Value**: Provide an example query that your GraphRAG component can answer that would be impossible or very difficult with vector search alone.

------------------------------------------------------------------------

## Submission Rules

### Required Deliverables

Complete source code organized in the specified directory structure

`README.md` with:

- Setup instructions (dependencies, environment variables, database setup)

- Usage examples for different query types

- Architecture diagram (can be text-based)

`ANSWERS.md` with written responses to the 5 questions

`docker-compose.yml` for Neo4j and any other services

Sample queries demonstrating each component’s functionality

Screenshots or logs showing successful execution

### Submission Checklist

All code runs without errors

Semantic Chunking preserves document semantics

HNSW index is properly configured and benchmarked

Hybrid Search correctly combines BM25 and Vector results

Query Transformation handles vague and complex queries

Cross-Encoder improves ranking precision

MMR ensures result diversity

GraphRAG answers relationship queries

All components are integrated in unified pipeline

Documentation is complete and clear

------------------------------------------------------------------------

## Grading Rubrics

| Criterion | Weight | Excellent (90-100%) | Good (70-89%) | Satisfactory (50-69%) | Needs Improvement (\<50%) |
|----|----|----|----|----|----|
| **Advanced Indexing** | 20% | Semantic chunking preserves context perfectly; HNSW optimally configured with benchmarks | Chunking works with minor issues; HNSW configured but not optimized | Basic chunking implemented; HNSW uses default parameters | Chunking breaks context; HNSW not implemented |
| **Hybrid Search** | 20% | BM25 and RRF perfectly implemented; Query router makes intelligent decisions | Hybrid search works; Router has some misclassifications | Basic hybrid search; No query routing | Hybrid search not functional |
| **Query Transformation** | 15% | HyDE and Decomposition both work excellently; Smart routing between them | Both techniques work; Routing is rule-based | One technique works; No routing | Neither technique functional |
| **Post-Retrieval** | 15% | Cross-Encoder significantly improves precision; MMR provides diverse results | Both components work; Measurable improvement | One component works | Neither component functional |
| **GraphRAG** | 20% | Complete entity extraction; Rich graph; Answers complex relationship queries | Graph populated; Basic queries work | Partial graph; Limited queries | Graph not functional |
| **Integration** | 10% | Seamless pipeline; Excellent error handling; Clean configuration | Components integrated; Some rough edges | Partial integration | Components not connected |

------------------------------------------------------------------------

## Estimated Time

| Task                         | Time Allocation           |
|------------------------------|---------------------------|
| Task 1: Advanced Indexing    | 45 minutes                |
| Task 2: Hybrid Search        | 45 minutes                |
| Task 3: Query Transformation | 35 minutes                |
| Task 4: Post-Retrieval       | 35 minutes                |
| Task 5: GraphRAG             | 50 minutes                |
| Task 6: Integration          | 30 minutes                |
| **Total**                    | **240 minutes (4 hours)** |

------------------------------------------------------------------------

## Hints

**General Tips**:

- Start by setting up the infrastructure (Neo4j, Vector DB) before writing code

- Test each component independently before integration

- Use the companion notebooks from assignments as references

- Cache LLM responses during development to save API costs

**Component-Specific Tips**:

- For Semantic Chunking: Use `sentence-transformers` for efficient similarity calculation

- For HNSW: Prioritize `ef_search` tuning for query-time optimization

- For BM25: Use `nltk.word_tokenize()` for consistent tokenization

- For HyDE: The hypothetical answer doesn’t need to be factually correct

- For Cross-Encoder: Batch processing significantly improves throughput

- For GraphRAG: Test Cypher queries in Neo4j Browser before implementing in code

------------------------------------------------------------------------

## Notes:

- **You can use the your implementation in your previous assignment lab that you have done as the starting point.**
