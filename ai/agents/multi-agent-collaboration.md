# Multi-Agent Collaboration

This page covers how to design systems where multiple specialized agents work together, including common collaboration patterns, orchestration strategies, context injection, and how agents decide when to escalate or hand off to another agent.

## Learning Objectives

- Understand multi-agent system architecture

- Implement agent collaboration patterns

- Design agent teams

- Coordinate agent workflows

- Master context injection and state management

- Understand how agents decide to end or escalate

## Multi-Agent Systems

### Why Multiple Agents?

#### Specialization

Each agent specializes in a specific domain, improving accuracy and efficiency.

#### Parallel processing

Multiple agents can process independent tasks in parallel, reducing processing time.

#### Modularity

System is easier to maintain and extend when each agent is an independent module.

#### Scalability

Easy to add new agents without affecting existing agents.

### Use Cases

#### Software development team

- Product Manager: Define requirements

- Architect: Design system

- Developer: Write code

- Tester: Test and report bugs

#### Customer service

- FAQ Agent: Answer frequently asked questions

- Ticket Agent: Manage support tickets

- IT Support Agent: Technical support

- Booking Agent: Schedule/book rooms

#### Content creation pipeline

- Research Agent: Search for information

- Writer Agent: Write content

- Editor Agent: Edit content

## Collaboration Patterns

### 1. Sequential (Pipeline)

#### Agent 1 → Agent 2 → Agent 3

Output of previous agent is input of next agent, linear workflow.

**Example:**

``` mermaid
        graph LR
    R["Research Agent"] --> A["Analyzer Agent"] --> W["Writer Agent"]
```

### 2. Hierarchical (Supervisor)

#### Primary Assistant → Specialized Agents

Primary assistant coordinates and routes to specialized agents.

**FPT Chatbot Example:**

``` mermaid
        graph TB
    PA["Primary Assistant"] --> FAQ["FAQ Agent (RAG)"]
    PA --> TS["Ticket Support Agent"]
    PA --> IT["IT Support Agent"]
    PA --> BA["Booking Agent"]
```

### 3. Network (Peer-to-Peer)

#### Agents communicate directly

Agents communicate directly with each other, no supervisor needed.

### 4. Competitive

#### Multiple agents solve same problem

Multiple agents solve the same problem, choose the best solution.

## Hierarchical Multi-Agent System

### Architecture

``` mermaid
        graph TB
    PA["Primary Assistant<br/>(Supervisor)"] --> FAQ["FAQ Agent"]
    PA --> TS["Ticket Support Agent"]
    PA --> IT["IT Support Agent"]
    PA --> BA["Booking Agent"]
```

### State Management in Hierarchical System

#### Each Agent has its own State

In hierarchical systems, each agent can have its own state, but all share a common **shared state**.

**State Definition:**

``` python
from typing import Annotated, Optional, List

from typing_extensions import TypedDict, Literal

from langchain.messages import AnyMessage

from langgraph.graph import add_messages

from pydantic import EmailStr



class AgenticState(TypedDict):

    """Shared state for all agents"""



    # Messages - automatically merged

    messages: Annotated[list[AnyMessage], add_messages]



    # Dialog stack - track agent hierarchy

    dialog_state: Annotated[

        list[Literal["primary_assistant", "ticket_agent", "it_agent", "booking_agent"]],

        update_dialog_stack,

    ]



    # Context information

    conversation_id: Annotated[str, "Conversation ID"]

    user_id: Annotated[str, "User ID"]

    email: Annotated[Optional[EmailStr], "Email from context (optional)"]
```

#### Dialog Stack Management

**Push State (Switch to child agent):**

``` python
def update_dialog_stack(left: list[str], right: Optional[str]) -> list[str]:

    """Push or pop dialog stack"""

    if right is None:

        return left

    if right == "pop":

        return left[:-1]  # Pop: return to previous agent

    return left + [right]  # Push: add new agent to stack
```

**Dialog Stack Example:**

``` python
# Start

dialog_state = ["primary_assistant"]



# Switch to Ticket Agent

dialog_state = ["primary_assistant", "ticket_agent"]



# Pop (return to Primary)

dialog_state = ["primary_assistant"]
```

#### Pop State (Return to Primary Assistant)

