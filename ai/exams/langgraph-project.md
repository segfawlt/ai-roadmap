# Final Project Exam: FPT Customer Chatbot - Multi-Agent AI System

## overview

| Field | Value |
|----|----|
| **Course** | LangGraph and Agentic AI |
| **Project Name** | `fpt-customer-chatbot-ai` |
| **Duration** | 360 minutes (6 hours) |
| **Passing Score** | 70% |
| **Total Points** | 100 |
| **Framework** | Python 3.11+, LangGraph 1.x, LangChain 1.x, Tavily API, FAISS, Anthropic (or OpenAI) |

------------------------------------------------------------------------

## Description

You have been hired as an **AI Engineer** at **FPT Software**, tasked with building a **Multi-Agent Customer Service Chatbot AI Core** that demonstrates mastery of all concepts covered in the LangGraph and Agentic AI module.

This final project consolidates all five assignments into a single comprehensive multi-agent system:

1.  **Assignment 01**: LangGraph Foundations & State Management

2.  **Assignment 02**: Multi-Expert ReAct Research Agent

3.  **Assignment 03**: Tool Calling & Tavily Search Integration

4.  **Assignment 04**: FPT Customer Chatbot - Multi-Agent System

5.  **Assignment 05**: Human-in-the-Loop & Persistence

You will build the **AI Core** for an FPT Customer Chatbot with hierarchical multi-agent architecture, real-time web search, human approval workflows, response caching, and persistent state management.

This exam focuses purely on the **AI/LangGraph logic**. For the **Engineering layer** (FastAPI, database, REST APIs), please refer to the **Building Monolith API with FastAPI** module’s final exam.

------------------------------------------------------------------------

## Objectives

By completing this exam, you will demonstrate mastery of:

- **State Management**: Implementing messages-centric patterns with TypedDict and add_messages reducer

- **ReAct Pattern**: Building reasoning + acting loops with iteration control

- **Tool Calling**: Integrating external APIs (Tavily) with parallel execution

- **Multi-Agent Architecture**: Designing hierarchical systems with specialized agents

- **Human-in-the-Loop**: Implementing interrupt patterns for user confirmation

- **Persistence**: Configuring checkpointers for long-running conversations

- **Caching**: Building vector store-based response caching with FAISS

------------------------------------------------------------------------

## Problem Description

Build the **AI Core** for an FPT Customer Service Chatbot named `fpt-customer-chatbot-ai` that includes:

| Agent | Responsibilities |
|----|----|
| **Primary Assistant** | Routes user queries to appropriate specialized agents |
| **FAQ Agent** | Answers FPT policy questions using RAG with cached responses |
| **Ticket Agent** | Handles ticket-related conversations with HITL approval (mock tools) |
| **Booking Agent** | Handles booking conversations with HITL confirmation (mock tools) |
| **IT Support Agent** | Troubleshoots technical issues using Tavily search + caching |

The system must:

- Maintain conversation context across multiple turns

- Require human confirmation before sensitive operations

- Cache responses for similar queries

- Persist state across process restarts

- Handle agent transitions gracefully with dialog stack

The Ticket and Booking agents will use **mock tools** that simulate database operations. The actual database integration is covered in the FastAPI module exam.

------------------------------------------------------------------------

## Prerequisites

- Completed all 5 module assignments (recommended)

- Anthropic API key (`ANTHROPIC_API_KEY`) or OpenAI API key (`OPENAI_API_KEY`)

- Tavily API key (`TAVILY_API_KEY`)

- Python 3.11+ with virtual environment

