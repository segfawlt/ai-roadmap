# LangGraph Foundations & State Management

This page introduces the core concepts of LangGraph — graphs, nodes, edges, and message-centric state management — and shows how to wire them together to build reliable, stateful AI workflows with LLM integration.

## Learning Objectives

- Understand LangGraph architecture and the role of Messages in State

- Master State Management with messages-centric pattern

- Distinguish between messages (I/O) and context (metadata)

- Build Nodes and Edges with LangChain messages

- Create workflows with LLM integration

## What is LangGraph?

### Introduction

LangGraph is a powerful framework for building complex AI applications with capabilities:

#### Orchestration framework for LLMs

- Orchestrate multiple LLM calls in a workflow

- Manage conversation flow with messages

- Optimize API calls and parallel processing

#### Built on top of LangChain

- Seamless integration with LangChain components

- Uses LangChain message types (AIMessage, HumanMessage, SystemMessage)

- Extends LangChain capabilities with state management

#### State-based workflow engine

- **Messages**: Core of I/O between nodes

- **Context**: Additional metadata and configuration

- Type-safe state with TypedDict

#### Developed by LangChain team

- Actively maintained with regular updates

- Production-ready and battle-tested

- Rich community support and documentation

### Why LangGraph?

#### Complex workflows

**Problem**: LangChain chains only support linear flows (A → B → C)

**LangGraph Solution**:

``` python
# LangChain: Linear only
chain = prompt | llm | output_parser

# LangGraph: Complex flows with messages
workflow.add_node("analyze", analyze_fn)
workflow.add_node("research", research_fn)
workflow.add_conditional_edges("analyze", router, {
    "need_more_data": "research",
    "ready": "synthesize"
})
```

#### Cyclic flows

Supports loops and iterations - messages accumulate across cycles:

``` python
# Retry loop with message history
workflow.add_conditional_edges(
    "generate",
    check_quality,
    {
        "pass": END,
        "fail": "refine"  # Messages retain history
    }
)
```

#### Human-in-the-loop

Pause workflow, inject HumanMessage:

``` python
# Human adds message to flow
workflow.add_node("review", human_review_node)
# State["messages"] will have HumanMessage after review
```

#### Stateful applications

Messages naturally store conversation history:

``` python
class ConversationState(TypedDict):
    messages: Annotated[List[AnyMessage], add_messages]
    user_id: str  # Context
    session_id: str  # Context
```

#### Agent systems

Multi-agent with shared message history:

``` python
# Agents communicate through messages
workflow.add_node("researcher", research_agent)
workflow.add_node("writer", writing_agent)
# All read/write to state["messages"]
```

### LangChain vs LangGraph

| Aspect              | LangChain          | LangGraph                  |
|---------------------|--------------------|----------------------------|
| **Flow Type**       | Linear, sequential | Cyclic, conditional        |
| **State**           | Implicit           | Explicit with messages     |
| **Message History** | In chain only      | Persistent in state        |
| **Loops**           | Not supported      | Native support             |
| **Conditionals**    | Limited            | Flexible routing           |
| **Use Case**        | Simple pipelines   | Complex agents, multi-turn |

## Core Concepts

### Graph (StateGraph)

Graph is a directed graph to orchestrate LLM workflows:

``` python
from langgraph.graph import StateGraph, END

# Create graph with state type
workflow = StateGraph(AgentState)
```

### State - Messages-Centric Pattern

**🔑 Key Principle**: State in LangGraph follows the pattern:

- **`messages`**: Core field for ALL input/output from nodes

- **Other fields**: Context, metadata, configuration

#### State Structure

``` python
from typing import TypedDict, List, Annotated
from langchain.messages import AnyMessage
from langgraph.graph import add_messages

class AgentState(TypedDict):
    """
    State structure for LangGraph agent.

    messages: REQUIRED - Core communication channel
    Other fields: Optional context and metadata
    """
    # CORE: Messages for I/O
    messages: Annotated[List[AnyMessage], add_messages]

    # CONTEXT: Additional data not I/O
    user_id: str
    session_id: str
    max_iterations: int
    current_iteration: int
```

