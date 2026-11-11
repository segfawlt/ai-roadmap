# Human-in-the-Loop & Persistence

This page covers how to integrate human review and approval steps into LangGraph agent workflows, and how to persist agent state across interruptions using checkpointing and memory so long-running tasks can be safely paused and resumed.

## Learning Objectives

- Implement human approval workflows

- Add breakpoints in agent flow

- Persist agent state

- Implement memory and checkpointing

### Why HITL is Needed

#### High-stakes decisions

- **Critical operations**: Decisions with major impact like data deletion, system configuration changes

- **Financial implications**: Money transactions, payments, transfers

- **Legal consequences**: Actions that may violate laws or terms of service

- **Irreversible actions**: Operations that cannot be undone (permanent deletions, account closures)

#### Quality control

- **Content accuracy**: Ensure generated information is accurate, no hallucinations

- **Brand consistency**: Check content aligns with brand guidelines

- **Tone and style**: Verify tone is appropriate for audience

- **Error detection**: Catch logic errors, grammar mistakes, or factual errors before publishing

#### Compliance requirements

- **Regulatory mandates**: Comply with GDPR, HIPAA, SOC 2, ISO 27001

- **Industry standards**: Meet sector standards like PCI-DSS for payments

- **Audit trails**: Record who approved what and when for audit purposes

- **Data protection**: Ensure sensitive data is handled correctly

#### User feedback

- **Continuous improvement**: Collect feedback to improve model

- **Personalization**: Learn user preferences over time

- **Trust building**: Allowing user control increases trust

- **Error correction**: User can fix mistakes and teach agent the correct way

### Use Cases

#### Content approval before publishing

- **Blog posts & articles**: Review tone, SEO, accuracy before posting

- **Social media posts**: Check brand voice, compliance with platform policies

- **Marketing emails**: Verify personalization, links, unsubscribe options

- **Product descriptions**: Ensure product information is accurate and complete

#### Financial transactions

- **Large payments**: Require approval for transactions above certain threshold

- **Refunds & chargebacks**: Human review before processing

- **Investment decisions**: Verify trades, portfolio changes

- **Budget allocations**: Approve spending across different categories

#### Medical recommendations

- **Diagnosis suggestions**: Doctor must review and approve AI diagnosis

- **Treatment plans**: Verify medication, dosage, contraindications

- **Lab result interpretation**: Human expert confirms AI analysis

- **Emergency protocols**: Critical decisions always require human oversight

#### Legal advice

- **Contract review**: Lawyer verifies AI-generated contract clauses

- **Legal research**: Confirm case law citations and interpretations

- **Compliance checks**: Validate regulatory compliance assessments

- **Risk assessments**: Human lawyer approves risk evaluations

#### Code deployment

- **Production releases**: Senior dev approves code before deployment

- **Database migrations**: Review schema changes, data transformations

- **Infrastructure changes**: Approve terraform/cloudformation changes

- **Security patches**: Verify patches don’t break existing functionality

## Interrupts in LangGraph

### Concept

#### Pause execution

- **Controlled stopping**: Stop graph execution at predetermined point

- **State preservation**: All state is saved for later resumption

- **Deterministic behavior**: Always pause at same node/condition

- **No data loss**: No computation already performed is lost

#### Wait for human input

- **Blocking operation**: Graph doesn’t continue until input received

- **Timeout handling**: Can set timeout to avoid infinite waiting

- **Input validation**: Check input from human before proceeding

- **Context provision**: Provide full context for human to make decision

#### Resume from checkpoint

- **Exact state restoration**: Restore exact state at pause moment

- **Continue execution**: Continue from next node after interrupt

- **State updates**: Can update state before resuming

- **Multiple resumes**: Can pause and resume multiple times in one workflow

### Adding Interrupts

``` python
from langgraph.checkpoint.memory import InMemorySaver

workflow = StateGraph(AgentState)
# Add nodes...

# Compile with checkpointer - REQUIRED for interrupts
memory = InMemorySaver()
app = workflow.compile(
    checkpointer=memory,  # Enables state persistence
    interrupt_before=["approval_node"],  # Pause before these nodes
    interrupt_after=["generate_content"]  # Pause after these nodes
)
```