``` python
def pop_dialog_state(state: AgenticState) -> dict:

    """Pop dialog stack and return to primary assistant"""

    messages = []

    if state["messages"][-1].tool_calls:

        messages.append(

            ToolMessage(

                content="Resuming dialog with the host assistant. "

                       "Please reflect on the past conversation and assist the user as needed.",

                tool_call_id=state["messages"][-1].tool_calls[0]["id"],

            )

        )

    return {

        "dialog_state": "pop",  # Trigger pop operation

        "messages": messages,

    }
```

### Context Injection

#### Why Context Injection?

Context injection allows automatic injection of user information (email, user_id) into all tool calls without agents needing to remember and pass it each time.

#### Context Injection Implementation

**1. Store Context in State:**

``` python
class AgenticState(TypedDict):

    # ... other fields

    user_id: Annotated[str, "User ID"]

    email: Annotated[Optional[EmailStr], "Email from context (optional)"]
```

**2. Inject Context into Tool Calls:**

``` python
def inject_user_info(state: AgenticState, result):

    """Automatically inject user_id and email into tool calls"""

    if hasattr(result, "tool_calls") and result.tool_calls:

        for tool_call in result.tool_calls:

            # Inject into tool calls that need user context

            if tool_call["name"] in [

                "ToTicketAssistant",

                "ToITAssistant",

                "ToBookingAssistant"

            ]:

                # Automatically add user_id and email from state

                tool_call["args"]["user_id"] = state["user_id"]

                if state.get("email"):

                    tool_call["args"]["email"] = state["email"]

    return result
```

**3. Use in Primary Assistant:**

``` python
def assistant_runnable_with_user_info(state):

    """Primary assistant with context injection"""

    # Bind tools for primary assistant

    result = (

        primary_assistant_prompt

        | llm.bind_tools([

            ToTicketAssistant,

            ToITAssistant,

            ToBookingAssistant,

            RAG_Agent

        ])

    ).invoke(state)



    # Inject user info into tool calls

    return inject_user_info(state, result)
```

**4. Tool Schema with Optional Email:**

``` python
from pydantic import BaseModel, EmailStr

from typing import Optional



class CreateTicket(BaseModel):

    """Schema for create_ticket tool"""

    content: str

    description: Optional[str] = None

    customer_name: Optional[str] = None

    customer_phone: Optional[str] = None

    # Email is OPTIONAL - can be injected from context

    email: Optional[EmailStr] = None

    # user_id will be automatically injected

    user_id: Optional[str] = None
```

**5. Tool Implementation using Context:**

``` python
@tool("create_ticket", args_schema=CreateTicket)

def create_ticket(

    content: str,

    description: Optional[str] = None,

    customer_name: Optional[str] = None,

    customer_phone: Optional[str] = None,

    email: Optional[EmailStr] = None,  # Optional - can be from context

    user_id: Optional[str] = None,  # Will be automatically injected

) -> dict:

    """Create new ticket"""

    # Email may have been injected from context

    # If not available, tool still works normally

    ticket_id = f"TICKET_{generate_short_id()}"



    new_ticket = Ticket(

        ticket_id=ticket_id,

        content=content,

        description=description,

        customer_name=customer_name,

        customer_phone=customer_phone,

        email=email,  # Can be None

        user_id=user_id,

        status="Pending"

    )



    # Save to database

    db.add(new_ticket)

    db.commit()



    # Only send email if email exists

    if email:

        send_email_notification(email, ticket_id)



    return {

        "ticket_id": ticket_id,

        "message": f"Ticket created successfully. Email sent to {email}" if email else "Ticket created successfully."

    }
```

### State Transitions

#### Entry Node (Enter Child Agent)

``` python
def create_entry_node(assistant_name: str, new_dialog_state: str):

    """Create entry node to switch to child agent"""



    def entry_node(state: AgenticState) -> dict:

        tool_call_id = state["messages"][-1].tool_calls[0]["id"]



        return {

            "messages": [

                ToolMessage(

                    content=f"The assistant is now the {assistant_name}. "

                           f"Reflect on the above conversation. "

                           f"Use the provided tools to assist the user. "

                           f"If task is complete, call CompleteOrEscalate to return control.",

                    tool_call_id=tool_call_id,

                )

            ],

            "dialog_state": new_dialog_state,  # Push to stack

        }



    return entry_node
```

#### Routing Logic