#### Why Messages are Core?

1.  **Standardized I/O**: All nodes read/write messages

2.  **LangChain Integration**: Compatible with LLMs, tools, agents

3.  **History Tracking**: Auto accumulate conversation

4.  **Type Safety**: AIMessage, HumanMessage, SystemMessage, ToolMessage

### Messages Types

``` python
from langchain.messages import (
    AIMessage,      # LLM responses
    HumanMessage,   # User inputs
    SystemMessage,  # System prompts
    ToolMessage,    # Tool outputs
    FunctionMessage # Function calls (deprecated)
)

# Example messages
messages = [
    SystemMessage(content="You are a helpful assistant"),
    HumanMessage(content="What is LangGraph?"),
    AIMessage(content="LangGraph is a framework for..."),
    HumanMessage(content="Can you explain more?"),
]
```

### add_messages Reducer

`add_messages` is a special reducer for messages:

``` python
from langgraph.graph import add_messages

class State(TypedDict):
    messages: Annotated[List[AnyMessage], add_messages]
```

**Behavior**:

- Append new messages to list

- Handle message IDs and deduplication

- Merge messages intelligently

``` python
# Node 1 returns
{"messages": [AIMessage(content="Hello")]}
# State: messages = [AIMessage("Hello")]

# Node 2 returns
{"messages": [HumanMessage(content="Hi")]}
# State: messages = [AIMessage("Hello"), HumanMessage("Hi")]
```

### Context Fields

Context fields are metadata NOT I/O:

``` python
class ResearchState(TypedDict):
    # Core I/O
    messages: Annotated[List[AnyMessage], add_messages]

    # Context: Configuration
    max_iterations: int
    search_depth: str  # "shallow" | "deep"

    # Context: Tracking
    current_iteration: int
    sources_found: List[str]

    # Context: User info
    user_id: str
    preferences: dict
```

**When to use context fields?**

- Configuration (max_iterations, timeouts)

- Metadata (user_id, session_id, timestamps)

- Tracking (iteration count, metrics)

- Non-conversational data (file paths, API keys) =\> When you want to pass additional context from outside to agent tools

### Nodes (Functions)

#### Node Pattern with Messages

``` python
def my_node(state: AgentState) -> dict:
    """
    Node function pattern:
    1. Read messages from state
    2. Process (call LLM, tools, etc)
    3. Return new messages
    """
    # Read messages
    messages = state["messages"]
    last_message = messages[-1]

    # Process with LLM
    response = llm.invoke(messages)

    # Return new messages
    return {"messages": [response]}
```

#### LLM Node Example

``` python
from langchain.chat_models import init_chat_model

llm = init_chat_model("claude-sonnet-4-6")

def llm_node(state: AgentState) -> dict:
    """Call LLM with message history"""
    # LLM automatically uses all messages
    response = llm.invoke(state["messages"])

    # Return AIMessage
    return {"messages": [response]}
```

#### Tool Node Example

``` python
from langchain.messages import ToolMessage

def tool_node(state: AgentState) -> dict:
    """Execute tool and return ToolMessage"""
    last_message = state["messages"][-1]

    # Extract tool call
    tool_call = last_message.tool_calls[0]

    # Execute tool
    result = execute_tool(tool_call)

    # Return ToolMessage
    tool_message = ToolMessage(
        content=str(result),
        tool_call_id=tool_call["id"]
    )

    return {"messages": [tool_message]}
```

### Edges (Connections)

#### Normal edges

``` python
workflow.add_edge("node_a", "node_b")
```

#### Conditional edges based on messages

``` python
def should_continue(state: AgentState) -> str:
    """Route based on last message"""
    last_message = state["messages"][-1]

    # Check if AI wants to use tool
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"

    # Check iteration limit
    if state["current_iteration"] >= state["max_iterations"]:
        return "end"

    return "continue"

workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "tools": "tool_node",
        "continue": "agent",
        "end": END
    }
)
```

