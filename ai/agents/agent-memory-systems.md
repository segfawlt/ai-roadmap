# Agent Memory Systems NEW

As agents tackle longer, more complex tasks, managing memory becomes critical. An agent without proper memory loses context, repeats mistakes, and cannot learn from experience. This page covers the taxonomy of agent memory, implementation patterns, and production tooling for building persistent, context-aware agents.

## Learning Objectives

- Understand the taxonomy of agent memory types

- Distinguish working memory, semantic memory, episodic memory, and procedural memory

- Implement memory patterns in LangGraph (checkpoints and cross-thread stores)

- Know production memory tools (Mem0, LangGraph Store)

- Recognize and avoid common memory anti-patterns

## 1. Why Memory Matters

Without memory, agents are “goldfish” — each interaction starts fresh. Every conversation turn is isolated, every mistake is repeated, and every user preference must be restated. Memory enables:

- **Continuity** across conversation turns within a session

- **Learning** from past interactions and errors

- **Personalization** based on user history and preferences

- **Long-running task resumption** after interruptions or failures

The result is an agent that feels less like a stateless API and more like a knowledgeable colleague.

## 2. Memory Taxonomy

Agent memory is not monolithic. Different types of memory serve different purposes and require different storage strategies.

``` mermaid
        graph TD
    M[Agent Memory] --> W[Working Memory\nToken-level / Context Window]
    M --> S[Semantic Memory\nLong-term Facts]
    M --> E[Episodic Memory\nPast Experiences]
    M --> P[Procedural Memory\nHow-To / Workflows]

    W --> W1[In-context messages]
    W --> W2[Managed via context engineering]

    S --> S1[Vector database]
    S --> S2[Key-value store]

    E --> E1[Interaction logs]
    E --> E2[Success/failure records]

    P --> P1[Stored workflows]
    P --> P2[Learned skills]
```

### 2.1 Token-Level (Working Memory)

The context window itself. Everything the model can “see” right now.

- **Scope**: Current conversation turn and recent history

- **Limit**: 8K–1M tokens depending on the model

- **Lifetime**: Lost when the conversation ends (unless persisted via checkpointing)

- **Managed by**: Context engineering — summarization, truncation, selective inclusion

Working memory is the foundation, but it is finite. All other memory types exist to extend what the agent effectively knows beyond the context window.

### 2.2 Semantic Memory (Long-Term Facts)

Persistent knowledge stored in vector databases or key-value stores.

- **Contains**: User preferences, learned facts, domain knowledge, entity relationships

- **Survives**: Across sessions and restarts

- **Retrieved by**: Semantic similarity search or direct key lookup

- **Examples**: “User prefers concise summaries”, “Company X uses SAP ERP”

Semantic memory allows an agent to accumulate knowledge over time without stuffing facts into every prompt.

### 2.3 Episodic Memory (Past Experiences)

Records of past interactions — what happened, what worked, what failed.

- **Contains**: Conversation summaries, action logs, task outcomes, error records

- **Survives**: Across sessions

- **Retrieved by**: Temporal lookup or semantic similarity

- **Examples**: “Last week the user asked about Q3 variance; we flagged a data quality issue”

Episodic memory enables “lessons learned” behavior. An agent that remembers a past failure can avoid repeating it.

### 2.4 Procedural Memory (How-To)

Stored workflows and procedures the agent has learned or been given.

- **Contains**: Step-by-step workflows, conditional decision trees, tool usage patterns

- **Survives**: Indefinitely; versioned

- **Retrieved by**: Task classification and intent matching

- **Examples**: “When the user requests a financial summary, fetch GL data → reconcile → format → attach”

Procedural memory allows agents to acquire and refine skills over time, turning ad-hoc task completion into repeatable, optimized processes.

## 3. Implementation Patterns

### 3.1 LangGraph Thread Checkpoints (Working Memory Persistence)

LangGraph checkpoints persist conversation state across turns within a thread. Each thread maintains its own isolated state, enabling multi-user deployments without memory bleed.

``` python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

# Each thread has its own conversation state
async with AsyncPostgresSaver.from_conn_string(db_url) as saver:
    graph = workflow.compile(checkpointer=saver)

    config = {"configurable": {"thread_id": "user-123-session-456"}}

    # State persists across turns within the same thread_id
    result = await graph.ainvoke(
        {"messages": [HumanMessage(content="What did we discuss last time?")]},
        config=config,
    )
```

