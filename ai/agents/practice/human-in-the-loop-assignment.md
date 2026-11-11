# Assignment: Human-in-the-Loop & Persistence

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Human-in-the-Loop & Persistence |
| **Course** | LangGraph and Agentic AI |
| **Project Name** | `hitl-persistence-agent` |
| **Estimated Time** | 150 minutes |
| **Framework** | Python 3.11+, LangGraph 1.x, LangChain 1.x, SQLite/PostgreSQL, FAISS |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Implement** interrupt patterns for human approval workflows

- **Configure** different checkpointer backends (InMemorySaver, SQLiteSaver, PostgresSaver)

- **Design** confirmation UIs for critical operations

- **Build** cache tools using vector stores for response caching

- **Apply** state update patterns for manual corrections

------------------------------------------------------------------------

## Problem Description

Extend the FPT Customer Chatbot from Assignment 04 with:

1.  **Human-in-the-Loop Confirmation**: Require user approval before ticket/booking operations

2.  **Response Caching**: Store RAG/IT support responses in FAISS for follow-up queries

3.  **Persistent State**: Enable long-running conversations with SQLite checkpointer

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langgraph` \>= 1.0

  - `langchain[anthropic]` \>= 1.0

  - `faiss-cpu` \>= 1.7.0

  - `sentence-transformers` \>= 2.2.0

### Prerequisite

- Completed Assignment 04 (FPT Multi-Agent Chatbot)

------------------------------------------------------------------------

## Tasks

### Task 1: Interrupt Before Tool Execution (35 points)

1.  **Configure interrupt_before** for sensitive tools:

    - All ticket creation/update/cancel operations

    - All booking creation/update/cancel operations

2.  **Implement confirmation flow**:

    - Detect pending tool state via `graph.get_state()`

    - Generate human-readable confirmation message

    - Parse user response (“y” to continue, other to cancel)

3.  **Handle user responses**:

    - “y”: Resume graph execution with `app.invoke(None,`` ``config)`

    - Other: Update state to cancel and return appropriate message

4.  **Create confirmation message generator** that:

    - Extracts tool name and arguments from pending state

    - Formats readable summary for user review

    - Includes clear instructions for approval/rejection

### Task 2: Cache Tool Implementation (35 points)

1.  **Create cache_tool** that:

    - Stores all RAG and IT Support responses in FAISS vectorstore

    - Indexes by query embedding for similarity search

    - Stores metadata: timestamp, query_type, source_agent

2.  **Implement cache lookup** in orchestrator:

    - Before calling RAG/IT tools, check cache for similar queries

    - Use similarity threshold (e.g., 0.85) to determine cache hit

    - Return cached response if found, otherwise proceed to tool

3.  **Add cache management**:

    - TTL-based invalidation (e.g., 24 hours)

    - Manual cache clear capability

    - Cache statistics logging

### Task 3: Checkpointer Configuration (20 points)

1.  **Replace InMemorySaver with SQLiteSaver**:

    - Configure persistent storage in `checkpoints.db`

    - Test conversation resumption after process restart

2.  **Implement thread management**:

    - List active threads

    - View checkpoint history for a thread

    - Delete old threads (cleanup)

3.  **Document migration path** to PostgresSaver for production

### Task 4: Testing & Validation (10 points)

1.  **Test interrupt workflow**:

    - Create ticket → Confirm “y” → Verify ticket created

    - Create booking → Reject with “n” → Verify booking NOT created

2.  **Test cache functionality**:

    - Query IT support → Verify cached

    - Similar follow-up query → Verify cache hit

    - After TTL → Verify cache miss

3.  **Test persistence**:

    - Start conversation → Stop process → Resume → Verify context retained

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code extending Assignment 04

`README.md` with setup instructions

Demo video or screenshots showing:

- Interrupt confirmation workflow

- Cache hit/miss scenarios

- Conversation persistence across restarts

### Submission Checklist

interrupt_before configured for all sensitive tools

Confirmation message clearly shows pending action

Cache tool stores and retrieves responses correctly

SQLiteSaver enables conversation persistence

Code runs without errors

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                          | Points  |
|-----------------------------------|---------|
| Interrupt workflow implementation | 35      |
| Cache tool with FAISS             | 35      |
| Checkpointer configuration        | 20      |
| Testing & validation              | 5       |
| Code quality and documentation    | 5       |
| **Total**                         | **100** |

------------------------------------------------------------------------

## Hints

- Use `app.get_state(config)` to inspect current state including pending tool calls

- The `next` field in state shows which node(s) are pending

- For FAISS caching, use `sentence-transformers` for consistent embeddings

- SQLiteSaver requires context manager: `with`` ``SqliteSaver.from_conn_string(...)`` ``as`` ``saver:`

------------------------------------------------------------------------

## References

- [LangGraph Human-in-the-Loop](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/wait-user-input/)

- [LangGraph Checkpointers](https://langchain-ai.github.io/langgraph/how-tos/persistence/)

- [FAISS Documentation](https://github.com/facebookresearch/faiss)