## State Management Deep Dive

### Messages-First Design

``` python
# ✅ GOOD: Messages-centric
class GoodState(TypedDict):
    messages: Annotated[List[AnyMessage], add_messages]
    user_id: str
    config: dict

# ❌ BAD: No messages
class BadState(TypedDict):
    input_text: str
    output_text: str
    context: dict
```

### Input/Output Pattern

``` python
class WorkflowState(TypedDict):
    """
    Messages: ALL conversational I/O
    Context: Everything else
    """
    # I/O Channel
    messages: Annotated[List[AnyMessage], add_messages]

    # Context
    documents: List[str]      # Retrieved docs
    search_queries: List[str] # Generated queries
    metrics: dict             # Performance tracking
```

**Nodes communicate through messages:**

``` python
def node_1(state: WorkflowState) -> dict:
    # Read from messages
    user_query = state["messages"][-1].content

    # Use context
    docs = state["documents"]

    # Return via messages
    response = f"Based on {len(docs)} documents: ..."
    return {"messages": [AIMessage(content=response)]}
```

### Context Injection Pattern

Context is injected into initial state:

``` python
# Initialize state with context
initial_state = {
    "messages": [
        SystemMessage(content="You are a helpful assistant"),
        HumanMessage(content="User question here")
    ],
    # Inject context
    "user_id": "user_123",
    "session_id": "session_456",
    "max_iterations": 5,
    "current_iteration": 0,
    "preferences": {"style": "concise"}
}

# Run workflow
result = app.invoke(initial_state)

# Access messages
final_messages = result["messages"]
```

### Multi-Agent State Pattern

``` python
class MultiAgentState(TypedDict):
    """State for multi-agent system"""
    # Shared message channel
    messages: Annotated[List[AnyMessage], add_messages]

    # Agent context
    current_agent: str
    agent_outputs: dict[str, str]

    # Workflow context
    task_type: str
    priority: int

def researcher_agent(state: MultiAgentState) -> dict:
    """Research agent adds messages"""
    messages = state["messages"]

    # Do research
    findings = research(messages[-1].content)

    return {
        "messages": [AIMessage(
            content=findings,
            name="researcher"  # Tag with agent name
        )],
        "current_agent": "researcher"
    }

def writer_agent(state: MultiAgentState) -> dict:
    """Writer agent reads researcher's messages"""
    messages = state["messages"]

    # Get researcher's findings
    researcher_msg = [m for m in messages if m.name == "researcher"][-1]

    # Write based on findings
    article = write_article(researcher_msg.content)

    return {
        "messages": [AIMessage(
            content=article,
            name="writer"
        )],
        "current_agent": "writer"
    }
```

## Checkpointer (State Persistence & Memory)

### What Is a Checkpointer?

A **checkpointer** is the component responsible for **persisting and restoring graph state** between executions.

It allows LangGraph to:

- Remember conversations across turns

- Resume execution after failures

- Replay or “time-travel” to previous steps

- Inspect intermediate states for debugging

Without a checkpointer, every graph invocation is **stateless**.

### Why Checkpointers Work Best with Messages-First Design

- `messages` is append-only and deterministic

- Each node returns a **state delta**

- The full conversational history can be reconstructed

- Graph execution becomes replayable and debuggable

This is why **all conversational I/O must live in `messages`**.

## Built-in Checkpointer: `InMemorySaver`

`InMemorySaver` is the simplest checkpointer implementation provided by LangGraph. (Older code may reference `MemorySaver` — it is kept as an alias but `InMemorySaver` is the canonical name in current LangGraph releases.)

### Characteristics

| Feature       | Description                       |
|---------------|-----------------------------------|
| Storage       | In-memory                         |
| Persistence   | Lost on process restart           |
| Thread Safety | Per process                       |
| Best Use      | Local development, demos, testing |

