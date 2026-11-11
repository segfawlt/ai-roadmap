# Quiz & Appendix - Advanced

## Knowledge Summary

### Section 1: Optimization

#### Skills Learned

- Advanced Indexing (Semantic Chunking, HNSW, Multi-Vector)

- Hybrid Search (Vector + BM25 + RRF)

- Query Transformation (HyDE, Decomposition)

- Post-Retrieval (Reranking, MMR)

#### Key Takeaways

- HNSW dramatically improves search speed

- Reranking adds precision, MMR adds diversity

### Section 2: GraphRAG

#### Skills Learned

- Graph Database concepts (Nodes, Edges)

- Knowledge Graph construction

- GraphRAG architecture

- Entity/Relation extraction

#### Key Takeaways

- Entity extraction is critical

### Section 3: LangGraph & Agentic AI

#### Skills Learned

- LangGraph foundations (State, Nodes, Edges)

- Agentic patterns (Reflection, Planning)

- Tool calling (Tavily Search)

- Multi-agent collaboration

- Human-in-the-loop

- Streaming with FastAPI

#### Key Takeaways

- State management is core

- Reflection improves quality

- Planning handles complexity

- Agents extend LLM capabilities

### Section 4: LLMOps

#### Skills Learned

- RAGAS evaluation metrics

- LangFuse & LangSmith observability

- Experimental comparison

- Production best practices

#### Key Takeaways

- Evaluation is essential

- Observability enables debugging

- Each architecture has trade-offs

## Multiple Choice Questions

### Part 1: Optimization

#### **Question 1:** How does semantic chunking differ from fixed-size chunking?

- A\) Splits based on character count

- B\) Splits based on semantic meaning

- C\) Splits randomly

- D\) No difference

#### **Question 2:** What does the ‘M’ parameter in HNSW indexing represent?

- A\) Number of vectors in index

- B\) Number of connections per node

- C\) Embedding dimension

- D\) Number of layers

#### **Question 3:** What does Multi-Vector Retrieval store?

- A\) Only original documents

- B\) Multiple embeddings for each document (summary, questions, etc.)

- C\) Multiple copies of the same embedding

- D\) Only queries

#### **Question 4:** What is BM25?

- A\) Vector similarity algorithm

- B\) Keyword-based ranking algorithm

- C\) Embedding model

- D\) LLM model

#### **Question 5:** What is RRF (Reciprocal Rank Fusion) used for?

- A\) Embed documents

- B\) Generate answers

- C\) Merge rankings from multiple retrievers

- D\) Chunk documents

#### **Question 6:** What is the core principle of HyDE (Hypothetical Document Embeddings)?

- A\) Translating natural language to SQL

- B\) Generating a fake hypothetical answer, then embedding it for search

- C\) Breaking a complex query into sub-questions

- D\) Using keyword search only

#### **Question 7:** When is Query Decomposition most effective?

- A\) For simple, single-fact lookups

- B\) When the query asks about multiple distinct topics or requires multi-step reasoning

- C\) When the user provides keywords only

- D\) When using an image encoder

#### **Question 8:** What is a potential risk of using HyDE?

- A\) It is too fast

- B\) It requires a knowledge graph

- C\) Hallucinated hypothetical answers might lead retrieval astray

- D\) It cannot handle English text

#### **Question 9:** Which technique involves generating multiple variations of a user query to improve recall?

- A\) Multi-Query Retrieval / Expansion

- B\) Vector Quantization

- C\) Semantic Chunking

- D\) Graph Traversal

#### **Question 10:** What is “Step-Back Prompting” in the context of Query Transformation?

- A\) Asking the user to simplify the question

- B\) Generating a more abstract, high-level question to retrieve broad context first

- C\) Ignoring the prompt entirely

- D\) Reversing the order of words in the query

#### **Question 11:** How does a Cross-Encoder differ from a Bi-Encoder?

- A\) Cross-Encoder separates query and document

- B\) Cross-Encoder processes query and document together (Joint extraction)

- C\) Bi-Encoder is slower than Cross-Encoder

- D\) There is no difference

#### **Question 16:** What does the Lambda parameter in MMR control?

- A\) The number of chunks retrieved

- B\) The embedding dimension

- C\) The balance between Relevance vs Diversity

- D\) The learning rate of the model

**View Part 1 Answer Key**

- **Q1**: **B**

- **Q2**: **B**

- **Q3**: **B**

- **Q4**: **B**

- **Q5**: **C**

- **Q6**: **B** - HyDE bridges the gap between question vectors and answer vectors.

- **Q7**: **B** - Complex questions often need to be solved in parallel or sequence.

- **Q8**: **C** - If the LLM hallucinates facts in the HyDE document, the vector search will look for those incorrect facts.

- **Q9**: **A** - Different phrasings capture different semantic angles of the same intent.

- **Q10**: **B** - Abstraction helps retrieve foundational knowledge that specific details might miss.

- **Q11**: **B** - Cross-Encoder captures deep interaction but is slower; Bi-Encoder is fast but less precise.

- **Q16**: **C** - The trade-off between relevance (λ=1) and diversity (λ=0).

### Part 2: GraphRAG

#### **Question 17:** In a graph database, what do Nodes and Edges represent?

- A\) Nodes = data, Edges = code

- B\) Nodes = entities, Edges = relationships

- C\) Nodes = queries, Edges = answers

- D\) No difference

#### **Question 18:** What is Cypher?

- A\) Encryption algorithm

- B\) Query language for Neo4j

- C\) Embedding model

- D\) Python library

#### **Question 19:** What role does “Community Detection” (e.g., Leiden algorithm) play in GraphRAG?

- A\) It finds users with similar IP addresses

- B\) It groups related nodes to generate high-level summaries/themes