### Interrupt Nodes

``` python
# Method 1: Global interrupt at compile time
app = workflow.compile(
    checkpointer=memory,
    interrupt_before=["human_approval", "deployment"]
)

# Method 2: Conditional interrupt in node
def check_needs_approval(state):
    if state["amount"] > 10000:
        return "approval_node"
    return "auto_process"

workflow.add_conditional_edges(
    "process_transaction",
    check_needs_approval
)
```

### Execution Flow

#### 1. Run until interrupt

- Graph executes nodes normally

- When interrupt point is reached, stops

- Returns control to caller

- State is checkpointed automatically

#### 2. Save checkpoint

- All state is serialized

- Checkpoint ID is generated

- Metadata is saved (timestamp, thread_id, node_name)

- Saved to configured checkpointer (memory/sqlite/postgres)

#### 3. Return control

- Graph execution ends temporarily

- Returns checkpoint info to caller

- Application can show UI for human review

- State can be queried to display info

#### 4. Human reviews

- Human views context and current state

- Makes decision (approve/reject/modify)

- Can update state with feedback

- Can cancel workflow if needed

#### 5. Resume with decision

- Call app.stream() or app.invoke() with same config

- Graph loads checkpoint and continues

- Executes from next node

- Can pause again if another interrupt is encountered

### Execution Flow

#### 1. Run until interrupt

- Graph executes nodes normally

- When interrupt point is reached, stops

- Returns control to caller

- State is checkpointed automatically

#### 2. Save checkpoint

- All state is serialized

- Checkpoint ID is generated

- Metadata is saved (timestamp, thread_id, node_name)

- Saved to configured checkpointer (memory/sqlite/postgres)

#### 3. Return control

- Graph execution ends temporarily

- Returns checkpoint info to caller

- Application can show UI for human review

- State can be queried to display info

#### 4. Human reviews

- Human views context and current state

- Makes decision (approve/reject/modify)

- Can update state with feedback

- Can cancel workflow if needed

#### 5. Resume with decision

- Call app.stream() or app.invoke() with same config

- Graph loads checkpoint and continues

- Executes from next node

- Can pause again if another interrupt is encountered

## Persistence & Checkpointing

### Why Persistence?

#### Long-running workflows

- **Multi-day processes**: Workflows that can run over multiple days (approval chains, review cycles)

- **Async operations**: Waiting for external services, APIs, or batch jobs

- **Scheduled tasks**: Resume workflow at specific times

- **Resource optimization**: No need to keep process running continuously

#### Resume after failure

- **Crash recovery**: Server restart, deployment doesn’t lose progress

- **Network issues**: Retry failed external calls without restarting from beginning

- **Resource exhaustion**: OOM, timeout can be recovered from

- **Partial execution**: Don’t lose completed work

#### Audit trail

- **Compliance**: Track all decisions and state changes

- **Debugging**: View exact state at any point in time

- **Analytics**: Analyze workflow patterns and bottlenecks

- **Accountability**: Know who did what and when

#### Multi-session conversations

- **User context**: Maintain conversation context across sessions

- **Long-term memory**: Remember previous interactions

- **Personalization**: Build profile based on history

- **Seamless UX**: User can logout/login without losing context

### Checkpoint Savers

#### InMemorySaver (in-memory)

- **Use case**: Development, testing, demos

- **Pros**: Fast, simple, no setup required

- **Cons**: Lost on restart, doesn’t scale, single process only

- **Not for production**: Only use for local development

#### SQLiteSaver (local persistence)

- **Use case**: Small apps, prototypes, single-server deployments

- **Pros**: File-based, portable, no server needed

- **Cons**: Not distributed, limited concurrency

- **Good for**: POCs, small production apps with low traffic

#### PostgresSaver (production)

- **Use case**: Production apps, distributed systems