------------------------------------------------------------------------

### Importing the Checkpointer

``` python
from langgraph.checkpoint.memory import InMemorySaver
```

## Building First Graph

### Setup

``` bash
pip install -U "langchain[anthropic]" langgraph
```

### Complete Example: Simple Chat Agent

``` python
from typing import TypedDict, List, Annotated

from langchain.messages import (
    AnyMessage,
    HumanMessage,
    AIMessage,
    SystemMessage,
)
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END, add_messages
from langgraph.checkpoint.memory import InMemorySaver


# 1. Define State
class ChatState(TypedDict):
    """Simple chat state"""
    messages: Annotated[List[AnyMessage], add_messages]
    user_name: str  # Context

# 2. Initialize LLM
llm = init_chat_model("claude-sonnet-4-6")

# 3. Define Nodes
def chatbot_node(state: ChatState) -> dict:
    """Main chatbot node"""
    # Get user name from context
    user_name = state.get("user_name", "User")

    # Personalize system message
    messages = state["messages"]
    if not any(isinstance(m, SystemMessage) for m in messages):
        system_msg = SystemMessage(
            content=f"You are helping {user_name}. Be friendly and concise."
        )
        messages = [system_msg] + messages

    # Call LLM
    response = llm.invoke(messages)

    # Return new message
    return {"messages": [response]}

# 4. Create Graph
workflow = StateGraph(ChatState)

# 5. Add Node
workflow.add_node("chatbot", chatbot_node)

# 6. Set Entry and Exit
workflow.add_edge(START, "chatbot")
workflow.add_edge("chatbot", END)
checkpointer = InMemorySaver()

# 7. Compile
app = workflow.compile(checkpointer=checkpointer)

# 8. Run
config = {
    "configurable": {
        "thread_id": "alice-chat"
    }
}

result = app.invoke(
    {
        "messages": [HumanMessage(content="What is LangGraph?")],
        "user_name": "Alice",
    },
    config=config,
)


result = app.invoke(
    {
        "messages": [HumanMessage(content="Why is it better than chains?")]
    },
    config=config,
)

# Print conversation
for msg in result["messages"]:
    print(f"{msg.__class__.__name__}: {msg.content}\n")
```

### Output

```
HumanMessage: What is LangGraph?

AIMessage: Hi Alice! LangGraph is a framework built on top of LangChain
that allows you to create stateful, multi-step workflows with LLMs...

HumanMessage: Why is it better than chains?

AIMessage: Great question! LangGraph improves on chains by adding
explicit state, branching, and durable memory via checkpoints.
```

## Conditional Routing with Messages

### Tool Calling Pattern

``` python
from typing import Annotated, List, TypedDict

from langchain.messages import AnyMessage, ToolMessage
from langchain.tools import tool
from langgraph.graph import StateGraph, START, END, add_messages

# Define tool
@tool
def search_web(query: str) -> str:
    """Search the web for information"""
    return f"Search results for: {query}"

class AgentState(TypedDict):
    messages: Annotated[List[AnyMessage], add_messages]

# Bind tool to LLM
llm_with_tools = llm.bind_tools([search_web])

def agent_node(state: AgentState) -> dict:
    """Agent decides to call tool or respond"""
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

def tool_node(state: AgentState) -> dict:
    """Execute tools from last message"""
    last_message = state["messages"][-1]

    tool_messages = []
    for tool_call in last_message.tool_calls:
        # Execute tool
        result = search_web.invoke(tool_call["args"])

        # Create ToolMessage
        tool_messages.append(ToolMessage(
            content=result,
            tool_call_id=tool_call["id"]
        ))

    return {"messages": tool_messages}

def should_continue(state: AgentState) -> str:
    """Route based on tool calls"""
    last_message = state["messages"][-1]

    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return "end"

# Build graph
workflow = StateGraph(AgentState)
workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)

workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "tools": "tools",
        "end": END
    }
)
workflow.add_edge("tools", "agent")  # Loop back

app = workflow.compile()
```