``` python
def route_primary_assistant(state: AgenticState):

    """Route from primary assistant to appropriate agent"""

    last_message = state["messages"][-1]



    if not hasattr(last_message, "tool_calls") or not last_message.tool_calls:

        return END



    tool_name = last_message.tool_calls[0]["name"]



    # Route based on called tool

    routing_map = {

        "ToTicketAssistant": "enter_ticket_node",

        "ToITAssistant": "enter_it_node",

        "ToBookingAssistant": "enter_booking_node",

        "rag_agent": "rag_agent_node",

    }



    return routing_map.get(tool_name, END)
```

#### Graph Construction

``` python
def setup_agentic_graph():

    """Create graph with nodes and edges"""

    builder = StateGraph(AgenticState)



    # Primary Assistant

    builder.add_node("primary_assistant", Assistant(assistant_runnable, "Primary Assistant"))



    # Ticket Agent nodes

    builder.add_node("enter_ticket_node", create_entry_node("Ticket Agent", "ticket_agent"))

    builder.add_node("call_ticket_agent", Assistant(ticket_runnable, "Ticket Agent"))

    builder.add_node("ticket_tools", create_tool_node_with_fallback(ticket_tools))



    # IT Agent nodes

    builder.add_node("enter_it_node", create_entry_node("IT Agent", "it_agent"))

    builder.add_node("call_it_agent", Assistant(it_runnable, "IT Agent"))

    builder.add_node("it_tools", create_tool_node_with_fallback(it_tools))



    # Booking Agent nodes

    builder.add_node("enter_booking_node", create_entry_node("Booking Agent", "booking_agent"))

    builder.add_node("call_booking_agent", Assistant(booking_runnable, "Booking Agent"))

    builder.add_node("booking_tools", create_tool_node_with_fallback(booking_tools))



    # Pop state node

    builder.add_node("leave_skill", pop_dialog_state)



    # Edges

    builder.add_edge(START, "primary_assistant")

    builder.add_edge("enter_ticket_node", "call_ticket_agent")

    builder.add_edge("ticket_tools", "call_ticket_agent")

    builder.add_edge("enter_it_node", "call_it_agent")

    builder.add_edge("it_tools", "call_it_agent")

    builder.add_edge("enter_booking_node", "call_booking_agent")

    builder.add_edge("booking_tools", "call_booking_agent")

    builder.add_edge("leave_skill", "primary_assistant")



    # Conditional routing

    builder.add_conditional_edges(

        "primary_assistant",

        route_primary_assistant,

        ["enter_ticket_node", "enter_it_node", "enter_booking_node", "rag_agent_node", END]

    )



    builder.add_conditional_edges(

        "call_ticket_agent",

        route_ticket_agent,

        ["ticket_tools", "leave_skill", END]

    )



    return builder.compile(checkpointer=saver)
```

### Tool Call Fallback

#### Why Fallback?

When tool call fails (network error, database error, validation error), system needs fallback mechanism to handle gracefully.

#### Fallback Implementation

``` python
def create_tool_node_with_fallback(tools: list):

    """Create tool node with fallback handling"""



    def log_tool_result(state):

        """Wrapper to log and handle errors"""

        try:

            # Execute tool

            result = ToolNode(tools).invoke(state)

            return result

        except Exception as e:

            # Fallback: return error message

            last_message = state.get("messages", [])[-1]

            tool_call_id = "unknown"

            if last_message and hasattr(last_message, "tool_calls"):

                tool_call_id = last_message.tool_calls[0]["id"]



            return {

                "messages": [

                    ToolMessage(

                        content=f"Error executing tool: {str(e)}. Please try again or contact support.",

                        tool_call_id=tool_call_id

                    )

                ]

            }



    # Use with_fallbacks from LangChain

    tool_node = RunnableLambda(log_tool_result)



    return tool_node.with_fallbacks(

        [RunnableLambda(lambda state: {

            "messages": [

                ToolMessage(

                    content="Tool execution failed. Please try again.",

                    tool_call_id=state["messages"][-1].tool_calls[0]["id"]

                    if state.get("messages") and state["messages"][-1].tool_calls else "unknown"

                )

            ]

        })],

        exception_key="error"

    )
```

### Complete or Escalate Pattern

#### When should Agent End?

Agent should end (END) when:

1.  Task completed successfully

2.  Fully answered user’s question

3.  No additional information needed from user

#### When should Agent Escalate?

Agent should escalate (return to Primary Assistant) when:

1.  Task outside agent’s scope

2.  Need capabilities only available in Primary Assistant

3.  User changes mind or cancels task

4.  Need clarification from user but cannot handle independently

#### Implementation CompleteOrEscalate

**1. Define Schema:**

``` python
class CompleteOrEscalate(BaseModel):

    """Tool for agent to decide end or escalate"""



    cancel: bool = True  # True = escalate, False = continue

    reason: str  # Reason



    class Config:

        json_schema_extra = {

            "example": {

                "cancel": True,

                "reason": "Task completed successfully. Returning to primary assistant."

            },

            "example 2": {

                "cancel": True,

                "reason": "User changed their mind. Need to return to primary assistant."

            },

            "example 3": {

                "cancel": False,

                "reason": "Need more information from user to complete the task."

            }

        }
```

**2. Agent uses CompleteOrEscalate:**

``` python
# Ticket Agent has tools

ticket_safe_tools = [track_ticket]  # Read-only

ticket_sensitive_tools = [create_ticket, update_ticket, cancel_ticket]  # Write operations

ticket_tools = ticket_safe_tools + ticket_sensitive_tools + [CompleteOrEscalate]



def create_ticket_tool(model):

    """Create ticket agent with tools"""

    ticket_tools_runnable = (

        ticket_assistant_prompt

        | model.bind_tools(ticket_tools + [CompleteOrEscalate])

    )

    return ticket_tools_runnable
```

**3. Routing Logic:**

``` python
def route_ticket_agent(state: AgenticState):

    """Route for ticket agent"""

    route = tools_condition(state)

    if route == END:

        return END



    tool_calls = state["messages"][-1].tool_calls

    tool_names = [tc.get("name") for tc in tool_calls]



    # If calling CompleteOrEscalate with cancel=True -> return to primary

    if "CompleteOrEscalate" in tool_names:

        for tc in tool_calls:

            if tc["name"] == "CompleteOrEscalate":

                args = tc.get("args", {})

                if args.get("cancel", True):

                    return "leave_skill"  # Pop state



    # Distinguish safe tools and sensitive tools

    safe_toolnames = [t.name for t in ticket_safe_tools]

    if all(tc["name"] in safe_toolnames for tc in tool_calls):

        return "ticket_safe_tools"



    return "ticket_sensitive_tools"
```

**4. Agent Prompt usage guide:**

``` python
TICKET_SYSTEM_PROMPT = """

You are a Ticket Support Agent for FPT Customer Service.



Your responsibilities:

- Create tickets for IT or customer support issues

- Track ticket status

- Update ticket information

- Cancel tickets when requested



When to use CompleteOrEscalate:

1. Task completed successfully -> cancel=True, reason="Task completed"

2. User changed their mind -> cancel=True, reason="User changed mind"

3. Question outside your scope -> cancel=True, reason="Out of scope"

4. Need more info but can't proceed -> cancel=False, reason="Need clarification"



Always try to complete the task before escalating.

"""
```

### Tool Arg Schema in Multi-Tool System

#### Why Tool Arg Schema?

Tool arg schema helps:

- Validate input before executing tool

- Clearly document parameters

- Type safety

- Auto-completion in IDE

#### Pydantic Schema for Tools

**1. Basic Schema:**

``` python
from pydantic import BaseModel, EmailStr, Field

from typing import Optional, Annotated



class CreateTicket(BaseModel):

    """Schema for create_ticket tool"""



    content: str = Field(..., description="Ticket content")

    description: Optional[str] = Field(None, description="Detailed description")

    customer_name: Optional[str] = Field(None, description="Customer name")

    customer_phone: Optional[str] = Field(None, description="Phone number")

    email: Optional[EmailStr] = Field(None, description="Email (optional, can be injected from context)")

    user_id: Optional[str] = Field(None, description="User ID (will be automatically injected)")



    class Config:

        json_schema_extra = {

            "example": {

                "content": "Laptop screen is broken",

                "description": "Screen repair needed",

                "customer_name": "John Doe",

                "customer_phone": "0123456789",

                "email": "[email protected]"

            }

        }
```

**2. Schema with Validation:**

