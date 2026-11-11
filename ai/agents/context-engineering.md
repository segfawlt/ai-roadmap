# Context Engineering

Context engineering is the discipline of deliberately designing, structuring, and managing the total information environment provided to an LLM — system prompts, retrieved documents, tool outputs, conversation history, and structured metadata — to maximize the quality and reliability of its outputs.

While prompt engineering focuses on crafting the instruction text, context engineering treats the **entire context window as an engineered artifact** with a budget, architecture, and lifecycle.

## Learning Objectives

- Understand the distinction between prompt engineering and context engineering

- Map the anatomy of an LLM context window and its components

- Apply token budgeting strategies for context allocation

- Distinguish write context from read context

- Implement dynamic context assembly with RAG and tool outputs

- Leverage context caching for cost optimization

- Recognize common context anti-patterns and their fixes

## 1. From Prompt Engineering to Context Engineering

Prompt engineering treats the user instruction as the primary lever for model behavior. Context engineering expands the scope to everything the model sees:

```
Prompt Engineering          Context Engineering
─────────────────          ────────────────────
"Write a summary"    →     System prompt + persona
                           + retrieved documents
                           + tool call results
                           + conversation history
                           + output format schema
                           + few-shot examples
```

The shift matters because production LLM systems rarely depend on a single instruction. A customer-support agent’s quality depends on the retrieved ticket history, the customer profile, the policy documents, and the tool outputs — not just “Answer the customer’s question.”

## 2. Anatomy of a Context Window

Every LLM call assembles a context window from distinct components, each serving a different role:

``` mermaid
        graph TB
    CW["Context Window (e.g. 128k tokens)"]
    CW --> SP["System Prompt<br/>Identity, rules, output format"]
    CW --> FS["Few-Shot Examples<br/>Desired input→output pairs"]
    CW --> RAG["Retrieved Context<br/>Documents from vector search"]
    CW --> TO["Tool Outputs<br/>API responses, DB results"]
    CW --> CH["Conversation History<br/>Prior turns in the session"]
    CW --> UI["User Instruction<br/>The current request"]
```

### Component Roles

| Component | Purpose | Typical Size |
|----|----|----|
| System prompt | Define persona, rules, output format | 500–2,000 tokens |
| Few-shot examples | Demonstrate desired behavior | 500–3,000 tokens |
| Retrieved context | Ground answers in source material | 2,000–20,000 tokens |
| Tool outputs | Inject real-time data | 500–5,000 tokens |
| Conversation history | Maintain multi-turn coherence | 1,000–50,000 tokens |
| User instruction | The current task | 50–500 tokens |

## 3. Context Budgeting

A context budget allocates the available token window across components, just like a financial budget allocates funds across departments. Exceeding the budget degrades quality; wasting it leaves the model under-informed.

### Priority-Based Allocation

``` python
from tiktoken import encoding_for_model

enc = encoding_for_model("gpt-4o")

CONTEXT_LIMIT = 128_000
RESPONSE_RESERVE = 4_096  # tokens reserved for output

BUDGET = {
    "system_prompt": 1_500,
    "few_shot": 2_000,
    "retrieved_context": 20_000,
    "tool_outputs": 5_000,
    "conversation_history": CONTEXT_LIMIT - RESPONSE_RESERVE - 28_500,
    # remaining tokens go to history
}


def count_tokens(text: str) -> int:
    return len(enc.encode(text))


def truncate_to_budget(text: str, max_tokens: int) -> str:
    """Truncate text to fit within token budget."""
    tokens = enc.encode(text)
    if len(tokens) <= max_tokens:
        return text
    return enc.decode(tokens[:max_tokens])
```

### Truncation Strategies

When a component exceeds its budget, you need a truncation strategy:

1.  **Recency**: Keep the most recent items (conversation history)

2.  **Relevance**: Keep the highest-scored items (retrieved documents)

3.  **Summarization**: Compress older content into summaries

4.  **Sliding window**: Drop the oldest items beyond a threshold

``` python
def manage_conversation_history(
    messages: list[dict],
    max_tokens: int,
    summarizer=None,
) -> list[dict]:
    """Keep recent messages within budget, summarize overflow."""
    total = sum(count_tokens(m["content"]) for m in messages)

    if total <= max_tokens:
        return messages

    # Always keep the first message (system context) and last N turns
    keep_recent = 6  # last 3 exchanges
    old_messages = messages[:-keep_recent]
    recent_messages = messages[-keep_recent:]

    if summarizer:
        summary = summarizer(old_messages)
        return [{"role": "system", "content": f"Earlier conversation summary: {summary}"}] + recent_messages

    # Fallback: drop oldest messages until within budget
    while old_messages and total > max_tokens:
        dropped = old_messages.pop(0)
        total -= count_tokens(dropped["content"])

    return old_messages + recent_messages
```