## Practice: Re-implement Toolcalling Agent as above

**Reference**:

- LangGraph ReAct Pattern: <https://langchain-ai.github.io/langgraph/how-tos/react-agent-from-scratch/>

- Prebuilt ToolNode: [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/langgraph/prebuilt/tool_node.py)

- Message Types: <https://docs.langchain.com/oss/python/langchain/overview>

## Best Practices

### 1. Always Use Messages for I/O

``` python
# ✅ GOOD
class State(TypedDict):
    messages: Annotated[List[AnyMessage], add_messages]
    user_id: str

def node(state: State) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# ❌ BAD
class State(TypedDict):
    input: str
    output: str

def node(state: State) -> dict:
    output = llm.invoke(state["input"])
    return {"output": output}
```

### 2. Separate Concerns

``` python
class WellDesignedState(TypedDict):
    # I/O: Conversational data
    messages: Annotated[List[AnyMessage], add_messages]

    # Context: User info
    user_id: str
    preferences: dict

    # Context: Workflow control
    max_iterations: int
    current_step: str

    # Context: Results tracking
    sources: List[str]
    confidence_scores: List[float]
```

### 3. Type Message Roles

``` python
def create_system_message(user_name: str) -> SystemMessage:
    """Factory for system messages"""
    return SystemMessage(
        content=f"You are assisting {user_name}. Be helpful and concise."
    )

def node(state: State) -> dict:
    # Tag messages with metadata
    response = AIMessage(
        content="Response here",
        name="research_agent",  # Agent identifier
        additional_kwargs={"confidence": 0.95}
    )
    return {"messages": [response]}
```

### 4. Handle Message History (Trimmessage)

<https://langchain-ai.github.io/langgraph/how-tos/create-react-agent-manage-message-history/>

### 5. Context Injection Pattern

``` python
# Initialize with full context
def create_initial_state(user_query: str, user_id: str) -> dict:
    return {
        "messages": [
            SystemMessage(content="You are a helpful assistant"),
            HumanMessage(content=user_query)
        ],
        "user_id": user_id,
        "session_id": generate_session_id(),
        "timestamp": datetime.now().isoformat(),
        "max_iterations": 5,
        "current_iteration": 0
    }

initial_state = create_initial_state(
    user_query="What is LangGraph?",
    user_id="user_123"
)

result = app.invoke(initial_state)
```

## Common Patterns

### 1. Agent with Tools Pattern

``` python
class AgentState(TypedDict):
    messages: Annotated[List[AnyMessage], add_messages]

llm_with_tools = llm.bind_tools([tool1, tool2])

workflow = StateGraph(AgentState)
workflow.add_node("agent", lambda s: {"messages": [llm_with_tools.invoke(s["messages"])]})
workflow.add_node("tools", tool_executor)

workflow.add_conditional_edges("agent", should_continue, {
    "tools": "tools",
    "end": END
})
workflow.add_edge("tools", "agent")
```

### 2. Multi-Agent Collaboration

``` python
class MultiAgentState(TypedDict):
    messages: Annotated[List[AnyMessage], add_messages]
    current_agent: str

def agent_1(state):
    response = llm.invoke(state["messages"])
    return {
        "messages": [AIMessage(content=response.content, name="agent_1")],
        "current_agent": "agent_1"
    }

def agent_2(state):
    # Filter messages from agent_1
    agent_1_messages = [m for m in state["messages"] if m.name == "agent_1"]
    response = llm.invoke(agent_1_messages)
    return {
        "messages": [AIMessage(content=response.content, name="agent_2")],
        "current_agent": "agent_2"
    }
```

### 3. Human-in-the-Loop:

<https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/wait-user-input/>

## Debugging

### Print Message History

``` python
def debug_node(state: State) -> dict:
    """Debug node to inspect messages"""
    print("\n=== MESSAGE HISTORY ===")
    for i, msg in enumerate(state["messages"]):
        print(f"{i+1}. {msg.__class__.__name__}: {msg.content[:100]}...")
    print("=" * 50)
    return {}

workflow.add_node("debug", debug_node)
```

