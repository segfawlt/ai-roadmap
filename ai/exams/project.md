# AI Project Exams

This page consolidates project exam descriptions from all advanced AI training modules.

------------------------------------------------------------------------

## RAG Optimization Project Exam

# Final Exam: Enterprise RAG System

## overview

| Field             | Value                 |
|-------------------|-----------------------|
| **Course**        | RAG and Optimization  |
| **Duration**      | 240 minutes (4 hours) |
| **Passing Score** | 70%                   |
| **Total Points**  | 100                   |

------------------------------------------------------------------------

## Description

You have been hired as an **AI Engineer** at **TechDocs Inc.**, a company that provides enterprise documentation solutions. Your task is to build a production-ready **Enterprise RAG System** that can answer complex questions about technical documentation, company policies, and product specifications.

The current basic RAG system has several limitations:

- Poor retrieval quality due to fixed-size chunking

- Slow search performance with growing document collections

- Inability to handle keyword-specific queries (error codes, product IDs)

- Redundant and irrelevant results in retrieved documents

- Missing relationship information between entities (policies, stakeholders, regulations)

You must apply **all five optimization techniques** learned in this module to build a comprehensive, production-grade RAG system.

------------------------------------------------------------------------

## Objectives

By completing this exam, you will demonstrate mastery of:

- Implementing **Semantic Chunking** for intelligent document segmentation

- Configuring **HNSW Index** for high-performance vector search

- Building **Hybrid Search** combining BM25 and Vector Search with RRF fusion

- Applying **Query Transformation** techniques (HyDE and Query Decomposition)

- Implementing **Post-Retrieval Processing** with Cross-Encoder and MMR

- Designing a **GraphRAG** architecture for relationship-aware retrieval

------------------------------------------------------------------------

## Problem Description

Build an **Enterprise RAG System** named `enterprise-rag-system` that processes a collection of technical documents and provides accurate, contextual answers to user queries. The system must handle:

1.  **Technical documentation** with code snippets, error codes, and specifications

2.  **Policy documents** with stakeholder relationships and regulatory references

3.  **Product catalogs** with model numbers, features, and comparisons

The system should intelligently route queries to the appropriate retrieval strategy and provide high-quality, diverse, and accurate results.

------------------------------------------------------------------------

## Assumptions

- You have access to sample documents (technical docs, policies, product specs) or will use provided sample data

- OpenAI API key or compatible LLM endpoint is available

- Neo4j database is available (local Docker or cloud instance)

- Python 3.11+ environment with necessary packages installed

