# Assignment: LangGraph Foundations & State Management

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | LangGraph Foundations & State Management |
| **Course** | LangGraph and Agentic AI |
| **Project Name** | `langgraph-chatbot-agent` |
| **Estimated Time** | 90 minutes |
| **Framework** | Python 3.11+, LangGraph 1.x, LangChain 1.x, Anthropic (or OpenAI) API |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Understand** LangGraph architecture and the role of Messages in State

- **Implement** State Management with messages-centric pattern using TypedDict

- **Build** Nodes and Edges with LangChain message types

- **Configure** InMemorySaver checkpointer for conversation persistence

- **Create** a complete chatbot workflow with conditional routing

------------------------------------------------------------------------

## Problem Description

You are tasked with building a conversational AI agent using LangGraph that can:

1.  Maintain conversation history across multiple turns

2.  Use proper message types (HumanMessage, AIMessage, SystemMessage)

3.  Implement conditional routing based on user intent

4.  Persist state using InMemorySaver for multi-session conversations

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langgraph` \>= 1.0

  - `langchain[anthropic]` \>= 1.0 (or `langchain[openai]`)

### API Requirements

- Anthropic API key (or OpenAI API key) configured as environment variable

------------------------------------------------------------------------

## Tasks

### Task 1: State Definition (20 points)

1.  **Define an AgentState** using TypedDict that includes:

    - `messages`: Using `Annotated[List[AnyMessage],`` ``add_messages]` pattern

    - `user_name`: Context field for personalization

    - `session_id`: Context field for tracking

2.  **Implement proper message handling** that:

    - Uses `add_messages` reducer for automatic message merging

    - Handles all message types (HumanMessage, AIMessage, SystemMessage)

### Task 2: Node Implementation (30 points)

1.  **Create a chatbot node** that:

    - Reads messages from state

    - Adds a personalized system message if not present

    - Calls the LLM with message history

    - Returns new AIMessage to state

2.  **Create a routing node** that:

    - Analyzes the last user message

    - Routes to appropriate handler based on intent (help, general, goodbye)

### Task 3: Graph Construction (30 points)

1.  **Build the StateGraph** with:

    - Entry point set to chatbot node

    - Conditional edges based on routing logic

    - Proper END state configuration

2.  **Configure InMemorySaver checkpointer** for state persistence

3.  **Implement multi-turn conversation** that maintains context across invocations

### Task 4: Testing & Validation (20 points)

1.  **Create test conversations** demonstrating:

    - Multi-turn dialogue with context retention

    - Different routing paths (help, general, goodbye)

    - Session persistence (same thread_id across calls)

2.  **Debug output** showing message history at each step

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code in Python script or Jupyter notebook

`README.md` with setup instructions

Screenshots showing multi-turn conversation output

Graph visualization (using `get_graph().draw_mermaid_png()`)

### Submission Checklist

State uses messages-centric pattern with `add_messages`

All node functions follow the correct signature `(state)`` ``->`` ``dict`

Conditional routing works correctly

InMemorySaver enables conversation persistence

Code runs without errors

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                       | Points  |
|--------------------------------|---------|
| State definition with messages | 20      |
| Node implementation            | 30      |
| Graph construction & routing   | 30      |
| Testing & validation           | 15      |
| Code quality and documentation | 5       |
| **Total**                      | **100** |

------------------------------------------------------------------------

## Hints

- Use `state["messages"][-1]` to access the most recent message

- The `add_messages` reducer automatically handles message deduplication

- Use `config`` ``=`` ``{"configurable":`` ``{"thread_id":`` ``"unique-id"}}` for session tracking

- Reference the knowledge document for node function patterns

------------------------------------------------------------------------

## References

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)

- [LangGraph ReAct Pattern](https://langchain-ai.github.io/langgraph/how-tos/react-agent-from-scratch/)

- [LangChain Message Types](https://docs.langchain.com/oss/python/langchain/overview)