## 4. Write Context vs. Read Context

Andrej Karpathy’s framing distinguishes two modes of context:

- **Write context**: Information you author and control — system prompts, few-shot examples, output schemas. These are **static** and optimized offline.

- **Read context**: Information assembled at runtime — retrieved documents, tool outputs, user messages. These are **dynamic** and require runtime engineering.

``` mermaid
        graph LR
    subgraph Write["Write Context (authored offline)"]
        SP2["System prompt"]
        FS2["Few-shot examples"]
        OS["Output schema"]
    end
    subgraph Read["Read Context (assembled at runtime)"]
        RAG2["RAG results"]
        TO2["Tool outputs"]
        CH2["Chat history"]
        UI2["User message"]
    end
    Write --> MW["Context Window"]
    Read --> MW
```

### Optimization Strategies by Type

**Write context** (optimize once, reuse many times):

- A/B test system prompt variants across evaluation datasets

- Curate few-shot examples that cover edge cases

- Version-control prompts alongside application code

**Read context** (optimize per-request):

- Rank retrieved documents by relevance before injection

- Summarize verbose tool outputs

- Compress conversation history as it grows

## 5. Dynamic Context Assembly

In production, context is assembled from multiple sources at request time. The assembly pipeline determines what the model sees:

``` python
from langchain_openai import OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.documents import Document

SYSTEM_TEMPLATE = """You are a technical support agent for Acme Cloud.

## Rules
- Only answer based on the provided context
- If unsure, say "I'll escalate this to a specialist"
- Always cite the source document

## Customer Profile
{customer_profile}

## Retrieved Documentation
{retrieved_docs}

## Recent Ticket History
{ticket_history}
"""


def assemble_context(
    user_query: str,
    customer_id: str,
    retriever,
    ticket_db,
    customer_db,
) -> list:
    """Assemble the full context for a support query."""
    # 1. Retrieve relevant documentation
    docs = retriever.invoke(user_query)
    retrieved_text = "\n\n".join(
        f"[{doc.metadata['source']}]\n{doc.page_content}" for doc in docs[:5]
    )

    # 2. Fetch customer profile (tool output)
    profile = customer_db.get_profile(customer_id)
    profile_text = f"Name: {profile['name']}, Plan: {profile['plan']}, Region: {profile['region']}"

    # 3. Get recent ticket history
    tickets = ticket_db.get_recent(customer_id, limit=5)
    history_text = "\n".join(f"- [{t['date']}] {t['subject']}: {t['status']}" for t in tickets)

    # 4. Assemble into prompt
    prompt = ChatPromptTemplate.from_messages([
        ("system", SYSTEM_TEMPLATE),
        ("human", "{query}"),
    ])

    return prompt.invoke({
        "customer_profile": profile_text,
        "retrieved_docs": retrieved_text,
        "ticket_history": history_text,
        "query": user_query,
    })
```

### Source Attribution

Always tag retrieved content with its source so the model can cite it:

``` python
def format_retrieved_docs(docs: list[Document]) -> str:
    """Format documents with source attribution."""
    formatted = []
    for i, doc in enumerate(docs, 1):
        source = doc.metadata.get("source", "unknown")
        formatted.append(f"[Source {i}: {source}]\n{doc.page_content}")
    return "\n\n---\n\n".join(formatted)
```

## 6. Context Caching and Optimization

LLM providers offer caching mechanisms that reduce cost and latency when the context prefix is reused across requests:

| Provider | Feature | How It Works |
|----|----|----|
| Anthropic | Prompt caching | Cache static prefix; pay reduced rate for cached tokens |
| OpenAI | Automatic prefix caching | Automatically caches matching prefixes (\>1024 tokens) |
| Google | Context caching | Explicit cache creation with TTL |

### Anthropic Cache Example

``` python
from anthropic import Anthropic

client = Anthropic()

# The system prompt and few-shot examples are cached
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are a technical documentation assistant...",  # long system prompt
            "cache_control": {"type": "ephemeral"},
        }
    ],
    messages=[{"role": "user", "content": "How do I configure SSL?"}],
)
```

### Design for Cacheability

Structure your context so that the **stable prefix** (system prompt, few-shot examples, reference documentation) comes first, and **dynamic content** (user query, tool outputs) comes last:

``` mermaid
        graph TB
    subgraph CACHED["Cached — Stable across requests"]
        A["System prompt"]
        B["Few-shot examples"]
        C["Reference docs"]
    end
    subgraph DYNAMIC["Dynamic — Changes per request"]
        D["Retrieved context"]
        E["User query"]
    end
    CACHED --> DYNAMIC

    style CACHED fill:#1a3a1a,stroke:#4caf50,color:#e0e0e0
    style DYNAMIC fill:#3a1a1a,stroke:#f44336,color:#e0e0e0
```