- Familiarity with Pydantic for schema validation

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langgraph` \>= 1.0

  - `langchain[anthropic]` \>= 1.0 \# or `langchain[openai]`

  - `langchain-tavily` \>= 0.1.0

  - `faiss-cpu` \>= 1.7.0

  - `sentence-transformers` \>= 2.2.0

  - `pydantic` \>= 2.0.0

### Mock Data Models

For testing purposes, define the following Pydantic models (actual database integration is in FastAPI module):

**Ticket Model:**

| Field          | Type         | Constraints                          |
|----------------|--------------|--------------------------------------|
| ticket_id      | str          | Auto-generated UUID                  |
| content        | str          | Required                             |
| description    | str \| None  | Optional                             |
| customer_name  | str          | Required                             |
| customer_phone | str          | Required                             |
| email          | str \| None  | Optional                             |
| status         | TicketStatus | Pending/InProgress/Resolved/Canceled |
| created_at     | datetime     | Auto-set                             |

**Booking Model:**

| Field          | Type          | Constraints                 |
|----------------|---------------|-----------------------------|
| booking_id     | str           | Auto-generated UUID         |
| reason         | str           | Required                    |
| time           | datetime      | Required, must be future    |
| customer_name  | str           | Required                    |
| customer_phone | str           | Required                    |
| email          | str \| None   | Optional                    |
| note           | str \| None   | Optional                    |
| status         | BookingStatus | Scheduled/Finished/Canceled |

------------------------------------------------------------------------

## Tasks

### Task 1: State Management Foundation (15 points)

**Time Allocation**: 60 minutes

Build the core state management infrastructure for the multi-agent system.

#### Requirements:

1.  **Define AgenticState** using TypedDict with:

    - `messages`: Using `Annotated[List[AnyMessage],`` ``add_messages]` pattern

    - `dialog_state`: Stack for tracking agent hierarchy

    - `user_id`, `email` (optional): Context injection fields

    - `conversation_id`: Session tracking

2.  **Implement dialog stack functions**:

    - `update_dialog_stack(left,`` ``right)`: Push/pop agent transitions

    - `pop_dialog_state(state)`: Return to Primary Assistant

3.  **Create context injection** that auto-populates user info into tool calls

4.  **Configure InMemorySaver checkpointer** for initial development

#### Deliverables:

- `state/agent_state.py` - State definition with all fields

- `state/dialog_stack.py` - Stack management functions

- `state/context_injection.py` - User context injection logic

------------------------------------------------------------------------

### Task 2: Specialized Agents Implementation (25 points)

**Time Allocation**: 120 minutes

Implement all four specialized agents with their tools and schemas.

#### Requirements:

1.  **Ticket Support Agent** (8 points):

    - Define Pydantic schemas: `CreateTicket`, `TrackTicket`, `UpdateTicket`, `CancelTicket`

    - Implement **mock tools** that simulate CRUD operations (return success messages, store in memory dict)

    - Status transitions: Pending → InProgress → Resolved (or Canceled)

    - Add `CompleteOrEscalate` tool for returning to Primary Assistant

    - Tools should accept and validate all required fields

2.  **Booking Agent** (7 points):

    - Define Pydantic schemas with time validation (must be future)

    - Implement **mock tools**: `BookRoom`, `TrackBooking`, `UpdateBooking`, `CancelBooking`

    - Status transitions: Scheduled → Finished (or Canceled)

    - Include `CompleteOrEscalate` tool

3.  **IT Support Agent** (5 points):

    - Integrate Tavily Search with `max_results:`` ``5`, `search_depth:`` ``"advanced"`

    - Return practical troubleshooting guides from reliable sources

    - Include `CompleteOrEscalate` tool

4.  **FAQ Agent** (5 points):

    - Implement simple RAG for FPT policy questions

    - Return answers with source references

    - Include `CompleteOrEscalate` tool

Mock tools should use an in-memory dictionary to store data for testing. This allows the AI system to function independently without database dependencies. The actual database integration will be handled in the FastAPI module exam.

Example mock implementation pattern:

``` python
# In-memory storage for testing
_ticket_store: dict[str, dict] = {}

@tool
def create_ticket(content: str, customer_name: str, customer_phone: str, ...) -> str:
    """Create a new support ticket."""
    ticket_id = str(uuid.uuid4())
    _ticket_store[ticket_id] = {...}
    return f"Ticket created successfully with ID: {ticket_id}"