``` python
from pydantic import field_validator



class BookRoom(BaseModel):

    """Schema for book_room tool"""



    reason: str = Field(..., description="Reason for booking")

    time: datetime = Field(..., description="Booking time")

    customer_name: Optional[str] = None

    customer_phone: Optional[str] = None

    email: Optional[EmailStr] = None

    note: Optional[str] = Field(None, max_length=500)



    @field_validator("time")

    @classmethod

    def validate_time(cls, v):

        if v < datetime.now():

            raise ValueError("Booking time must be in the future")

        return v



    @field_validator("customer_phone")

    @classmethod

    def validate_phone(cls, v):

        if v and not v.isdigit():

            raise ValueError("Phone number must contain only digits")

        return v
```

**3. Schema with Literal Types:**

``` python
from typing import Literal



class UpdateTicket(BaseModel):

    """Schema for update_ticket tool"""



    ticket_id: str = Field(..., description="Ticket ID")

    content: Optional[str] = None

    description: Optional[str] = None

    status: Optional[Literal["Pending", "Resolving", "Finished", "Canceled"]] = None

    email: Optional[EmailStr] = None
```

**4. Use Schema in Tool:**

``` python
from langchain_core.tools import tool



@tool("create_ticket", args_schema=CreateTicket)

def create_ticket(

    content: str,

    description: Optional[str] = None,

    customer_name: Optional[str] = None,

    customer_phone: Optional[str] = None,

    email: Optional[EmailStr] = None,

    user_id: Optional[str] = None,

) -> dict:

    """Create new ticket"""

    # Implementation...

    pass
```

**5. Schema for Multi-Tool System:**

``` python
# Each agent has its own tool schemas

ticket_tool_schemas = [CreateTicket, UpdateTicket, TrackTicket, CancelTicket]

it_tool_schemas = [TavilySearch]  # IT agent only has 1 tool

booking_tool_schemas = [BookRoom, UpdateBooking, TrackBooking, CancelBooking]



# Primary assistant has routing tools

primary_tool_schemas = [

    ToTicketAssistant,

    ToITAssistant,

    ToBookingAssistant,

    RAG_Agent

]
```

## Communication Patterns

### Message Passing

#### Structured Messages

``` python
{

    "from": "primary_assistant",

    "to": "ticket_agent",

    "type": "tool_call",

    "tool": "ToTicketAssistant",

    "args": {

        "user_id": "user123",

        "email": "[email protected]"

    }

}
```

#### State Updates

Each agent updates state through messages:

``` python
# Agent returns message

return {

    "messages": [

        AIMessage(

            content="Ticket created successfully",

            tool_calls=[...]

        )

    ]

}
```

### Shared Memory

#### Common Knowledge Base

All agents share:

- Conversation context (email, user_id)

- Dialog stack

- Messages history

#### State Accessibility

``` python
# Agent can access entire state

def ticket_agent(state: AgenticState):

    # Access context

    user_id = state["user_id"]

    email = state.get("email")

    conversation_id = state["conversation_id"]



    # Access messages

    messages = state["messages"]



    # Access dialog stack

    dialog_state = state["dialog_state"]
```

## Error Handling

### Retry logic

``` python
@retry(max_attempts=3, backoff=exponential_backoff)

def create_ticket_with_retry(...):

    return create_ticket(...)
```

### Graceful degradation

``` python
def create_ticket_graceful(...):

    try:

        # Try full flow

        return create_ticket(...)

    except DatabaseError:

        # Fallback: only save to memory

        return {"ticket_id": "temp_123", "status": "pending_save"}
```

### Communication Failure

When agent cannot communicate, system will:

1.  Log error

2.  Return error message to user

3.  Optionally retry or escalate

### Caching

Cache tool call results to avoid repeated calls:

``` python
from functools import lru_cache



@lru_cache(maxsize=100)

def get_ticket_status(ticket_id: str):

    return db.query(Ticket).filter(Ticket.ticket_id == ticket_id).first()
```

## Multi-Agent Pattern Comparison

### Detailed Comparison Table