- Basic understanding of all five RAG optimization techniques

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain[anthropic]` \>= 1.0 \# or `langchain[openai]`

  - `langchain-neo4j` \>= 0.1.0

  - `sentence-transformers` \>= 2.2.0

  - `chromadb` \>= 0.4.0 OR `qdrant-client` \>= 1.7.0

  - `rank-bm25` \>= 0.2.2

  - `pydantic` \>= 2.0.0

  - `neo4j` \>= 5.0.0

### Infrastructure

- Vector Database: ChromaDB or Qdrant with HNSW indexing

- Graph Database: Neo4j (Docker recommended)

- Embedding Model: `text-embedding-3-small` or `all-MiniLM-L6-v2`

- Cross-Encoder: `cross-encoder/ms-marco-MiniLM-L-6-v2`

- LLM: A frontier model such as Claude Sonnet 4.6, GPT-5.2, or Gemini 3

------------------------------------------------------------------------

## Tasks

### Task 1: Advanced Indexing Pipeline (20 points)

**Time Allocation**: 45 minutes

Implement an intelligent document indexing pipeline that preserves semantic coherence.

#### Requirements:

1.  **Semantic Chunking Implementation**

    - Build a chunker that splits documents based on semantic similarity between sentences

    - Configure similarity threshold (0.7-0.85) and chunk size limits

    - Handle edge cases: code blocks, tables, lists, short documents

2.  **HNSW Index Configuration**

    - Set up vector database with HNSW indexing

    - Configure optimal parameters: `M=32`, `ef_construction=200`, `ef_search=100`

    - Document the trade-offs for your chosen configuration

3.  **Indexing Pipeline**

    - Process at least 20 documents through the pipeline

    - Store metadata (source, chunk_id, document_type) with each vector

    - Implement batch processing for efficiency

#### Deliverables:

- `indexing/semantic_chunker.py`

- `indexing/vector_store.py`

- Indexed document collection with metadata

------------------------------------------------------------------------

### Task 2: Hybrid Search Implementation (20 points)

**Time Allocation**: 45 minutes

Build a hybrid retrieval system that combines keyword and semantic search.

#### Requirements:

1.  **BM25 Retriever**

    - Implement BM25 indexing for all document chunks

    - Proper tokenization with case normalization and punctuation handling

    - Return top-K results with BM25 scores

2.  **Hybrid Search with RRF**

    - Execute both BM25 and Vector Search in parallel

    - Implement RRF fusion: `RRF(d)`` ``=`` ``Σ`` ``1/(60`` ``+`` ``rank(d))`

    - Handle documents appearing in only one result list

3.  **Query Router**

    - Analyze query to determine optimal search strategy

    - Route keyword-heavy queries to prioritize BM25

    - Route semantic queries to prioritize Vector Search

    - Use Hybrid Search as default

#### Deliverables:

- `retrieval/bm25_retriever.py`

- `retrieval/hybrid_search.py`

- `retrieval/query_router.py`

------------------------------------------------------------------------

### Task 3: Query Transformation Layer (15 points)

**Time Allocation**: 35 minutes

Implement query transformation to handle vague and complex queries.

#### Requirements:

1.  **HyDE Implementation**

    - Generate hypothetical answer paragraphs using LLM

    - Use hypothetical answer embedding for retrieval

    - Design domain-appropriate generation prompts

2.  **Query Decomposition**

    - Detect multi-part questions requiring information from multiple sources

    - Generate independent sub-queries for parallel retrieval

    - Aggregate results from all sub-queries

3.  **Transformation Router**

    - Classify queries: simple, vague (use HyDE), complex (use Decomposition)

    - Apply appropriate transformation before retrieval

#### Deliverables:

- `transformation/hyde.py`

- `transformation/query_decomposition.py`

- `transformation/transformation_router.py`

------------------------------------------------------------------------

### Task 4: Post-Retrieval Processing (15 points)

**Time Allocation**: 35 minutes

Implement re-ranking and diversity optimization for retrieved results.

#### Requirements:

1.  **Cross-Encoder Re-ranking**

    - Retrieve top-50 candidates with Bi-Encoder

    - Re-rank using Cross-Encoder (`cross-encoder/ms-marco-MiniLM-L-6-v2`)

    - Return top-10 re-ranked results

2.  **MMR for Diversity**

    - Implement MMR algorithm with configurable λ parameter

    - Default λ=0.5 for balanced relevance/diversity

    - Ensure diverse information coverage in final results

3.  **Configurable Pipeline**

    - Support both: Cross-Encoder → MMR and MMR → Cross-Encoder orders

    - Allow configuration of k values at each stage

#### Deliverables:

- `post_retrieval/cross_encoder_reranker.py`

- `post_retrieval/mmr.py`

- `post_retrieval/post_retrieval_pipeline.py`

------------------------------------------------------------------------

### Task 5: GraphRAG Integration (20 points)

**Time Allocation**: 50 minutes

Build a knowledge graph for relationship-aware retrieval.

#### Requirements:

1.  **Entity Extraction**

    - Define Pydantic models for domain entities (Policy, Stakeholder, Product, Regulation, etc.)

    - Extract entities and relationships using LLM with structured output

    - Validate extracted data against schema

2.  **Knowledge Graph Construction**

    - Populate Neo4j with extracted entities and relationships

    - Use MERGE to prevent duplicates

    - Create appropriate indexes for query performance

3.  **Graph-Aware Retrieval**

    - Implement natural language to Cypher translation

    - Support relationship traversal queries

    - Combine graph results with vector search results

#### Deliverables:

- `graph/entity_models.py`

- `graph/entity_extractor.py`

- `graph/knowledge_graph.py`

- `graph/graph_retriever.py`

------------------------------------------------------------------------

### Task 6: Integration and Orchestration (10 points)

**Time Allocation**: 30 minutes

Integrate all components into a unified RAG system.

#### Requirements:

1.  **Unified Query Pipeline**

    - Accept user query as input

    - Apply query classification and routing

    - Execute appropriate retrieval strategy

    - Apply post-retrieval processing

    - Generate final answer using LLM

2.  **Configuration Management**

    - Externalize all configurable parameters

    - Support different modes: fast (less accurate), accurate (slower), balanced

3.  **Error Handling and Logging**

    - Graceful degradation if a component fails

    - Structured logging for debugging and monitoring

#### Deliverables:

- `main.py` or `enterprise_rag.py`

- `config.py` or `config.yaml`

- `README.md` with setup and usage instructions

------------------------------------------------------------------------

## Questions to Answer

Include written answers to these questions in your `README.md` or a separate `ANSWERS.md` file:

1.  **Architecture Decision**: Explain why you chose your specific HNSW parameters and how they balance speed vs. accuracy for this use case.

2.  **Hybrid Search Trade-offs**: Describe a scenario where Hybrid Search significantly outperforms pure Vector Search, and explain why.

3.  **Query Transformation Selection**: How does your system decide when to use HyDE vs. Query Decomposition? What signals does it look for?

4.  **Re-ranking Strategy**: Why did you choose your specific order of Cross-Encoder and MMR? What would change if the use case prioritized diversity over precision?

5.  **GraphRAG Value**: Provide an example query that your GraphRAG component can answer that would be impossible or very difficult with vector search alone.

------------------------------------------------------------------------

## Submission Rules

### Required Deliverables

Complete source code organized in the specified directory structure

`README.md` with:

- Setup instructions (dependencies, environment variables, database setup)

- Usage examples for different query types

- Architecture diagram (can be text-based)

`ANSWERS.md` with written responses to the 5 questions

`docker-compose.yml` for Neo4j and any other services

Sample queries demonstrating each component’s functionality

Screenshots or logs showing successful execution

### Submission Checklist

All code runs without errors

Semantic Chunking preserves document semantics

HNSW index is properly configured and benchmarked

Hybrid Search correctly combines BM25 and Vector results

Query Transformation handles vague and complex queries

Cross-Encoder improves ranking precision

MMR ensures result diversity

GraphRAG answers relationship queries

All components are integrated in unified pipeline

Documentation is complete and clear

------------------------------------------------------------------------

## Grading Rubrics

| Criterion | Weight | Excellent (90-100%) | Good (70-89%) | Satisfactory (50-69%) | Needs Improvement (\<50%) |
|----|----|----|----|----|----|
| **Advanced Indexing** | 20% | Semantic chunking preserves context perfectly; HNSW optimally configured with benchmarks | Chunking works with minor issues; HNSW configured but not optimized | Basic chunking implemented; HNSW uses default parameters | Chunking breaks context; HNSW not implemented |
| **Hybrid Search** | 20% | BM25 and RRF perfectly implemented; Query router makes intelligent decisions | Hybrid search works; Router has some misclassifications | Basic hybrid search; No query routing | Hybrid search not functional |
| **Query Transformation** | 15% | HyDE and Decomposition both work excellently; Smart routing between them | Both techniques work; Routing is rule-based | One technique works; No routing | Neither technique functional |
| **Post-Retrieval** | 15% | Cross-Encoder significantly improves precision; MMR provides diverse results | Both components work; Measurable improvement | One component works | Neither component functional |
| **GraphRAG** | 20% | Complete entity extraction; Rich graph; Answers complex relationship queries | Graph populated; Basic queries work | Partial graph; Limited queries | Graph not functional |
| **Integration** | 10% | Seamless pipeline; Excellent error handling; Clean configuration | Components integrated; Some rough edges | Partial integration | Components not connected |

------------------------------------------------------------------------

## Estimated Time

| Task                         | Time Allocation           |
|------------------------------|---------------------------|
| Task 1: Advanced Indexing    | 45 minutes                |
| Task 2: Hybrid Search        | 45 minutes                |
| Task 3: Query Transformation | 35 minutes                |
| Task 4: Post-Retrieval       | 35 minutes                |
| Task 5: GraphRAG             | 50 minutes                |
| Task 6: Integration          | 30 minutes                |
| **Total**                    | **240 minutes (4 hours)** |

------------------------------------------------------------------------

## Hints

**General Tips**:

- Start by setting up the infrastructure (Neo4j, Vector DB) before writing code

- Test each component independently before integration

- Use the companion notebooks from assignments as references

- Cache LLM responses during development to save API costs

**Component-Specific Tips**:

- For Semantic Chunking: Use `sentence-transformers` for efficient similarity calculation

- For HNSW: Prioritize `ef_search` tuning for query-time optimization

- For BM25: Use `nltk.word_tokenize()` for consistent tokenization

- For HyDE: The hypothetical answer doesn’t need to be factually correct

- For Cross-Encoder: Batch processing significantly improves throughput

- For GraphRAG: Test Cypher queries in Neo4j Browser before implementing in code

------------------------------------------------------------------------

## Notes:

- **You can use the your implementation in your previous assignment lab that you have done as the starting point.**

------------------------------------------------------------------------

## LangGraph and Agentic AI Project Exam

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

------------------------------------------------------------------------

## LLMOps and Evaluation Project Exam

# Final Exam: Production-Ready RAG Evaluation System

## overview

| Field             | Value                 |
|-------------------|-----------------------|
| **Course**        | LLMOps and Evaluation |
| **Duration**      | 240 minutes (4 hours) |
| **Passing Score** | 70%                   |
| **Total Points**  | 100                   |

------------------------------------------------------------------------

## Description

You have been hired as an **MLOps Engineer** at **AI Solutions Corp.**, a company that builds enterprise AI assistants. Your task is to build a **Production-Ready RAG Evaluation System** that combines automated quality assessment, comprehensive observability, and rigorous architecture comparison.

The current system lacks:

- Automated evaluation metrics to measure answer quality

- Observability into LLM execution, costs, and latency

- Data-driven architecture selection based on experiments

You must apply knowledge from **RAGAS Evaluation Metrics**, **LLM Observability (LangFuse/LangSmith)**, and **RAG Architecture Comparison** to build a comprehensive evaluation and monitoring platform.

------------------------------------------------------------------------

## Objectives

By completing this exam, you will demonstrate mastery of:

- Implementing **RAGAS metrics** (Faithfulness, Answer Relevancy, Context Precision, Context Recall)

- Integrating **LangFuse and/or LangSmith** for comprehensive LLM tracing and cost tracking

- Designing and executing **RAG architecture experiments** with scientific rigor

- Building an **end-to-end evaluation pipeline** that combines all three components

- Making **data-driven architecture recommendations** based on experimental results

------------------------------------------------------------------------

## Problem Description

Build a **Production-Ready RAG Evaluation System** named `rag-evaluation-platform` that:

1.  **Evaluates RAG quality** using RAGAS metrics on generated responses

2.  **Traces all LLM operations** with full observability (tokens, costs, latency)

3.  **Compares multiple RAG architectures** systematically

4.  **Produces actionable reports** for architecture selection

The system should serve as a complete toolkit for evaluating, monitoring, and optimizing RAG systems in production.

------------------------------------------------------------------------

## Assumptions

- You have completed the assignments on RAGAS, Observability, and Experiment Comparison

- OpenAI API key or compatible LLM endpoint is available

- LangFuse Cloud account OR local Docker setup for self-hosted LangFuse

- LangSmith account (free tier)

- Python 3.11+ environment with necessary packages installed

- Sample documents and test questions are provided or created

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `ragas` \>= 0.2

  - `langfuse` \>= 2.0.0

  - `langchain[openai]` \>= 1.0

  - `langsmith` \>= 0.1.0

  - `chromadb` \>= 0.4.0 OR `qdrant-client` \>= 1.7.0

  - `sentence-transformers` \>= 2.2.0

  - `pandas` \>= 2.0.0

  - `matplotlib` \>= 3.7.0

### Infrastructure

- Vector Database: ChromaDB or Qdrant

- Observability: LangFuse (required) + LangSmith (optional)

- Embedding Model: `text-embedding-3-small` or equivalent

- LLM: A frontier model such as Claude Sonnet 4.6, GPT-5.2, or Gemini 3

------------------------------------------------------------------------

## Tasks

### Task 1: RAGAS Evaluation Pipeline (25 points)

**Time Allocation**: 60 minutes

Build a comprehensive evaluation pipeline using all four RAGAS metrics.

#### Requirements:

1.  **Implement RAGAS Evaluation Module**

    - Create functions to calculate Faithfulness, Answer Relevancy, Context Precision, and Context Recall

    - Support batch evaluation on datasets

    - Handle edge cases (empty contexts, very short answers)

2.  **Create Evaluation Dataset**

    - Prepare at least 30 test questions with ground truth answers

    - Categorize questions: Factual (40%), Relational (30%), Multi-hop (20%), Analytical (10%)

    - Include retrieved contexts for each question

3.  **Run Evaluation**

    - Execute evaluation on the complete dataset

    - Calculate aggregate statistics (mean, std, min, max)

    - Identify failure cases (scores \< 0.5)

#### Deliverables:

- `evaluation/ragas_evaluator.py` - Core evaluation logic

- `evaluation/dataset.py` - Dataset loading and preparation

- `data/test_questions.json` - Test dataset with ground truth

------------------------------------------------------------------------

### Task 2: LLM Observability Integration (25 points)

**Time Allocation**: 60 minutes

Implement comprehensive tracing and monitoring for all LLM operations.

#### Requirements:

1.  **LangFuse Integration**

    - Configure LangFuse SDK with proper authentication

    - Implement `CallbackHandler` for all LangChain operations

    - Capture: input/output, token counts, latency, costs

2.  **Cost Tracking Dashboard**

    - Track token usage per query

    - Calculate costs based on model pricing

    - Generate cost breakdown reports

3.  **Production Best Practices**

    - Implement configurable sampling (100% dev, 5% prod)

    - Add PII masking for sensitive data

    - Create correlation IDs for request tracking

4.  **(Bonus) LangSmith Integration**

    - Configure auto-tracing via environment variables

    - Demonstrate Playground debugging for a failed trace

#### Deliverables:

- `observability/langfuse_handler.py` - LangFuse integration

- `observability/cost_tracker.py` - Cost calculation logic

- `observability/pii_masker.py` - PII handling

- Screenshots of LangFuse dashboard with traces

------------------------------------------------------------------------

### Task 3: RAG Architecture Comparison (25 points)

**Time Allocation**: 60 minutes

Design and execute a rigorous experiment comparing multiple RAG architectures.

#### Requirements:

1.  **Implement Two RAG Architectures**

    - **Naive RAG**: Fixed chunking, Top-K retrieval, direct generation

    - **Advanced RAG**: Semantic chunking, hybrid search, re-ranking

2.  **Run Comparative Experiments**

    - Execute both architectures on the same test dataset

    - Capture all RAGAS metrics for each architecture

    - Track latency and cost per query

3.  **Performance Analysis**

    - Break down performance by question category

    - Calculate statistical significance of differences

    - Create visualizations (bar charts, tables)

#### Deliverables:

- `architectures/naive_rag.py` - Naive RAG implementation

- `architectures/advanced_rag.py` - Advanced RAG implementation

- `experiments/runner.py` - Experiment execution

- `results/comparison_table.md` - Results summary

------------------------------------------------------------------------

### Task 4: Integrated Evaluation Platform (25 points)

**Time Allocation**: 60 minutes

Combine all components into a unified evaluation platform.

#### Requirements:

1.  **End-to-End Pipeline**

    - Single entry point to run complete evaluation

    - Automatic tracing of all operations

    - Configurable architecture selection

2.  **Comprehensive Reporting**

    - Generate evaluation report with all metrics

    - Include observability insights (cost, latency distribution)

    - Architecture comparison summary

    - Actionable recommendations

3.  **CLI Interface**

    ``` bash
    python evaluate.py --architecture naive --dataset data/test.json --output results/
    python evaluate.py --architecture advanced --dataset data/test.json --output results/
    python compare.py --results-dir results/ --output comparison_report.md
    ```

4.  **Answer Key Questions**

    - Which architecture should be used for production and why?

    - What is the cost-quality trade-off between architectures?

    - What are the top 3 failure patterns and how to address them?

#### Deliverables:

- `evaluate.py` - Main evaluation script

- `compare.py` - Architecture comparison script

- `reports/evaluation_report.md` - Complete evaluation report

- `ANSWERS.md` - Written responses to key questions

------------------------------------------------------------------------

## Questions to Answer

Include written responses to these questions in `ANSWERS.md`:

1.  **RAGAS Interpretation**: Analyze your Faithfulness and Answer Relevancy scores. What do low scores indicate about your RAG system, and how would you improve them?

2.  **Observability Value**: How did LangFuse/LangSmith tracing help you identify issues in your RAG pipeline? Provide a specific example.

3.  **Architecture Decision**: Based on your experiments, which RAG architecture would you recommend for a customer support chatbot vs. a legal document Q&A system? Justify with data.

4.  **Cost Optimization**: If you had to reduce costs by 50% while maintaining 90% of quality, what strategies would you employ? Reference your experimental results.

5.  **Production Readiness**: What additional monitoring, alerting, or evaluation would you add before deploying this system to production?

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Complete source code organized in the specified directory structure

`README.md` with:

- Setup instructions (dependencies, API keys, observability setup)

- Usage examples for CLI commands

- Architecture diagram of the evaluation platform

`ANSWERS.md` with written responses to the 5 questions

Test dataset with at least 30 categorized questions

Results tables and visualizations

Screenshots of observability dashboards

### Submission Checklist

All code runs without errors

RAGAS evaluation produces valid scores for all metrics

LangFuse traces are captured and visible in dashboard

Both RAG architectures are implemented and evaluated

Comparison report includes statistical analysis

All questions answered with data-backed reasoning

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria | Weight | Excellent (90-100%) | Good (70-89%) | Needs Improvement (50-69%) | Unsatisfactory (\<50%) |
|----|----|----|----|----|----|
| **RAGAS Evaluation** | 25% | All 4 metrics implemented correctly; comprehensive dataset; insightful failure analysis | Metrics implemented; adequate dataset; basic analysis | Partial metrics; small dataset; minimal analysis | Missing metrics; no dataset |
| **Observability** | 25% | Full LangFuse integration; cost tracking; PII handling; production best practices | LangFuse working; basic cost tracking; some best practices | Partial tracing; no cost tracking | No observability integration |
| **Architecture Comparison** | 25% | Both architectures implemented; rigorous experiments; statistical analysis; visualizations | Both architectures; experiments run; basic comparison | One architecture; limited experiments | No architecture comparison |
| **Integration & Reporting** | 15% | Seamless pipeline; comprehensive reports; CLI interface; actionable insights | Components integrated; adequate reports | Partial integration; basic reports | Components not connected |
| **Code Quality & Documentation** | 10% | Clean code; comprehensive docs; clear README; well-organized | Readable code; adequate docs | Messy code; minimal docs | Poor quality; no docs |

------------------------------------------------------------------------

## Estimated Time

| Task                                   | Time Allocation           |
|----------------------------------------|---------------------------|
| Task 1: RAGAS Evaluation Pipeline      | 60 minutes                |
| Task 2: LLM Observability Integration  | 60 minutes                |
| Task 3: RAG Architecture Comparison    | 60 minutes                |
| Task 4: Integrated Evaluation Platform | 60 minutes                |
| **Total**                              | **240 minutes (4 hours)** |

------------------------------------------------------------------------

## Hints

**Task 1 - RAGAS**:

- Use the companion notebook `10_RAG_Evaluation_with_Ragas.ipynb` as a reference

- Start with a small dataset (10 questions) to verify your pipeline before scaling up

- For claim decomposition in Faithfulness, pick a strong judge model (e.g., Claude Opus 4.6 or GPT-5.2) for higher accuracy

**Task 2 - Observability**:

- Set up LangFuse first since it requires explicit callback handlers (good for understanding)

- Use environment variables to switch between dev (100% tracing) and prod (5% sampling) modes

- Test PII masking with fake data before using real sensitive information

**Task 3 - Experiments**:

- Use the same embedding model for both architectures to ensure fair comparison

- Run each query multiple times if measuring latency to account for variance

- Calculate confidence intervals when comparing metric differences

**Task 4 - Integration**:

- Use Python’s `argparse` or `click` library for CLI implementation

- Generate markdown reports that can be easily shared with stakeholders

- Include both quantitative metrics and qualitative insights in recommendations