```

#### Deliverables:

- `agents/ticket_agent.py` - Ticket Support Agent with mock tools

- `agents/booking_agent.py` - Booking Agent with mock tools

- `agents/it_support_agent.py` - IT Support Agent with Tavily

- `agents/faq_agent.py` - FAQ Agent with RAG

- `schemas/` directory with all Pydantic models

------------------------------------------------------------------------

### Task 3: Primary Assistant & Graph Construction (20 points)

**Time Allocation**: 90 minutes

Build the Primary Assistant and construct the complete multi-agent graph.

#### Requirements:

1.  **Define routing tools** for Primary Assistant:

    - `ToTicketAssistant`: Route ticket-related queries

    - `ToBookingAssistant`: Route booking-related queries

    - `ToITAssistant`: Route technical issues

    - `ToFAQAssistant`: Route policy questions

    - Include user context injection in all routing tools

2.  **Implement entry nodes** for agent transitions:

    - Create `create_entry_node(assistant_name)` factory function

    - Entry nodes push new agent to `dialog_state` stack

    - Generate appropriate welcome message

3.  **Build StateGraph** with:

    - Primary Assistant as entry point

    - All specialized agent nodes

    - ToolNode for each agent’s tools

    - Conditional routing based on intent

    - Edge handling for `CompleteOrEscalate`

4.  **Create `tool_node_with_fallback`** for graceful error handling

#### Deliverables:

- `agents/primary_assistant.py` - Primary Assistant with routing

- `graph/entry_nodes.py` - Entry node factory function

- `graph/builder.py` - Complete graph construction

- `graph/routing.py` - Conditional routing logic

- Graph visualization PNG using `get_graph().draw_mermaid_png()`

------------------------------------------------------------------------

### Task 4: Human-in-the-Loop Confirmation (20 points)

**Time Allocation**: 90 minutes

Implement interrupt patterns for sensitive operations.

#### Requirements:

1.  **Configure `interrupt_before`** for sensitive tools:

    - All ticket creation/update/cancel operations

    - All booking creation/update/cancel operations

    - NOT for read operations (track) or search operations

2.  **Implement confirmation flow**:

    - Detect pending tool state via `graph.get_state(config)`

    - Generate human-readable confirmation message

    - Parse user response: “y” to continue, other to cancel

3.  **Create confirmation message generator**:

    - Extract tool name and arguments from pending state

    - Format readable summary for user review

    - Include clear instructions for approval/rejection

4.  **Handle user responses**:

    - “y” or “yes”: Resume execution with `app.invoke(None,`` ``config)`

    - Other: Update state to cancel operation and return message

    - Log all confirmation decisions

#### Deliverables:

- `hitl/interrupt_config.py` - List of sensitive tools

- `hitl/confirmation.py` - Confirmation flow logic

- `hitl/message_generator.py` - Human-readable message formatting

------------------------------------------------------------------------

### Task 5: Response Caching with FAISS (10 points)

**Time Allocation**: 60 minutes

Implement vector store-based caching for RAG and IT Support responses.

#### Requirements:

1.  **Create cache_tool** that:

    - Stores all RAG and IT Support responses in FAISS vectorstore

    - Indexes by query embedding using `sentence-transformers`

    - Stores metadata: timestamp, query_type, source_agent

2.  **Implement cache lookup** in orchestrator:

    - Before calling RAG/IT tools, check cache for similar queries

    - Use similarity threshold (0.85) to determine cache hit

    - Return cached response if found, otherwise proceed to tool

3.  **Add cache management**:

    - TTL-based invalidation (24 hours)

    - Manual cache clear capability

    - Cache statistics logging (hits, misses, hit rate)

#### Deliverables:

- `cache/faiss_cache.py` - FAISS caching implementation

- `cache/cache_manager.py` - Cache management and TTL logic

- `cache/cache_stats.py` - Statistics tracking

------------------------------------------------------------------------

### Task 6: Persistence & Production Readiness (10 points)

**Time Allocation**: 60 minutes

Configure persistent state and production-ready error handling.

#### Requirements:

1.  **Replace InMemorySaver with SQLiteSaver**:

    - Configure persistent storage in `checkpoints.db`

    - Test conversation resumption after process restart

    - Document the migration path to PostgresSaver

2.  **Implement thread management**:

    - List active threads

    - View checkpoint history for a thread

    - Delete old threads (cleanup)

3.  **Add error handling and logging**:

    - Structured logging with conversation context

    - Graceful error recovery for tool failures

    - User-friendly error messages

#### Deliverables:

- `persistence/checkpointer.py` - SQLiteSaver configuration

- `persistence/thread_manager.py` - Thread management utilities

- `utils/logging.py` - Structured logging setup

- `utils/error_handler.py` - Error handling utilities

------------------------------------------------------------------------

## Test Scenarios

Complete these test scenarios to demonstrate system functionality:

### Scenario 1: Multi-Agent Conversation Flow

```
User: "Hi, I need help with a few things"
→ Primary Assistant welcomes user