| **Criteria** | **ReAct Agent** | **Hierarchical Multi-Agent** | **Supervisor Pattern** |
|----|----|----|----|
| **Architecture** | Single agent with reasoning loop | Multiple specialized agents with hierarchy | Central supervisor routing to specialists |
| **Number of Agents** | 1 agent | 3-10+ agents | 1 supervisor + N workers |
| **Complexity** | Low - Simple | High - Complex | Medium |
| **State Management** | Simple state | Shared state + Dialog stack | Centralized state |
| **Reasoning** | Step-by-step with tools | Distributed reasoning | Supervisor decides routing |
| **Tool Access** | All tools available | Tools distributed by agent | Tools grouped by specialist |
| **Scalability** | Limited (1 agent) | Good (easy to add agents) | Good (add specialists) |
| **Latency** | Low | Medium-High | Medium |
| **Cost** | Low | High | Medium-High |
| **Use Cases** | Simple tasks, Q&A | Complex workflows, enterprises | Multi-domain support |
| **Best For** | \- FAQ answering- Simple automation- Single-domain tasks | \- Enterprise systems- Multi-step workflows- Department coordination | \- Customer service- Multi-category support- Triage systems |

## PRACTICE:

## FPT Customer Chatbot System - Multi-Agent System

## 1. System overview

FPT customer chatbot system is designed with hierarchical multi-agent architecture, allowing users to interact with multiple specialized agents through a primary assistant. The system supports: FAQ about FPT policies, support ticket management, IT support, and meeting room booking.

**REFERENCE**: <https://langchain-ai.github.io/langgraph/tutorials/customer-support/customer-support/>

*REQUIREMENT: REFERENCE THIS LINK*

## 2. Context Injection

### 2.1. User Information

- System must support automatic context injection from user information

- Users **can** provide **email** when starting conversation (OPTIONAL)

- This information is automatically injected into all related tool calls if provided

- Email is stored in conversation context and can be used by all agents

### 2.2. Context Management

- Context is maintained throughout the entire conversation

- Child agents can access and use context information from primary assistant

- Context can be updated during conversation

## 3. FAQ Agent - RAG Tool for FPT Policy

### 3.1. Functionality

- Answer questions about FPT policies and regulations

- Search information from knowledge base about FPT-related issues

- Support multiple languages (Vietnamese, English)

### 3.2. Requirements

- Agent must be able to understand questions and search for related information

- Results must be accurate and have reference sources

- Support semantic search to understand user intent

## 4. Ticket Support Agent

### 4.1. Functionality

Ticket Support Agent handles requests to create, track and update support tickets for IT or customer support issues.

### 4.2. Tools

#### 4.2.1. Create Ticket

- **Purpose**: Create new ticket for IT or customer support issues

- **Required information**:

  - Ticket content (content)

  - Detailed description (description)

- **Optional information**:

  - Email (automatically injected from context if available)

  - Customer name (customer_name)

  - Phone number (customer_phone)

- **Result**: Create ticket with “Pending” status and return ticket_id

#### 4.2.2. Track Ticket

- **Purpose**: Track status and information of ticket

- **Input**: ticket_id

- **Output**: Complete ticket information including:

  - ticket_id

  - content

  - description

  - status (Pending, Resolving, Canceled, Finished)

  - customer_name

  - customer_phone

  - email (if available)

  - creation time

  - user_id

#### 4.2.3. Update Ticket

- **Purpose**: Update information of existing ticket

- **Required input**: ticket_id

- **Optional input**:

  - content

  - description

  - customer_name

  - customer_phone

  - email (automatically injected from context if available)

- **Rule**: Only update provided fields, keep other fields unchanged

### 4.3. Ticket Status Management

- **Status**: Pending → Resolving → Finished

- **Special status**: Canceled (can be set from any status before Finished)

## 5. IT Support Agent

### 5.1. Functionality

IT Support Agent helps users resolve technical issues with computers and electronic devices through Tavily search tool.

### 5.2. Tool: Tavily Search

- **Purpose**: Search for troubleshooting information about computers and electronic devices

- **Input**: Question about technical issues

- **Output**:

  - Troubleshooting guide information

  - Suggested solutions

  - Reference links from reliable sources

- **Requirements**:

  - Search results must be from reliable technology sources

  - Support search in Vietnamese and English

  - Return practical and easy-to-understand information

## 6. Booking Agent

### 6.1. Functionality

Booking Agent handles meeting room booking for users, similar to Appointment Agent but specifically for room booking.

### 6.2. Tools

#### 6.2.1. Book Room

- **Purpose**: Book meeting room

- **Required information**:

  - Reason for booking (reason)

  - Time (time)

- **Optional information**:

  - Email (automatically injected from context if available)

  - Booker name (customer_name)

  - Phone number (customer_phone)

  - Note (note)

- **Result**: Create booking with “Scheduled” status and return booking_id

#### 6.2.2. Track Booking

