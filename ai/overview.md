# AI Training Overview

This overview covers all four AI training sub-domains: Foundations, RAG Optimization (Applied), LangGraph and Agentic AI, and LLMOps and Evaluation (Advanced).

------------------------------------------------------------------------

## AI Foundations

**Topic Code**: BAI_01 \| **Version**: 1.0 \| **Audience**: Freshers / Interns with basic Python knowledge

### Objectives

Introduces fundamental concepts of AI and Generative AI, with a deep dive into Large Language Models (LLMs) and Retrieval-Augmented Generation (RAG). Trainees learn the theoretical foundations of RAG, modern RAG architectures, and how to implement them using the LangChain framework. The course culminates in building a practical RAG agent to answer questions based on internal policy documents.

### Learning Outcomes

- Define Artificial Intelligence, Generative AI, and Large Language Models (LLMs)

- Identify key capabilities and limitations of LLMs (hallucinations, knowledge cutoff)

- Explain RAG and the difference between parametric and non-parametric memory

- Describe the Indexing Phase: document loading, chunking strategies, and embedding

- Explain retrieval techniques: Vector Search, Hybrid Search, and Re-ranking

- Understand generation strategies: context stuffing, compression, and prompt engineering

- Utilize LangChain Document Loaders, Text Splitters, Embeddings, and Vector Stores

- Build a complete RAG pipeline and develop a ReAct Agent using LangChain

### Prerequisites

- Python 3.11+

- Basic programming knowledge

### Schedule

| Unit | Content                               | Sessions |
|------|---------------------------------------|----------|
| 1    | Introduction to AI & Generative AI    | 2        |
| 2    | RAG Theoretical Foundations           | 2        |
| 3    | Modern RAG Architecture               | 4        |
| 4    | LangChain Framework & Core Components | 2        |
| 5    | Building RAG Agent Using LangChain    | 3        |

### Assessment

| Component       | Weight |
|-----------------|--------|
| Quiz (5)        | 20%    |
| Assignments (2) | 30%    |
| Final Project   | 50%    |

**Pass Criteria**: Total GPA \>= 70/100, 100% quizzes and assignments completed, successful RAG Agent demo.

------------------------------------------------------------------------

## RAG Optimization (Applied)

**Version**: 1.0 \| **Audience**: Freshers familiar with basic RAG

### Objectives

Introduces advanced techniques and optimizations for RAG systems, equipping trainees with skills specific to AI applications, enterprise vector search, and GraphRAG knowledge bases.

### Learning Outcomes

- Apply Semantic Chunking vs Recursive Chunking

- Optimize vector databases using the HNSW Index (M, ef_construction, ef_search)

- Leverage BM25 alongside Vector Search and merge results using Reciprocal Rank Fusion (RRF)

- Implement Hypothetical Document Embeddings (HyDE)

- Perform Query Decomposition for complex multi-intent queries

- Utilize Cross-Encoder for enhanced re-ranking vs Bi-Encoder approaches

- Apply Maximal Marginal Relevance (MMR) for diverse retrieval

- Design Neo4j graph schemas, extract entities, and maintain a knowledge graph

- Execute Graph Cypher QA Chains with Neo4j

### Prerequisites

- Completed AI Foundations module

- Familiarity with basic RAG systems

### Schedule

| Unit | Content                                           | Sessions |
|------|---------------------------------------------------|----------|
| 1    | Advanced Indexing (Semantic Chunking & HNSW)      | 1        |
| 2    | Hybrid Search (Vector + BM25, RRF)                | 1        |
| 1-2  | Lab: Indexing & Hybrid Search Setup               | 2        |
| 3    | Query Transformation (HyDE & Query Decomposition) | 1        |
| 4    | Post-Retrieval Processing (Cross-Encoder, MMR)    | 1        |
| 3-4  | Lab: Transformation & Re-ranking                  | 2        |
| 5    | GraphRAG Implementation                           | 2        |

### Assessment

| Component           | Weight |
|---------------------|--------|
| Quiz (5)            | 20%    |
| Assignments (2)     | 30%    |
| Final Practice Test | 50%    |