### Visualize Graph

``` python
from IPython.display import Image, display

# Display graph
display(Image(app.get_graph().draw_mermaid_png()))

# Or save to file
with open("graph.png", "wb") as f:
    f.write(app.get_graph().draw_mermaid_png())
```

## Summary

## LangGraph in 2026: What’s New NEW

### LangGraph v1.0 (Stable API)

LangGraph reached v1.0 in late 2025, stabilizing the API after several years of rapid iteration. The framework is now widely adopted in production:

- **34.5M monthly PyPI downloads** — one of the most-downloaded agent orchestration libraries

- **~400 companies in production**, including Uber, Cisco, LinkedIn, and JPMorgan

- **LangServe moved to maintenance mode** — LangGraph Platform is the official deployment path going forward

The stable API means fewer breaking changes between releases, making it safer to build long-lived production systems on LangGraph.

### Production Persistence

`InMemorySaver` is fine for development and testing, but production deployments require a durable checkpointer. When the process restarts, in-memory state is lost — which breaks multi-turn conversations and long-running workflows.

Recommended production checkpointers:

- **`AsyncPostgresSaver`** — recommended for most production setups; integrates with existing PostgreSQL infrastructure

- **`RedisStore`** — for high-throughput, low-latency checkpointing where sub-millisecond read/write matters

Additional production features:

- **Encryption** via the `LANGGRAPH_AES_KEY` environment variable — checkpoint data is encrypted at rest

- **TTL support** for automatic checkpoint cleanup — prevents unbounded storage growth

``` python
# Production persistence setup
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async with AsyncPostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/db"
) as checkpointer:
    graph = workflow.compile(checkpointer=checkpointer)
```

The `async`` ``with` context manager handles connection lifecycle. The graph compiled inside the block uses the PostgreSQL checkpointer for all state reads and writes.

### Cross-Thread Memory (Store Interface)

LangGraph v1.0 introduced the `Store` interface to solve a common production problem: sharing knowledge across separate conversation threads.

**The distinction matters:**

- **Thread-local state** (`messages` in `AgentState`) — conversation history within a single session; lost when the thread ends

- **Cross-thread Store** — persistent facts, user preferences, and learned patterns that survive across sessions

``` python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
graph = workflow.compile(checkpointer=checkpointer, store=store)

# Inside a node, access the store:
def my_node(state, config, *, store):
    user_id = config["configurable"]["user_id"]
    memories = store.search(("user", user_id))
    # ... use memories
    store.put(("user", user_id), "preference", {"theme": "dark"})
```

The store uses a namespace tuple `("user",`` ``user_id)` to scope data per user. `store.search()` retrieves existing entries; `store.put()` writes new ones. In production, replace `InMemoryStore` with a persistent backend (e.g., PostgreSQL-backed store) so preferences survive process restarts.

### LangGraph Platform (LangSmith Deployment)

LangGraph Platform GA’d in May 2025 as the official way to deploy LangGraph agents to production. It was renamed to **LangSmith Deployment** in October 2025, but the underlying capabilities are the same.

Deployment modes:

- **Cloud (SaaS)** — managed infrastructure, fastest to get started

- **Hybrid** — your data stays in your VPC; control plane is managed

- **Self-Hosted** — full control over infrastructure and data

Key capabilities:

- `langgraph`` ``deploy` command deploys directly from a git repository

- **Background runs** — agents execute asynchronously; results retrieved later

- **Cron scheduling** — trigger agent runs on a schedule

- **LangGraph Studio UI** — visual debugger for inspecting graph execution, state at each step, and message history

- **30+ API endpoints** for agent management (create runs, stream output, retrieve state, cancel runs, etc.)

LangGraph Studio is particularly useful during development — it renders the graph visually and lets you step through execution to see exactly what state each node received and returned.