Use thread checkpoints when you need turn-by-turn continuity within a single conversation. The thread ID acts as the conversation identifier — different thread IDs are fully isolated.

### 3.2 LangGraph Store (Cross-Thread Memory)

LangGraph’s store provides a shared namespace that persists across threads and sessions. It is the right layer for user preferences, long-term facts, and anything that should outlive a single conversation.

``` python
from langgraph.store.memory import InMemoryStore
from langgraph.store.postgres import AsyncPostgresStore

# Use InMemoryStore for development, AsyncPostgresStore for production
store = InMemoryStore()
graph = workflow.compile(checkpointer=saver, store=store)

# Inside a node — store.put writes to the shared namespace
def remember_preference(state, config, *, store):
    user_id = config["configurable"]["user_id"]
    store.put(
        ("user", user_id),          # namespace: tuple acting as a path
        "preference",               # key
        {"language": "en", "format": "concise"},  # value
    )

# Inside another node — store.get reads back
def apply_preference(state, config, *, store):
    user_id = config["configurable"]["user_id"]
    pref = store.get(("user", user_id), "preference")
    language = pref.value.get("language", "en") if pref else "en"
    return {"language": language}
```

Namespaces are tuples that create logical separation (e.g., `("user",`` ``user_id)` vs `("project",`` ``project_id)`). This prevents key collisions across different entity types.

### 3.3 Vector Store Memory (Semantic Retrieval)

For large-scale semantic memory, store memories as embeddings and retrieve by similarity. This scales to millions of memories per user without overwhelming the context window.

``` python
from langchain_core.documents import Document
from langchain_openai import OpenAIEmbeddings
from langchain_postgres import PGVector

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vector_store = PGVector(
    embeddings=embeddings,
    collection_name="agent_memories",
    connection=db_url,
)

# Save a memory
memory_text = "User prefers Vietnamese language for audit report summaries"
vector_store.add_documents([
    Document(
        page_content=memory_text,
        metadata={
            "user_id": "123",
            "type": "preference",
            "created_at": "2026-04-15",
        },
    )
])

# Recall relevant memories for the current task
memories = vector_store.similarity_search(
    "language preference for reports",
    k=5,
    filter={"user_id": "123"},
)

# Inject memories into agent context
memory_context = "\n".join(m.page_content for m in memories)
```

The key discipline here is **selective storage** — save only surprising, durable, or high-value information. Storing every turn degrades retrieval quality and inflates costs.

### 3.4 Episodic Memory with Summarization

Rather than storing raw transcripts, summarize interactions before saving. This compresses episodic memory and makes retrieval more effective.

``` python
async def save_episode(thread_id: str, messages: list, outcome: str, store):
    """Summarize and save a completed interaction as an episode."""
    summary_prompt = f"""
    Summarize this agent interaction in 2-3 sentences.
    Focus on: what was asked, what actions were taken, and what the outcome was.
    Note any errors, user corrections, or lessons learned.

    Outcome: {outcome}
    """
    summary = await llm.ainvoke(messages + [HumanMessage(content=summary_prompt)])

    store.put(
        ("episodes", thread_id),
        f"episode_{int(time.time())}",
        {
            "summary": summary.content,
            "outcome": outcome,
            "timestamp": time.time(),
        },
    )
```

## 4. Mem0: Production Memory Platform

Mem0 is a dedicated memory layer for AI applications, published at ECAI 2025. It abstracts the complexity of memory extraction, storage, and retrieval into a single SDK.

Key capabilities:

- **Automatic extraction**: Identifies memorable facts from raw conversation text without manual annotation

- **Semantic deduplication**: Updates existing memories instead of creating duplicates

- **Conflict resolution**: When new information contradicts a stored memory, Mem0 applies an update-or-replace strategy

- **Graph memory**: Models relationships between entities (experimental, moving to production in 2026)

- **Procedural memory**: Stores and retrieves learned workflows (v1.0.0, 2026)