User: "My laptop won't connect to WiFi"
→ Routes to IT Support Agent
→ Tavily search for troubleshooting
→ Cache response
→ Return to Primary Assistant

User: "I need to book a meeting room for tomorrow 2pm"
→ Routes to Booking Agent
→ Shows confirmation prompt (HITL)
→ User confirms "y"
→ Booking created
→ Return to Primary Assistant
```

### Scenario 2: HITL Rejection Flow

```
User: "Create a support ticket for broken monitor"
→ Routes to Ticket Agent
→ Shows confirmation prompt
→ User rejects with "no, wait"
→ Operation cancelled
→ Agent asks for clarification
```

### Scenario 3: Cache Hit Flow

```
User: "How do I reset my password?" (first time)
→ FAQ Agent answers from RAG
→ Response cached

User: "Password reset instructions?" (similar query)
→ Cache hit detected (similarity > 0.85)
→ Return cached response
```

### Scenario 4: Persistence Test

```
1. Start conversation, create a ticket
2. Stop the process
3. Restart with same thread_id
4. Verify conversation history retained
5. Track the created ticket
```

------------------------------------------------------------------------

## Questions to Answer

Include written responses to these questions in `ANSWERS.md`:

1.  **State Management**: Explain why the `add_messages` reducer is essential for multi-turn conversations. What problems would occur without it?

2.  **Multi-Agent Architecture**: Compare the dialog stack approach vs. flat routing. When would you choose one over the other?

3.  **Human-in-the-Loop Trade-offs**: What are the UX implications of requiring confirmation for every sensitive action? How would you balance security vs. user experience?

4.  **Caching Strategy**: How would you handle cache invalidation when the underlying FAQ documents are updated? Propose a solution.

5.  **Production Considerations**: What additional features would you add before deploying this system to production? Consider: monitoring, scaling, security.

------------------------------------------------------------------------

## Submission Requirements

### Directory Structure

```
fpt-customer-chatbot-ai/
├── agents/
│   ├── primary_assistant.py
│   ├── ticket_agent.py
│   ├── booking_agent.py
│   ├── it_support_agent.py
│   └── faq_agent.py
├── schemas/
│   ├── ticket_schemas.py
│   └── booking_schemas.py
├── state/
│   ├── agent_state.py
│   ├── dialog_stack.py
│   └── context_injection.py
├── tools/
│   ├── ticket_tools.py      # Mock tools for ticket operations
│   ├── booking_tools.py     # Mock tools for booking operations
│   └── mock_store.py        # In-memory storage for testing
├── graph/
│   ├── builder.py
│   ├── entry_nodes.py
│   └── routing.py
├── hitl/
│   ├── interrupt_config.py
│   ├── confirmation.py
│   └── message_generator.py
├── cache/
│   ├── faiss_cache.py
│   ├── cache_manager.py
│   └── cache_stats.py
├── persistence/
│   ├── checkpointer.py
│   └── thread_manager.py
├── utils/
│   ├── logging.py
│   └── error_handler.py
├── data/
│   └── fpt_policies.txt (or .json)
├── main.py
├── requirements.txt
├── README.md
├── ANSWERS.md
└── graph_visualization.png
```

This AI core is designed to be integrated with the FastAPI backend from the **Building Monolith API with FastAPI** module. The mock tools in `tools/` directory can be replaced with actual database operations when integrating.

### Required Deliverables

Complete source code following directory structure

`README.md` with:

- Setup instructions (environment, API keys, dependencies)

- Usage examples and CLI commands

- Architecture diagram or explanation

- Notes on how to integrate with FastAPI backend

`ANSWERS.md` with written responses to all 5 questions

`requirements.txt` with all dependencies

`graph_visualization.png` - Multi-agent graph visualization

Demo video or screenshots showing:

- All four agent flows working

- HITL confirmation workflow

- Cache hit scenario

- Persistence across restart

### Submission Checklist

All code runs without errors

All four specialized agents functional with mock tools

Primary Assistant routes correctly

HITL confirmation works for sensitive operations

Cache stores and retrieves responses

SQLiteSaver enables conversation persistence

Dialog stack tracks agent hierarchy

Context injection auto-populates user info

All test scenarios pass

Documentation is complete

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria | Points | Excellent (100%) | Good (75%) | Needs Improvement (50%) |
|----|----|----|----|----|
| **State Management** (Task 1) | 15 | Perfect messages pattern, dialog stack, injection | Working but minor issues in context handling | Basic state only, missing stack or injection |
| **Specialized Agents** (Task 2) | 25 | All agents with complete tools and validation | Most agents working, some validation missing | Only 1-2 agents functional |
| **Graph Construction** (Task 3) | 20 | Complete graph with all routing and fallbacks | Graph works but missing error handling | Basic graph without proper routing |
| **Human-in-the-Loop** (Task 4) | 20 | Smooth confirmation UX with proper state handling | HITL works but UX needs improvement | Basic interrupt without proper messaging |
| **Response Caching** (Task 5) | 10 | Full caching with TTL and statistics | Caching works but missing TTL or stats | Basic storage without similarity search |
| **Persistence & Production** (Task 6) | 10 | SQLite with thread management and error handling | Persistence works but limited management | InMemorySaver only, no persistence |
| **Total** | **100** |  |  |  |

------------------------------------------------------------------------

## Hints

- Use `state["messages"][-1]` to access the most recent message

- The `add_messages` reducer handles message deduplication automatically

- Store `dialog_state` as a list for stack operations (append/pop)

<!-- -->

- Use `ToolNode(tools).with_fallbacks([...])` for graceful error handling

- The `CompleteOrEscalate` tool should return a flag that routing can detect

- Entry nodes should push to stack, exit nodes should pop

<!-- -->

- Access pending state with `app.get_state(config).next` to see which node is pending

- Use `app.update_state(config,`` ``values)` to modify state before resuming

- Consider timeout handling for user confirmation

<!-- -->

- Use `sentence-transformers/all-MiniLM-L6-v2` for consistent embeddings

- Store original query and response as metadata, not just embedding

- Implement cache warmup for common queries

<!-- -->

- SQLiteSaver requires context manager: `with`` ``SqliteSaver.from_conn_string(...)`` ``as`` ``saver:`

- Thread IDs should be user-meaningful (e.g., `user123-session1`)

- Consider implementing session timeout (24h default)

------------------------------------------------------------------------

## References

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)

- [LangGraph Customer Support Tutorial](https://langchain-ai.github.io/langgraph/tutorials/customer-support/customer-support/)

- [LangGraph Human-in-the-Loop](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/wait-user-input/)

- [LangGraph Checkpointers](https://langchain-ai.github.io/langgraph/how-tos/persistence/)

- [Tavily Search API](https://docs.tavily.com/welcome)

- [FAISS Documentation](https://github.com/facebookresearch/faiss)

- [Pydantic V2 Documentation](https://pydantic.dev/docs/validation/latest/get-started/)