- **Purpose**: Track booking information and status

- **Input**: booking_id

- **Output**: Complete booking information

#### 6.2.3. Update Booking

- **Purpose**: Update booking information

- **Required input**: booking_id

- **Optional input**: reason, time, customer_name, customer_phone, note, email

- **Rule**: Only update provided fields

#### 6.2.4. Cancel Booking

- **Purpose**: Cancel booking

- **Input**: booking_id

- **Rule**: Can only cancel unfinished bookings

### 6.3. Booking Status Management

- **Status**: Scheduled → Finished

- **Special status**: Canceled

## 7. Primary Assistant

### 7.1. Functionality

Primary Assistant is the main agent that coordinates and routes requests to appropriate specialized agents.

### 7.2. Routing Requirements

- Analyze user intent

- Route to FAQ Agent when questions about FPT policies

- Route to Ticket Support Agent when user wants to create/track/update ticket

- Route to IT Support Agent when user has troubleshooting issues

- Route to Booking Agent when user wants to book room

- Support returning to primary assistant after completing task in child agent

## 8. Database Schema

### 8.2. Ticket Table

Store support ticket information:

- `ticket_id` (Primary Key): Unique ticket ID

- `content`: Ticket content

- `description`: Detailed description

- `customer_name`: Customer name

- `customer_phone`: Phone number

- `email`: Email (optional, can be null)

- `time`: Ticket creation time

- `status`: Status (Pending, Resolving, Canceled, Finished)

### 8.3. Booking Table

Store booking information:

- `booking_id` (Primary Key): Unique booking ID

- `customer_name`: Booker name

- `customer_phone`: Phone number

- `email`: Email (optional, can be null)

- `reason`: Reason for booking

- `time`: Booking time

- `note`: Note (optional)

- `status`: Status (Scheduled, Canceled, Finished)

### 8.4. Conversation Context Table

Store conversation context:

- `conversation_id` (Primary Key): Unique conversation ID

- `user_id` (Foreign Key): Reference to Customer Info

- `email`: Email from context (optional)

- `created_at`: Conversation creation time

- `updated_at`: Last update time

## 9. Business Logic Requirements

### 9.1. Ticket Management

- New tickets always have “Pending” status

- Can only update tickets not in “Finished” or “Canceled” status

- Email is **OPTIONAL**, can be injected from context or set to null

- System still creates ticket successfully even without email

### 9.2. Booking Management

- New bookings always have “Scheduled” status

- Can only cancel bookings not in “Finished” status

- Email is **OPTIONAL**, can be injected from context or set to null

- System still creates booking successfully even without email

### 9.3. Context Management

- Email is stored in conversation context when user provides it

- Tools automatically use email from context if not provided directly (if available in context)

- Users can override email in tool call if desired

### 9.4. Agent Communication

- Primary assistant can transfer control to child agent

- Child agent can return to primary assistant after completing task

- Context is shared between all agents in the same conversation

## 10. ADVANCED - Advanced Features (OPTIONAL)

### 10.1. Email Notification (OPTIONAL)

- **Feature**: Send email notification to users when there are changes to ticket or booking

- **Requirements**:

  - Email is **completely OPTIONAL** in all tools related to ticket and booking

  - System only sends email if user provides email in context or in tool call

  - If no email, system still works normally and does not send notification

  - Email can be automatically injected from context if previously provided

  - Users can override email in each tool call if they want to use different email

### 10.2. Human-in-the-Loop Confirmation (OPTIONAL)

- **Feature**: User confirmation before performing important operations

- **Requirements**:

  - Human-in-the-loop confirmation is **completely OPTIONAL**

  - System can operate with or without confirmation

  - If enabled, system will request confirmation before:

    - Creating new ticket

    - Creating new booking

    - Canceling ticket or booking

    - Updating important information

  - If not enabled, system will perform operations directly

  - Confirmation can be configured at conversation or tool level

### 10.3. Notes on Advanced Features

- All features in ADVANCED section are **OPTIONAL**

- System must function fully even without email or human-in-the-loop confirmation

- Implementation of these features does not affect core system functionality

- Users can use system without providing email or without confirmation

## 12. Workflow

### 12.1. Ticket Creation Flow

1.  User can provide email (OPTIONAL - first time)

2.  Primary assistant receives ticket creation request

3.  Route to Ticket Support Agent

4.  Ticket Support Agent uses create_ticket tool