``` python
from mem0 import Memory

m = Memory()

# Save memories from a conversation — extraction is automatic
messages = [
    {"role": "user", "content": "I prefer summaries in bullet points, not paragraphs."},
    {"role": "assistant", "content": "Noted! I will use bullet points going forward."},
]
m.add(messages, user_id="user-123")

# Retrieve relevant memories before responding
memories = m.search("formatting preference", user_id="user-123")
# Returns: [{"memory": "User prefers bullet point summaries", "score": 0.92}]
```

Mem0 is appropriate when you want managed memory with minimal implementation overhead. For fine-grained control or custom storage backends, use LangGraph Store directly.

## 5. Memory Anti-Patterns

| Anti-Pattern | Problem | Fix |
|----|----|----|
| Storing everything | Context pollution and high retrieval costs | Store only surprising, durable, or high-value information |
| No memory decay | Stale memories override current user state | Add TTL fields or confidence scores that decay over time |
| No conflict resolution | Contradictory memories confuse agent responses | Use update-or-replace, never blind append |
| Mixing memory types | Working memory fills with long-term facts | Separate stores: checkpoints for in-session, store/vector for long-term |
| Flat memory namespace | Key collisions across users or projects | Use structured namespaces: `("user",`` ``uid)`, `("project",`` ``pid)` |
| No retrieval filter | Memory from other users leaks into responses | Always filter by `user_id` or equivalent scope key |

## 6. Memory in Production

Building production memory systems requires decisions beyond the code layer.

### Privacy and Compliance

- **GDPR right to deletion**: AI memories about a person are personal data. Implement a `delete_user_memories(user_id)` path from day one.

- **Data minimization**: Only store what is necessary for the agent’s purpose. Audit stored memory periodically.

- **Consent**: Be transparent with users about what is being remembered and for how long.

### Staleness and Decay

Memories become outdated. A preference recorded six months ago may no longer reflect the user’s current needs.

Mitigation strategies:

- Add `created_at` and `last_accessed` timestamps to every memory record

- Apply confidence decay: reduce relevance score of memories older than a threshold

- Prompt the agent to verify critical memories before acting on them

### Scale and Index Design

At production scale, memory retrieval latency matters.

- Use approximate nearest neighbor (ANN) indexes (e.g., HNSW in pgvector) for vector retrieval

- Partition by `user_id` to keep indexes small and queries fast

- Cache frequently accessed memories (user preferences, static facts) in a fast key-value layer (Redis/Valkey)

### Evaluation

Memory quality is hard to measure. Useful metrics:

| Metric | Description |
|----|----|
| Recall@K | Does the correct memory appear in the top-K retrieved results? |
| Precision@K | What fraction of the top-K retrieved memories are actually relevant? |
| Task improvement rate | Does memory usage measurably improve task success rate? |
| Memory freshness | What percentage of stored memories are within their confidence threshold? |

Evaluate memory independently from the agent’s reasoning. A memory system that retrieves incorrect or outdated facts will degrade agent quality even if the reasoning itself is sound.

## Summary

``` mermaid
        graph LR
    A[New Interaction] --> B{Memory Type?}
    B -->|Current turn| C[Working Memory\nContext window]
    B -->|User fact / preference| D[Semantic Memory\nVector store / KV]
    B -->|What happened| E[Episodic Memory\nSummarized log]
    B -->|How to do X| F[Procedural Memory\nWorkflow store]

    C --> G[LangGraph Checkpoints]
    D --> H[LangGraph Store\nor Mem0]
    E --> H
    F --> H
```

Effective agent memory combines all four types:

1.  **Working memory** (checkpoints) for within-session continuity

2.  **Semantic memory** (vector store) for durable facts and preferences

3.  **Episodic memory** (summarized logs) for learning from past interactions

4.  **Procedural memory** (workflow store) for skill acquisition over time

The discipline is knowing what to store, how long to keep it, and when to retrieve it — not storing everything indefinitely.

## Further Reading

- [LangGraph Foundations & State Management](langgraph-foundations.md) — LangGraph state and checkpointing

- [Human-in-the-Loop & Persistence](human-in-the-loop.md) — Interrupting agents for human verification

- [Multi-Agent Collaboration](multi-agent-collaboration.md) — Memory sharing across agent teams

- [Observability: LangFuse & LangSmith](../llmops/llmops-observability.md) — Monitoring memory retrieval quality in production