## 7. Grounding and Hallucination Reduction

Context engineering is your primary defense against hallucination. The model can only be faithful to context it actually receives:

### Grounding Techniques

1.  **Explicit sourcing instructions**: “Only answer based on the provided documents. If the answer is not in the documents, say so.”

2.  **Source tagging**: Label each document chunk so the model can cite `[Source`` ``3]`

3.  **Relevance filtering**: Remove low-scoring retrieval results before injection — irrelevant context increases hallucination

4.  **Fact-check prompting**: Ask the model to verify its claims against the provided context in a second pass

### The Lost-in-the-Middle Problem

Research shows that LLMs pay more attention to information at the **beginning** and **end** of the context, with reduced attention to the middle. Mitigations:

- Place the most relevant documents first and last

- Keep the total number of retrieved documents small (3–5 instead of 10–20)

- Use reciprocal rank fusion to re-rank before injection

## 8. Patterns and Anti-Patterns

### Patterns (Do)

| Pattern | Description |
|----|----|
| **Budget-first design** | Define token budgets per component before building |
| **Layered context** | Stable prefix → semi-stable retrieval → dynamic query |
| **Progressive disclosure** | Start with summaries, fetch details on demand |
| **Context validation** | Log and monitor what the model actually sees |

### Anti-Patterns (Avoid)

| Anti-Pattern | Problem |
|----|----|
| **Context stuffing** | Dumping everything into the window degrades quality |
| **Blind retrieval** | Injecting all results without relevance filtering |
| **History hoarding** | Never truncating conversation history until token limit |
| **Prompt-only thinking** | Optimizing the instruction while ignoring retrieved context quality |

## 9. Putting It All Together

A production context engineering pipeline:

``` mermaid
        graph LR
    UQ["User Query"] --> RP["Route & Plan"]
    RP --> R["Retrieve"]
    RP --> T["Call Tools"]
    R --> F["Filter & Rank"]
    T --> S["Summarize Outputs"]
    F --> A["Assemble Context"]
    S --> A
    A --> B["Check Budget"]
    B -->|Over budget| TR["Truncate"]
    TR --> A
    B -->|Within budget| LLM["Send to LLM"]
    LLM --> V["Validate Output"]
```

## Prompt Caching: The Highest-ROI Optimization NEW

Prompt caching is the single most impactful cost optimization for LLM applications in 2026. When the prefix of a prompt matches a previously cached version, the provider serves it at a steep discount.

### How Prompt Caching Works

``` mermaid
        graph LR
    P["Prompt"] --> SP["Static Prefix<br/>(system prompt, few-shot examples,<br/>retrieved docs)"]
    P --> DP["Dynamic Suffix<br/>(user query, conversation turn)"]
    SP -->|"Cache Write (first call)"| CACHE[("Provider Cache<br/>TTL: 5 min")]
    CACHE -->|"Cache Read (subsequent calls)"| LLM["LLM"]
    DP --> LLM
```

### Provider Pricing

| Provider | Cache Write Cost | Cache Read Cost | Effective Discount | Min Tokens | TTL |
|----|----|----|----|----|----|
| Anthropic | 1.25x base price | 0.1x base price | 90% on reads | 1,024 | 5 min |
| OpenAI | 1x (auto, no extra) | 0.5x base price | 50% on reads | Auto | ~5-10 min |
| Google Gemini | Free (explicit caching) | 0.25x base price | 75% on reads | 32,768 | Configurable |

### Design Rules for Cache-Friendly Prompts

1.  **Static content FIRST**: System prompt → few-shot examples → stable retrieved docs → dynamic user query

2.  **Stable prefix**: Don’t change the system prompt between calls unless necessary

3.  **Batch similar requests**: Group requests that share the same context prefix

4.  **Monitor cache hit rate**: Track `cache_creation_input_tokens` vs `cache_read_input_tokens` in Anthropic responses

``` python
# Anthropic prompt caching example
from anthropic import Anthropic

client = Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "You are an expert audit assistant...",  # Long system prompt
            "cache_control": {"type": "ephemeral"}  # Cache this block
        }
    ],
    messages=[{"role": "user", "content": "What are the key risks?"}],
)
# Check cache performance in response.usage:
# cache_creation_input_tokens, cache_read_input_tokens
```

Typical production savings: 45-80% cost reduction when cache is properly configured.

## Practice

1.  Take an existing RAG pipeline and add explicit token budgeting. Measure how output quality changes as you vary the retrieved context budget from 2,000 to 20,000 tokens.

2.  Implement a conversation history manager that summarizes old messages when the history exceeds its token budget.

3.  Restructure a prompt to maximize cache hit rate by moving stable content to the prefix.
