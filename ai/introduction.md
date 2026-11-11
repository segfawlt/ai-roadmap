# Introduction

In the previous article, we built a basic Retrieval Augmented Generation (RAG) system together. We saw the power of combining a Large Language Model (LLM) with private data, from processing PDF documents, vector indexing, to asking the model to answer questions based on retrieved context.

However, the gap between theory and a real-world system is vast. When facing complex data, multi-intent questions, or strict accuracy requirements, the basic RAG architecture begins to reveal significant limitations in deep context processing capabilities, inaccuracy when retrieving specific information, and frequently causing the model to be overloaded by improperly filtered noisy data.

This article will explore advanced RAG concepts and implement experiments comparing upgraded versions of a RAG system on PDF documents.

## Glossary

| Term | Description |
|----|----|
| Hallucination | Phenomenon where the model generates false, fabricated, or non-existent information but with a confident tone. |
| Semantic Chunking | Technique of breaking text based on semantic changes instead of cutting by a fixed number of characters. |
| HNSW | Hierarchical Navigable Small World - Approximate vector search algorithm based on hierarchical graph structure, helping balance speed and accuracy. |
| Dense Retrieval | Search method based on vector embedding, focusing on semantic similarity. |
| Sparse Retrieval | Search method based on keywords, typified by the BM25 algorithm. |
| BM25 | Best Matching 25 - Document ranking algorithm based on keyword frequency statistics, an upgrade of TF-IDF with saturation and length normalization mechanisms. |
| Hybrid Search | Strategy combining results from both Dense Retrieval and Sparse Retrieval to leverage the advantages of both. |
| RRF | Reciprocal Rank Fusion - Algorithm for merging search result lists based on rank instead of score. |
| HyDE | Hypothetical Document Embeddings - Technique using LLM to generate a hypothetical answer, then using this answer’s vector to search for actual documents. |
| Bi-Encoder | Model architecture encoding question and document into two separate vectors, optimized for search speed. |
| Cross-Encoder | Model architecture processing question and document simultaneously to score relevance, having high accuracy but slow speed, often used for Re-ranking. |
| MMR | Maximal Marginal Relevance - Document selection algorithm aimed at balancing relevance and information diversity. |

## Basic RAG Architecture

Before exploring advanced techniques, we need to review the standard RAG architecture built in the previous article. Clearly understanding this workflow is a crucial prerequisite for accurately identifying the system’s bottlenecks needing optimization.

``` mermaid
        graph LR
    subgraph "Indexing"
        D["Documents<br>(PDF, Docx, Text, MD)"] --> CK[Chunking]
        CK --> EM[Embedding Model]
        EM --> VDB[(Vector Database<br>+ Metadata)]
    end
    subgraph "Retrieval"
        U([User]) --> Q[Question]
        Q --> EM2[Embedding Model]
        EM2 --> QV["Query Vector (q)"]
        QV --> SS[Similarity Search]
        VDB --> SS
        SS --> TK[TopK Chunks]
    end
    subgraph "Generation"
        TK --> PT[Prompt<br>= Context + Question]
        PT --> LLM[LLM]
        LLM --> ANS[Answer]
    end
```

*Figure 1: Diagram of data flow and components in basic RAG architecture.*

Retrieval-Augmented Generation (RAG) is an architectural solution aiming to overcome two inherent limitations of LLMs: hallucinations and the lack of updated, private knowledge.

Instead of relying entirely on parametric memory fixed after training, RAG equips LLMs with the ability to look up real-time information from external memories. A complete RAG pipeline operates based on the coordination of 3 main phases as shown in Figure 1:

## 1. Indexing

This is the data preparation phase. The goal is to convert raw knowledge into a representation that computers can search effectively.

- **Input**: Diverse raw data (PDF, Docx, Text, Markdown…).

- **Activity**:

  - **Load & Extract**: Extract pure text from file formats.

  - **Chunking**: Break text into segments called chunks, with lengths suitable for the model’s context window.

  - **Embedding**: Use an embedding model to encode chunks into real number vectors in n-dimensional semantic space.

- **Output**: Vector database storing vectors along with corresponding metadata.

## 2. Retrieval

This is the real-time processing phase, triggered when the user sends a question.

- **Activity**:

  1.  The question is passed through the same embedding model to create a query vector (`\(q\)`).

  2.  The system performs similarity search between vector `\(q\)` and all vectors in the database.

- **Output**: TopK chunks with the highest similarity, expected to contain information answering the question.

## 3. Generation

The final synthesis phase, where the linguistic power of the LLM is leveraged.

- **Activity**: Chunks found in the Retrieval step are concatenated to form context along with the original question put into a prompt following a predefined structure.

- **Output**: The LLM processes the prompt, synthesizes information from the context, and generates a natural, accurate, and factually grounded answer.
