# Assignment: FPT Customer Chatbot - Multi-Agent System

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | FPT Customer Chatbot - Multi-Agent System |
| **Course** | LangGraph and Agentic AI |
| **Project Name** | `fpt-chatbot-multi-agent` |
| **Estimated Time** | 240 minutes |
| **Framework** | Python 3.11+, LangGraph 1.x, LangChain 1.x, SQLite/PostgreSQL, Tavily API, Anthropic (or OpenAI) |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Design** hierarchical multi-agent architectures with Primary Assistant and specialized agents

- **Implement** dialog stack management for agent transitions

- **Apply** context injection patterns for user information

- **Build** CompleteOrEscalate pattern for agent decision making

- **Create** production-ready tool schemas with Pydantic validation

------------------------------------------------------------------------

## Problem Description

Build a customer service chatbot for FPT with the following capabilities:

1.  **FAQ Agent**: Answer questions about FPT policies using RAG

2.  **Ticket Support Agent**: Create, track, update, cancel support tickets

3.  **IT Support Agent**: Troubleshoot technical issues using Tavily search

4.  **Booking Agent**: Manage meeting room reservations

The Primary Assistant coordinates routing between specialized agents.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langgraph` \>= 1.0

  - `langchain[anthropic]` \>= 1.0 \# or `langchain[openai]`

  - `langchain-tavily` \>= 0.1.0

  - `sqlalchemy` \>= 2.0.0

  - `pydantic` \>= 2.0.0

### Database Schema

Implement the following tables:

- **Ticket**: ticket_id, content, description, customer_name, customer_phone, email (optional), status, time

- **Booking**: booking_id, reason, time, customer_name, customer_phone, email (optional), note, status

------------------------------------------------------------------------

## Tasks

### Task 1: State & Context Management (20 points)

1.  **Define AgenticState** with:

    - `messages`: Conversation history with add_messages

    - `dialog_state`: Stack tracking agent hierarchy

    - `user_id`, `email` (optional): Context injection fields

    - `conversation_id`: Session tracking

2.  **Implement dialog stack functions**:

    - `update_dialog_stack`: Push/pop agent transitions

    - `pop_dialog_state`: Return to Primary Assistant

3.  **Create context injection** that auto-populates user info into tool calls

### Task 2: Ticket Support Agent (25 points)

1.  **Define Pydantic schemas** for:

    - `CreateTicket`: content (required), description, customer_name, phone, email (optional)

    - `TrackTicket`: ticket_id

    - `UpdateTicket`: ticket_id (required), optional fields for updates

2.  **Implement tools**:

    - `create_ticket`: Creates ticket with “Pending” status

    - `track_ticket`: Returns ticket information

    - `update_ticket`: Updates provided fields only

    - `cancel_ticket`: Sets status to “Canceled”

3.  **Add CompleteOrEscalate** tool for returning to Primary Assistant

### Task 3: Booking Agent (20 points)

1.  **Define Pydantic schemas** with validation for:

    - `BookRoom`: reason (required), time (required, must be future)

    - `TrackBooking`, `UpdateBooking`, `CancelBooking`

2.  **Implement booking tools** with status management:

    - Status flow: Scheduled → Finished (or Canceled)

### Task 4: IT Support & FAQ Agents (15 points)

1.  **IT Support Agent**:

    - Uses Tavily search for troubleshooting information

    - Returns practical guides from reliable sources

2.  **FAQ Agent**:

    - Implements RAG for FPT policy questions

    - Returns answers with source references

### Task 5: Primary Assistant & Routing (20 points)

1.  **Define routing tools** for Primary Assistant:

    - `ToTicketAssistant`, `ToITAssistant`, `ToBookingAssistant`

    - Include user context injection in tool calls

2.  **Implement entry nodes** for agent transitions:

    - Create `create_entry_node` factory function

    - Push new agent to dialog_state stack

3.  **Build complete graph** with:

    - All agent nodes and their tool nodes

    - Conditional routing from Primary Assistant

    - Tool fallback handling for errors

4.  **Create tool_node_with_fallback** for graceful error handling

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code with all agents implemented

Database schema (SQLite or PostgreSQL)

`README.md` with setup and usage instructions

Test conversations demonstrating each agent flow

### Submission Checklist

All 4 specialized agents working correctly

Context injection auto-populates email when available

CompleteOrEscalate enables return to Primary Assistant

Tool schemas validate inputs properly

Dialog stack correctly tracks agent hierarchy

Code runs without errors

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                       | Points  |
|--------------------------------|---------|
| State & context management     | 20      |
| Ticket Support Agent           | 25      |
| Booking Agent                  | 20      |
| IT Support & FAQ Agents        | 15      |
| Primary Assistant & routing    | 15      |
| Code quality and documentation | 5       |
| **Total**                      | **100** |

------------------------------------------------------------------------

## Hints

- Use `ToolNode(tools).with_fallbacks([...])` for error handling

- The `dialog_state` stack enables proper agent hierarchy tracking

- Optional email fields should use `Optional[EmailStr]` in Pydantic schemas

- Reference the LangGraph customer support tutorial for architecture patterns

------------------------------------------------------------------------

## References

- [LangGraph Customer Support Tutorial](https://langchain-ai.github.io/langgraph/tutorials/customer-support/customer-support/)

- [Pydantic Field Validation](https://pydantic.dev/docs/validation/latest/concepts/validators/)

- [LangChain Tool Schema](https://docs.langchain.com/oss/python/langchain/overview)