**Pass Criteria**: Total GPA \>= 7.0/10, 100% videos, quizzes, assignments, labs, and final test completed.

------------------------------------------------------------------------

## LangGraph and Agentic AI (Advanced)

**Version**: 1.0 \| **Audience**: Freshers / Interns with basic Python knowledge who have passed prior AI modules

### Objectives

Deep dives into building advanced agentic AI systems using the LangGraph framework, moving beyond linear chains to cyclic, stateful workflows. Trainees master core LangGraph concepts such as State Management, Nodes, and Edges, while implementing advanced patterns like ReAct, Planning, and Multi-Expert orchestration. Training covers Tavily Search integration, multi-agent collaboration strategies, and Human-in-the-Loop mechanisms with persistent state checkpointing.

### Learning Outcomes

- **LangGraph Foundations**: Understand LangGraph architecture, messages-centric state management, and build cyclic workflows with Nodes and Edges

- **Agentic Patterns**: Implement ReAct agents, use LangGraph’s prebuilt ToolNode, apply reflection and planning techniques

- **Tool Calling**: Understand Tool/Function Calling in LLMs, integrate Tavily Search API, manage parallel tool execution

- **Multi-Agent Collaboration**: Design sequential and hierarchical multi-agent architectures, coordinate agent workflows

- **Human-in-the-Loop**: Implement human approval workflows, persist agent state using checkpointers, apply time-travel and state editing

### Prerequisites

- Passed previous AI modules

- Python 3.11+

- Anthropic API Key (or OpenAI API Key) and Tavily API Key

### Schedule

| Unit | Content                                       | Sessions |
|------|-----------------------------------------------|----------|
| 1    | LangGraph Foundations & State Management      | 1        |
| 2    | Agentic Patterns: Multi-Expert Research Agent | 3        |
| 3    | Tool Calling & Tavily Search                  | 3        |
| 4    | Multi-Agent Collaboration                     | 3        |
| 5    | Human-in-the-Loop & Persistence               | 3        |

### Assessment

| Component                          | Weight |
|------------------------------------|--------|
| Quiz                               | TBD    |
| Assignments                        | TBD    |
| Final Project (Multi-Agent System) | TBD    |

**Pass Criteria**: Total GPA \>= 60/100, 100% assignments and final project completed.

------------------------------------------------------------------------

## LLMOps and Evaluation (Advanced)

**Version**: 1.0 \| **Audience**: Freshers familiar with basic RAG

### Objectives

Introduces LLMOps and Evaluation knowledge, equipping trainees with skills specific to evaluating RAG systems, implementing observability tools like LangFuse and LangSmith, and designing deep architectural experiments.

### Learning Outcomes

- Comprehend and apply Ragas framework metrics for automated LLM evaluation

- Calculate Faithfulness and Answer Relevancy for generation quality

- Assess Context Precision and Context Recall for retrieval performance

- Set up and integrate LangFuse for open-source tracing and prompt management

- Configure LangSmith for native LangChain execution tracing and playground debugging

- Implement best practices for production LLMOps (sampling, PII, alerts)

- Design an experimental infrastructure for comparing different RAG architectures

- Analyze trade-offs (quality, cost, latency) across Naive, Advanced, Graph, and Hybrid setups

### Prerequisites

- Completed AI Foundations and RAG Optimization modules

- Familiarity with basic RAG systems

### Schedule

| Unit | Content                                     | Sessions |
|------|---------------------------------------------|----------|
| 1    | Ragas Evaluation Metrics                    | 3        |
| 2    | Observability: LangFuse & LangSmith         | 4        |
| 3    | Experiment Comparison: Naive, Graph, Hybrid | 3        |

### Assessment

| Component           | Weight |
|---------------------|--------|
| Quiz (5)            | 20%    |
| Assignments (2)     | 30%    |
| Final Practice Test | 50%    |

**Pass Criteria**: Total GPA \>= 7.0/10, 100% videos, quizzes, assignments, labs, and final test completed. Successful demo of a RAG system with evaluation and tracing.