- **Pros**: Scalable, concurrent, reliable, distributed

- **Cons**: Requires PostgreSQL setup, more complex

- **Best for**: Production applications with high availability needs

#### Custom savers

- **Use case**: Special requirements (Redis, MongoDB, S3, etc.)

- **Implementation**: Extend BaseCheckpointSaver class

- **Flexibility**: Implement custom serialization, storage logic

- **Examples**: Redis for speed, S3 for archival, MongoDB for flexibility

------------------------------------------------------------------------

## InMemorySaver

### Setup

``` python
from langgraph.checkpoint.memory import InMemorySaver

# Create in-memory checkpointer
memory = InMemorySaver()

# Compile graph with checkpointer
app = workflow.compile(checkpointer=memory)
```

### Usage

#### Automatic checkpointing

- Checkpoint created after each node execution

- No manual save calls needed

- State changes tracked automatically

- Rollback possible to any checkpoint

#### Thread-based isolation

- Each thread_id has separate checkpoint history

- Parallel conversations don’t interfere

- Thread-safe operations

- Clean separation of concerns

#### In-memory only

- **Lost on restart**: All checkpoints lost when process dies

- **No persistence**: Not saved to disk

- **Fast**: Extremely fast as only RAM operations

- **Development only**: Should not use in production

------------------------------------------------------------------------

## SQLite Checkpointer

### Setup

``` python
from langgraph.checkpoint.sqlite import SqliteSaver

# Option 1: From connection string
with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    app = workflow.compile(checkpointer=checkpointer)

    # Use app...
    for event in app.stream(initial_state, config):
        print(event)

# Option 2: Persistent connection
checkpointer = SqliteSaver.from_conn_string("checkpoints.db")
app = workflow.compile(checkpointer=checkpointer)

# Use across multiple calls
result1 = app.invoke(input1, config1)
result2 = app.invoke(input2, config2)
```

### Features

#### Persistent storage

- Checkpoints survive process restarts

- Stored in SQLite database file

- Can copy/backup database file

- Resume workflows after days/weeks

#### Thread management

- Multiple threads stored in same DB

- Efficient indexing by thread_id

- Query threads by metadata

- Clean up old threads

#### Checkpoint history

- Full history of state changes

- Can replay from any point

- Time-travel debugging

- Built-in audit trail

### Database Schema

``` sql
-- Checkpoints table
CREATE TABLE checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_id TEXT NOT NULL,
    parent_checkpoint_id TEXT,
    checkpoint BLOB NOT NULL,  -- Serialized state
    metadata JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (thread_id, checkpoint_id)
);

-- Index for efficient queries
CREATE INDEX idx_thread_created ON checkpoints(thread_id, created_at);
CREATE INDEX idx_parent ON checkpoints(parent_checkpoint_id);

-- Writes table (for tracking state updates)
CREATE TABLE writes (
    thread_id TEXT NOT NULL,
    checkpoint_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    idx INTEGER NOT NULL,
    channel TEXT NOT NULL,
    value BLOB,
    FOREIGN KEY (thread_id, checkpoint_id)
        REFERENCES checkpoints(thread_id, checkpoint_id)
);
```

------------------------------------------------------------------------

## Postgres Checkpointer

### Production-Ready

``` python
from langgraph.checkpoint.postgres import PostgresSaver

# Connection string
conn_string = "postgresql://user:password@localhost:5432/langgraph_db"

# Create checkpointer
checkpointer = PostgresSaver.from_conn_string(conn_string)

# Use with connection pooling
from psycopg_pool import ConnectionPool

pool = ConnectionPool(
    conn_string,
    min_size=5,
    max_size=20
)

checkpointer = PostgresSaver(pool)
app = workflow.compile(checkpointer=checkpointer)
```

### Benefits

#### Scalable

- Handle thousands of concurrent threads

- Efficient indexing and querying

- Large checkpoint history support

- No single-file limitations like SQLite

#### Distributed

- Multiple app instances share same checkpoints