5.  Tool automatically injects email from context (if available in context)

6.  Create ticket in database (with or without email)

7.  Return ticket_id to user

### 12.2. Booking Flow

1.  User requests room booking

2.  Primary assistant routes to Booking Agent

3.  Booking Agent collects information (reason, time)

4.  Use book_room tool with email from context (if available in context)

5.  Create booking in database (with or without email)

6.  Return booking_id to user

### 12.3. FAQ Flow

1.  User asks about FPT policies

2.  Primary assistant routes to FAQ Agent

3.  FAQ Agent uses RAG tool to search

4.  Return information from knowledge base

### 12.4 IT Support Flow

1.  User asks about technical issues

2.  Primary assistant routes to IT Support Agent

3.  IT Support Agent uses Tavily tool to search

4.  Return troubleshooting guide

## Multi-Agent Orchestration in 2026 NEW

### Dominant Orchestration Topologies

Four patterns have emerged as the dominant ways to structure multi-agent systems in production:

``` mermaid
        graph TB
    subgraph "Supervisor (Hub-and-Spoke)"
        SUP["Supervisor"] --> W1["Agent A"]
        SUP --> W2["Agent B"]
        SUP --> W3["Agent C"]
    end
    subgraph "Hierarchical"
        DIR["Director"] --> S1["Sub-Supervisor 1"]
        DIR --> S2["Sub-Supervisor 2"]
        S1 --> WA["Worker A"]
        S1 --> WB["Worker B"]
        S2 --> WC["Worker C"]
    end
    subgraph "Swarm (Peer-to-Peer)"
        PA["Agent A"] <--> PB["Agent B"]
        PB <--> PC["Agent C"]
        PA <--> PC
    end
    subgraph "Pipeline"
        P1["Stage 1"] --> P2["Stage 2"] --> P3["Stage 3"]
    end
```

**Supervisor (Hub-and-Spoke)**: The most common pattern in production deployments. A central orchestrator receives all tasks and routes them to the appropriate specialist agent. This topology is preferred where auditability and clear task attribution are required.

**Hierarchical**: Designed for systems with ten or more agents where a single supervisor would become a bottleneck. A director delegates broad objectives to sub-supervisors, each managing a cluster of workers. This pattern is used by large-scale coding agents such as SWE-bench top performers.

**Swarm (Peer-to-Peer)**: Agents communicate directly with each other through a shared memory or message bus, with no central controller. This topology offers better fault tolerance since no single node is a single point of failure, but the lack of a central authority makes tracing failures and debugging significantly harder.

**Pipeline**: Tasks flow sequentially through fixed stages, with each stage consuming the output of the previous one. Stages can be replayed independently on failure, making this pattern well-suited for content processing workflows such as ingestion, enrichment, and publication.

### Framework Comparison (2026)

| Framework | Stars | Strength | Best For |
|----|----|----|----|
| LangGraph | 14k+ | State management, production-ready | Complex stateful agents |
| CrewAI | 45k+ | Role-based crews, fast prototyping | Team-based workflows |
| AutoGen/AG2 | 43k+ | Rich conversation patterns | Research, group chat |
| Pydantic AI | 8k+ | Type-safe, FastAPI-style | Structured output agents |
| DSPy | 22k+ | Programmatic prompt optimization | Teams with eval data |

Each framework reflects a different design philosophy. LangGraph and Pydantic AI prioritize reliability and type safety for production systems. CrewAI and AutoGen optimize for developer velocity and expressive agent roles. DSPy takes a data-driven approach where prompts are treated as learnable parameters rather than hand-written strings.

### Agent-to-Agent Communication

Two protocols now underpin most inter-agent communication in production systems:

- **MCP (Model Context Protocol)**: Originally introduced by Anthropic, MCP standardizes how agents connect to external tools and data sources. As of 2025 it is governed by the Linux Foundation, giving it broad industry backing. MCP is the dominant standard for tool invocation across agent frameworks.

- **A2A (Agent-to-Agent)**: A newer protocol specifically designed for direct communication between autonomous agents rather than between an agent and a tool. A2A is still maturing and adoption is growing alongside MCP rather than replacing it.

Most production systems in 2026 use hybrid topologies: the supervisor pattern at the top level for task routing and auditability, with pipeline sub-workflows handling repetitive, multi-stage processing tasks within each agent’s scope.