- C\) It detects spam emails

- D\) It encrypts the database

#### **Question 20:** What is the primary difficulty in “Entity Extraction” for GraphRAG?

- A\) Text is too short

- B\) LLMs usually refuse to extract entities

- C\) Entity resolution / disambiguation (e.g., “Apple” vs “Apple Inc.”)

- D\) Storing vectors is expensive

#### **Question 21:** Which GraphRAG capability allows finding connections between indirect neighbors (A-\>B-\>C)?

- A\) Vector Similarity

- B\) Semantic Chunking

- C\) Multi-hop Traversal

- D\) Keyword Matching

#### **Question 22:** When is GraphRAG better than standard RAG?

- A\) For simple fact lookups

- B\) When strict low latency is required

- C\) For complex relational queries and multi-hop reasoning

- D\) When costs must be minimized

**View Part 2 Answer Key**

- **Q17**: **B**

- **Q18**: **B**

- **Q19**: **B** - Community detection identifies clusters of concepts to answer “global” questions.

- **Q20**: **C** - Ensuring the graph is connected correctly requires resolving entity duplicates.

- **Q21**: **C** - Traversal follows edges to find distant connections.

- **Q22**: **C** - When there are many relational queries and multi-hop reasoning requirements.

### Part 3: LangGraph & Agents

#### **Question 23:** What is State in LangGraph?

- A\) Database state

- B\) Shared data between nodes

- C\) API state

- D\) UI state

#### **Question 24:** What are conditional edges used for?

- A\) Routing based on state

- B\) Optimize performance

- C\) Save memory

- D\) Log data

#### **Question 25:** What does Andrew Ng’s Reflection pattern do?

- A\) Speed up inference

- B\) Self-evaluate and iteratively improve output

- C\) Reduce cost

- D\) Compress data

#### **Question 26:** What does tool calling allow an LLM to do?

- A\) Train faster

- B\) Access external data and perform actions

- C\) Generate images

- D\) Compress text

**View Part 3 Answer Key**

- **Q23**: **B**

- **Q24**: **A**

- **Q25**: **B**

- **Q26**: **B**

### Part 4: LLMOps

#### **Question 27:** What does the RAGAS metric “Faithfulness” measure?

- A\) Answer speed

- B\) Answer grounded in context (no hallucinations)

- C\) Answer length

- D\) Cost

#### **Question 28:** What is the difference between Context Precision and Recall?

- A\) Precision is speed; Recall is accuracy

- B\) Precision is relevant/retrieved; Recall is relevant/total_relevant

- C\) Precision is quantity; Recall is quality

- D\) Precision is for vectors; Recall is for graphs

#### **Question 29:** Which RAGAS metric measures if the generated answer actually addresses the user’s question?

- A\) Faithfulness

- B\) Answer Relevancy

- C\) Context Recall

- D\) Latency

#### **Question 30:** What is the main difference between LangFuse and LangSmith?

- A\) LangFuse is open-source/self-hosted; LangSmith is a managed SaaS with deep LangChain integration

- B\) LangFuse is for images; LangSmith is for text

- C\) LangFuse is paid; LangSmith is free

- D\) LangSmith is only for OpenAI models

#### **Question 31:** In Observability, what does “Tracing” refer to?

- A\) Drawing graphs

- B\) Tracking the step-by-step execution flow of chains/agents

- C\) Finding the IP address of the user

- D\) Copying data to a backup

#### **Question 32:** Why is “Sampling” important in Production Observability?

- A\) To save storage costs and reduce noise by not logging 100% of traces

- B\) To test new models

- C\) To improve latency

- D\) To satisfy GDPR

#### **Question 33:** Which tool provides an “Edit and Re-run” playground for debugging failed traces?

- A\) LangFuse

- B\) LangSmith

- C\) Prometheus

- D\) Grafana

#### **Question 34:** When should you use Hybrid RAG (Graph + Advanced)?

- A\) When you have no budget

- B\) When latency is the top priority

- C\) When you need the absolute best quality for mixed query types

- D\) When you only have unstructured text with no entities

**View Part 4 Answer Key**

- **Q27**: **B**

- **Q28**: **B** - Precision: Relevant/Retrieved (Noise level); Recall: Relevant/Total (Coverage).

- **Q29**: **B** - Answer Relevancy measures alignment with the query, not truthfulness.

- **Q30**: **A** - LangFuse is OS friendly; LangSmith is SaaS.

- **Q31**: **B** - Tracing visualizes the black box execution.

- **Q32**: **A** - Monitoring every single request in high-traffic apps is expensive.

- **Q33**: **B** - LangSmith’s deep integration allows re-running traces with modified prompts.

- **Q34**: **C** - When best quality is needed, there are mixed query types, and budget is not an issue.

## Appendix

### Graph Databases

- Neo4j: <https://neo4j.com>

### Evaluation

- RAGAS: [vibrantlabsai/ragas](https://github.com/vibrantlabsai/ragas)

### Observability

- LangFuse: <https://langfuse.com>

- LangSmith: <https://smith.langchain.com>

### Documentation Links

- LangChain: <https://docs.langchain.com/oss/python/langchain/overview>

- LangGraph: <https://langchain-ai.github.io/langgraph>

- RAGAS: <https://docs.ragas.io/en/stable/>

- Neo4j Cypher: <https://neo4j.com/docs/cypher-manual/current/introduction/>

### GitHub Repositories

#### Example Projects

- LangGraph Examples: [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph/tree/main/examples)

- GraphRAG: [microsoft/graphrag](https://github.com/microsoft/graphrag)

- RAG Techniques: [NirDiamant/RAG_Techniques](https://github.com/NirDiamant/RAG_Techniques)