- Load balancing across servers

- High availability setup possible

- Geographic replication support

#### Reliable

- ACID transactions

- Point-in-time recovery

- Backup and restore capabilities

- Production-grade durability

------------------------------------------------------------------------

## Thread Management

### Thread ID

``` python
# Simple thread ID
config = {
    "configurable": {
        "thread_id": "conversation-123"
    }
}

# Thread with user ID
config = {
    "configurable": {
        "thread_id": f"user-{user_id}-session-{session_id}"
    }
}

# Thread with metadata
config = {
    "configurable": {
        "thread_id": "order-processing-12345",
        "metadata": {
            "user_id": "user-123",
            "order_id": "order-456",
            "started_at": "2025-01-07T10:00:00Z"
        }
    }
}
```

### Multiple Threads

#### Parallel conversations

- Each user has separate thread

- Multiple conversations per user possible

- No cross-contamination of state

- Independent execution

#### User sessions

- Web session → thread mapping

- Mobile app sessions

- Desktop app instances

- Each isolated completely

#### Isolated state

- State changes in one thread don’t affect others

- Safe concurrent execution

- No race conditions

- Clean abstraction

### Thread Lifecycle

``` python
# Create new thread
config = {"configurable": {"thread_id": "new-conversation"}}
app.invoke(initial_input, config)

# Continue existing thread
app.invoke(follow_up_input, config)  # Same thread_id

# List all checkpoints for thread
history = app.get_state_history(config)

# Archive thread (custom implementation)
def archive_thread(thread_id: str):
    # Move to archive table or cold storage
    pass

# Delete thread
def delete_thread(thread_id: str):
    # Remove all checkpoints for thread
    checkpointer.delete_thread(thread_id)
```

------------------------------------------------------------------------

## State Updates

### Manual State Updates

``` python
# Basic update
app.update_state(
    config,
    {"approved": True}
)

# Multiple fields
app.update_state(
    config,
    {
        "approved": False,
        "feedback": "Needs more details",
        "revision_count": 2
    }
)

# Merge vs replace
app.update_state(
    config,
    {"new_field": "value"},
    as_node=None  # Merges with existing state
)
```

### As-Node Updates

``` python
# Update as if specific node executed
app.update_state(
    config,
    {"status": "reviewed"},
    as_node="human_review"  # Simulates node execution
)

# Useful for:
# - Manual corrections
# - Skipping nodes
# - Testing specific paths
# - Fixing stuck workflows

# Example: Skip approval
current = app.get_state(config)
if current.next == ("approval",):
    app.update_state(
        config,
        {"approved": True},
        as_node="approval"  # Acts like approval node ran
    )
```

------------------------------------------------------------------------

## Checkpoint History

### Get Checkpoints

``` python
# Get full history
config = {"configurable": {"thread_id": "conv-123"}}
checkpoints = app.get_state_history(config)

for checkpoint in checkpoints:
    print(f"Checkpoint ID: {checkpoint.config['configurable']['checkpoint_id']}")
    print(f"Parent: {checkpoint.parent_config}")
    print(f"Values: {checkpoint.values}")
    print(f"Next: {checkpoint.next}")
    print(f"Metadata: {checkpoint.metadata}")
    print("---")

# Get specific checkpoint
specific_config = {
    "configurable": {
        "thread_id": "conv-123",
        "checkpoint_id": "checkpoint-abc-123"
    }
}
specific_state = app.get_state(specific_config)

# Replay from specific checkpoint
for event in app.stream(None, specific_config):
    print(event)
```

**Checkpoint structure:**

``` python
{
    "config": {
        "configurable": {
            "thread_id": "conv-123",
            "checkpoint_id": "abc123",
            "checkpoint_ns": ""
        }
    },
    "values": {  # Current state
        "messages": [...],
        "user_input": "...",
        ...
    },
    "next": ("node_name",),  # Next nodes to execute
    "metadata": {
        "source": "loop",
        "step": 3,
        "writes": {...}
    },
    "parent_config": {...}  # Previous checkpoint
}
```

