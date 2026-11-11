# Building RAG Agent using LangChain

Reference Notebook: [0_create_agent.ipynb](https://github.com/langchain-ai/deep-agents-from-scratch/blob/main/notebooks/0_create_agent.ipynb)

In this lesson, we will focus on initializing a **ReAct (Reason + Act) Agent**. This is a model that helps LLM not only think but also be capable of performing specific actions through tools:

- **Operating Mechanism:** Agent performs a loop: Reasoning -\> Action -\> Observation.

- **Flexibility:** This is an important foundation for building complex Agent systems (Deep Agents), allowing smart processing of multi-step tasks.

------------------------------------------------------------------------

## Practice Project: RAG Agent System for FPT Policy

**Goal:** Build an automated question-answering system about FPT’s internal regulations and policies.

### Implementation Process:

1.  **Input Data:** Gather FPT’s policy documents, internal guide documents.

2.  **Vector Database:** Convert text data into embeddings and store in VectorDB for retrieval.

3.  **Retrieval System:** Use semantic search mechanism to retrieve most relevant text segments (Context).

4.  **Integrate ReAct Agent:**

- Agent will receive questions from users.

- Use `retriever` tool to search information in VectorDB.

- Synthesize information and answer based on actual data from company policy.

> **Processing Flow (Pipeline):** \> `FPT`` ``internal`` ``docs` → `Embeddings` → `VectorDB` → `ReAct`` ``Agent`` ``(Retrieval`` ``Tool)` → `Final`` ``Answer`