------------------------------------------------------------------------

## Memory Patterns

### Short-Term Memory (CHECKPOINTER)

#### Current conversation

- Recent messages (last 10-20)

- Current task context

- Temporary variables

- Session-specific data

#### Session state

``` python
class SessionState(TypedDict):
    user_id: str
    session_start: str
    recent_messages: list
    current_task: Optional[str]
    temp_data: dict
```

**Characteristics:**

- Fast access (in checkpoint)

- Ephemeral (may expire)

- Thread-scoped

- Lost when thread deleted

### Long-Term Memory

#### Historical context

- User preferences

- Past conversations summary

- Learned patterns

- User profile

#### External storage

``` python
# Store in separate database
class UserMemory:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.vector_store = get_vector_store()
        self.db = get_database()

    def remember(self, content: str, metadata: dict):
        """Store long-term memory"""
        embedding = get_embedding(content)
        self.vector_store.add(
            embedding=embedding,
            content=content,
            metadata={**metadata, "user_id": self.user_id}
        )

    def recall(self, query: str, k: int = 5):
        """Retrieve relevant memories"""
        query_embedding = get_embedding(query)
        return self.vector_store.search(
            query_embedding,
            filter={"user_id": self.user_id},
            k=k
        )

# Use in node
def process_with_memory(state):
    memory = UserMemory(state["user_id"])

    # Recall relevant context
    relevant_memories = memory.recall(state["current_input"])

    # Use in processing
    context = f"Past context: {relevant_memories}\nCurrent: {state['current_input']}"
    response = llm.invoke(context)

    # Remember this interaction
    memory.remember(
        content=f"User: {state['current_input']}\nAssistant: {response}",
        metadata={"timestamp": datetime.now(), "topic": state.get("topic")}
    )

    return {"output": response}
```

------------------------------------------------------------------------

## Practice Exercises

### Exercise 1: Cache Tool Implementation

**Objective**: In the **ORCHESTRATOR** layer (module 03), add a **cache_tool** that stores all outputs from RAG tool / IT support Agent into a vectorstore (e.g., FAISS). This tool will be used for follow-up requests or similar queries from previous requests about FPT / IT support.

**Requirements:**

- Create a cache_tool that stores RAG/IT support responses

- Use FAISS vectorstore for caching

- Implement similarity search for follow-up queries

- Store metadata (timestamp, query type, source)

- Implement cache invalidation strategy

### Exercise 2: Interrupt Before Tool Execution

**Objective**: Add interrupt_before to all tools related to ticket/booking, requiring user confirmation.

**Requirements:**

- Developer must have a confirmation message to return to user

- If user input is **“y”**, tool continues

- If input is different, agent must **end**

**Hint**:

- Write a confirmation message (string)

- Check in graph.get_state() to identify which tool is in pending state (most recent message in state is ToolMessage)

- Return confirmation message to user

- Process user response accordingly

**Implementation Steps:**

1.  Identify all ticket/booking related tools

2.  Add interrupt_before for these tools

3.  Create confirmation message generator

4.  Implement state checker to detect pending tools

5.  Handle user confirmation (y/n)

6.  Resume or cancel based on user input

------------------------------------------------------------------------

## Summary

**Key Takeaways:**

1.  **Human-in-the-Loop** is essential for high-stakes, compliance-critical workflows

2.  **Interrupts** enable pausing and resuming execution with full state preservation

3.  **Persistence** allows long-running workflows and crash recovery

4.  **Checkpointing** provides audit trails and time-travel debugging

5.  **Thread management** enables isolated, parallel conversations

6.  **Memory patterns** combine short-term (checkpointer) and long-term (external) storage

**Best Practices:**

- Always use checkpointer for production agents

- Use InMemorySaver for dev, PostgresSaver for production

- Design clear confirmation UIs for critical operations

- Implement proper error handling and recovery

- Store sensitive data securely

- Set appropriate checkpoint retention policies

- Monitor checkpoint size and performance
