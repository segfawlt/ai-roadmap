Reference: <https://langchain-ai.github.io/langgraph/how-tos/react-agent-from-scratch/>

# Multi-Expert Research Agent with ReAct Pattern

This page covers the ReAct (Reasoning + Acting) pattern for building multi-expert research agents, including how to compose specialized LLM-based tools, use LangGraph’s prebuilt ToolNode, and apply advanced techniques such as reflection, planning, and multi-expert orchestration.

## Learning Objectives

- Understand ReAct pattern (Reasoning + Acting)

- Implement Research Agent with LLM-based expert tools

- Use LangGraph’s prebuilt ToolNode

- Apply advanced techniques: reflection, planning, multi-expert

- Best practices for production-ready agents

## Research Agent Patterns

### overview

#### Introduction from Simple Research Agent

Basic Research Agent: User → LLM → Web Search → Answer

Problems:

- Single tool (web search) - limited expertise

- No reasoning about quality

- No planning for complex queries

- No reflection/improvement

#### 3 Core Improvements

1.  **Multi-Expert Tools**: Replace web search = specialized LLM experts

2.  **ReAct Pattern**: Structured reasoning + acting loop

3.  **Advanced Techniques**: Reflection, planning, iteration control

#### Agentic Workflows for Research

``` mermaid
        graph LR
    subgraph "Simple"
        SQ["Question"] --> SS["Search"] --> SA["Answer"]
    end
    subgraph "ReAct"
        RQ["Question"] --> RT["Think"] --> RA["Act"] --> RO["Observe"]
        RO -->|"repeat"| RT
        RO --> RANS["Answer"]
    end
```

### The 3 Key Patterns

#### 1. Tool Use (Multi-Expert)

Instead of 1 web search tool → Multiple specialized LLM experts

#### 2. ReAct (Reason + Act)

Coordinator LLM thinks before acting, observes results

#### 3. Reflection (Optional)

Agent reviews own output quality, refines if needed

### Why These Patterns Matter

**Higher accuracy**

- Expert knowledge \> generic search

- Structured reasoning \> direct answer

**Handle complex queries**

- Multi-step reasoning

- Combine multiple perspectives

**Quality control**

- Reflection catches errors

- Iterative improvement

## Pattern 1: Multi-Expert Tool Use

### Concept

#### Specialized LLM Experts

Instead of search tool → LLM with specialized knowledge:

- AI Research Expert (papers, trends, ML)

- Financial Analyst (stocks, markets, valuations)

- Medical Expert, Legal Expert, etc.

#### Why LLM Tools \> Web Search

- Consistent quality

- Structured reasoning

- Domain expertise

- No hallucination from bad search results

#### Tool as LLM Invoke

``` python
@tool
def ai_research_expert(query: str) -> str:
    """AI/ML research specialist"""
    messages = [
        SystemMessage(content=EXPERT_PROMPT),
        HumanMessage(content=query)
    ]
    return expert_llm.invoke(messages).content
```

### Implementation with LangGraph

#### State Definition

``` python
from typing import TypedDict, List, Annotated
from langchain.messages import AnyMessage
from langgraph.graph import add_messages

class ResearchState(TypedDict):
    messages: Annotated[List[AnyMessage], add_messages]
    max_iterations: int
    current_iteration: int
```

**Key points:**

- `messages`: Full conversation history

- `add_messages`: Auto-merge reducer

- Iteration tracking for loop control

#### Expert LLM Setup

``` python
from langchain.chat_models import init_chat_model

# Expert 1: AI Research
ai_research_llm = init_chat_model("claude-sonnet-4-6", temperature=0.3)
AI_RESEARCH_PROMPT = """You are an AI Research Expert.
Specialize in: ML papers, architectures, trends, research methods.
Provide academic-style responses with paper references."""

# Expert 2: Financial Analysis
financial_llm = init_chat_model("claude-sonnet-4-6", temperature=0.3)
FINANCIAL_PROMPT = """You are a Financial Analyst.
Specialize in: stocks, markets, valuations, investment strategy.
Provide data-driven insights with market context."""
```

#### Tool Definitions

``` python
from langchain.tools import tool
from langchain.messages import SystemMessage, HumanMessage

@tool
def ai_research_expert(query: str) -> str:
    """Consult AI Research Expert for ML/AI questions"""
    messages = [
        SystemMessage(content=AI_RESEARCH_PROMPT),
        HumanMessage(content=query)
    ]
    return ai_research_llm.invoke(messages).content

@tool
def financial_analyst(query: str) -> str:
    """Consult Financial Analyst for market/investment questions"""
    messages = [
        SystemMessage(content=FINANCIAL_PROMPT),
        HumanMessage(content=query)
    ]
    return financial_llm.invoke(messages).content
```

#### Coordinator Setup

``` python
coordinator_llm = init_chat_model("claude-sonnet-4-6", temperature=0)
coordinator_with_tools = coordinator_llm.bind_tools([
    ai_research_expert,
    financial_analyst
])
```

#### Graph Construction

``` python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode

# Prebuilt ToolNode - auto handles tool execution
tools = [ai_research_expert, financial_analyst]
tool_node = ToolNode(tools)

workflow = StateGraph(ResearchState)
workflow.add_node("coordinator", coordinator_node)
workflow.add_node("tools", tool_node)

workflow.add_edge(START, "coordinator")
workflow.add_conditional_edges(
    "coordinator",
    should_continue,
    {"continue": "tools", "end": END}
)
workflow.add_edge("tools", "coordinator")

app = workflow.compile()
```

### Benefits

#### Expert Knowledge Quality

- Specialized prompts \> generic search

- Consistent reasoning

- Domain-specific insights

#### Flexible Tool Composition

- Mix different experts per query

- Easy to add new experts

- Coordinator auto-routes

#### Better than Web Search for:

- Analysis questions (not just facts)

- Synthesis from multiple domains

- Consistent quality (no bad search results)

### Limitations

#### Higher Cost

- Multiple LLM calls per query

- Each expert = full LLM invoke

#### Latency

- Sequential expert consultations

- Longer than single search

#### Knowledge Cutoff

- Experts don’t know post-training events

- Still need web search for current news

## Pattern 2: ReAct (Reason + Act)

### Concept

#### Think (Reasoning)

Coordinator LLM explicitly reasons:

- “This question needs AI expertise”

- “I should consult financial analyst”

- “I have enough info now”

#### Act (Tool Use)

Based on reasoning → call appropriate tools:

``` python
AIMessage(
    content="I need AI research expertise",
    tool_calls=[{"name": "ai_research_expert", ...}]
)
```

#### Observe (Tool Results)

Receive and analyze expert responses:

``` python
ToolMessage(
    content="Expert says: transformers evolved to...",
    name="ai_research_expert"
)
```

#### Repeat

Loop until sufficient information or max iterations

### ReAct Strategies

#### Coordinator Prompt

``` python
COORDINATOR_PROMPT = """You are a Research Coordinator using ReAct.

Process:
1. THINK: Analyze user question
2. ACT: Decide which expert(s) to consult
3. OBSERVE: Review expert responses
4. REPEAT if needed OR give final answer

Tools:
- ai_research_expert: ML/AI questions
- financial_analyst: Market/investment questions

Guidelines:
- Use tools when you need expert knowledge
- Can consult multiple experts
- Synthesize inputs into coherent answer
- Max {max_iterations} iterations

Current: iteration {iteration}/{max_iterations}
"""
```

#### Routing Logic

``` python
def should_continue(state: ResearchState) -> str:
    last_message = state["messages"][-1]

    # Has tool calls? → Execute
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "use_tools"

    # Max iterations? → Force end
    if state["current_iteration"] >= state["max_iterations"]:
        return "end"

    # Has final answer → End naturally
    return "end"
```

### Workflow in Messages

#### Iteration 1: Think → Act

``` python
# User input
HumanMessage(content="What are transformer breakthroughs?")

# Coordinator thinks & acts
AIMessage(
    content="I need AI research expertise for this ML question",
    tool_calls=[{
        "name": "ai_research_expert",
        "args": {"query": "latest transformer architecture breakthroughs 2024"}
    }]
)
```

#### Iteration 1: Observe

``` python
# Tool executes → ToolMessage
ToolMessage(
    content="Recent advances: Mamba (state-space), RetNet (retention)...",
    name="ai_research_expert",
    tool_call_id="call_123"
)
```

#### Iteration 2: Synthesize or Continue

**Option A: Sufficient info**

``` python
AIMessage(content="Based on expert analysis, key breakthroughs are...")
# → END
```

**Option B: Need more info**

``` python
AIMessage(
    content="Need financial perspective on AI chip demand",
    tool_calls=[{"name": "financial_analyst", ...}]
)
# → Continue loop
```

### Implementation Details

#### Coordinator Node

``` python
def coordinator_node(state: ResearchState) -> dict:
    messages = state["messages"]

    # Add system prompt on first iteration
    if state["current_iteration"] == 0:
        system_msg = SystemMessage(content=COORDINATOR_PROMPT)
        messages = [system_msg] + messages

    # Coordinator decides: tool use or final answer
    response = coordinator_with_tools.invoke(messages)

    return {
        "messages": [response],
        "current_iteration": state["current_iteration"] + 1
    }
```

#### Prebuilt ToolNode (NOT custom!)

``` python
from langgraph.prebuilt import ToolNode

# ✨ Prebuilt handles everything:
# - Parse tool_calls from AIMessage
# - Execute tools
# - Create ToolMessages
# - Error handling

tool_node = ToolNode(tools)
```

**Why NOT custom tool execution:**

- Prebuilt is battle-tested

- Handles edge cases (errors, malformed calls)

- Cleaner code

- Standard LangGraph pattern

### Practice: ReAct Research Agent

#### Use Case: Complex Multi-Step Research

``` python
question = "How are AI companies valued in current market?"

# ReAct flow:
# 1. Think: Need both AI trends + financial analysis
# 2. Act: Call ai_research_expert
# 3. Observe: Get AI landscape info
# 4. Think: Now need valuation metrics
# 5. Act: Call financial_analyst
# 6. Observe: Get market multiples
# 7. Think: Have both perspectives
# 8. Final: Synthesize comprehensive answer
```

#### Complete Implementation

See full code in artifact `multi_expert_research`

Key files:

- State definition

- Expert LLM setup

- Tool definitions

- Coordinator + ToolNode

- Graph compilation

- Test cases

#### Testing Multi-Step Reasoning

``` python
# Question requiring iteration
run_research("""
Compare NVIDIA vs AMD for AI workloads.
Consider both technical capabilities and market position.
""")

# Expected flow:
# Iteration 1: ai_research_expert (technical specs)
# Iteration 2: financial_analyst (market analysis)
# Iteration 3: Synthesize comparison
```

## Combining Multi-Expert + ReAct

### Expert Consultation Strategy

#### Sequential Consultation

``` python
# Step 1: Get AI perspective
tool_calls=[{"name": "ai_research_expert", ...}]

# Step 2: Get financial perspective
tool_calls=[{"name": "financial_analyst", ...}]

# Step 3: Synthesize both
```

#### Parallel Consultation (Advanced)

``` python
# Call both experts simultaneously
tool_calls=[
    {"name": "ai_research_expert", ...},
    {"name": "financial_analyst", ...}
]

# ToolNode executes all, returns all ToolMessages
```

### Synthesis Patterns

#### Simple Concatenation

``` python
final_answer = f"""
AI Perspective: {ai_expert_response}

Financial Perspective: {financial_response}

Conclusion: ...
"""
```

#### Structured Synthesis

``` python
coordinator_prompt = """
Synthesize expert inputs into coherent answer.

Expert 1 (AI Research): {ai_response}
Expert 2 (Financial): {fin_response}

Provide: unified analysis addressing all aspects.
"""
```

## Advanced Techniques

### Dynamic Expert Selection

#### LLM-based Routing

``` python
routing_prompt = """
Given question: {question}

Available experts:
- ai_research_expert: ML/AI topics
- financial_analyst: Markets/investments

Which expert(s) should answer? Return JSON list.
"""
```

### Error Handling

#### Tool Execution Failures

``` python
try:
    result = expert_llm.invoke(messages)
except Exception as e:
    return ToolMessage(
        content=f"Expert unavailable: {str(e)}. Using fallback.",
        name=tool_name,
        tool_call_id=call_id
    )
```

#### Invalid Tool Calls

``` python
# ToolNode handles this automatically
# But you can add custom handling:

def safe_tool_node(state):
    try:
        return tool_node.invoke(state)
    except Exception as e:
        return {
            "messages": [AIMessage(content=f"Tool error: {e}. Proceeding without tool result.")]
        }
```

#### Recovery Strategies

## Implementation Best Practices

### Prompt Engineering

#### Clear Expert Roles

``` python
EXPERT_PROMPT = """You are [ROLE].

Specialization:
- [Domain 1]
- [Domain 2]

Response style:
- [Guideline 1]
- [Guideline 2]

Always cite sources when possible.
"""
```

#### Few-shot Examples

``` python
COORDINATOR_PROMPT = """
Example 1:
Question: "Explain transformers"
Thought: Need AI research expertise
Action: call ai_research_expert("transformer architecture")

Example 2:
Question: "Invest in NVDA?"
Thought: Need both AI trends and financial analysis
Action: call ai_research_expert + financial_analyst

Now handle: {question}
"""
```

#### Output Format Specification

``` python
"""
Provide answer in this format:

## Summary
[Brief overview]

## AI Research Perspective
[Expert 1 insights]

## Financial Analysis
[Expert 2 insights]

## Recommendation
[Synthesized conclusion]
"""
```

## Use Cases

### Multi-Expert Tool Pattern

#### Technical + Business Analysis

- Question: “Should we adopt Kubernetes?”

- Experts: DevOps expert + Cost analyst

#### Medical Diagnosis Support

- Question: “Patient symptoms analysis”

- Experts: Specialist doctors (cardiology, neurology)

#### Legal + Financial Advisory

- Question: “M&A implications”

- Experts: Legal counsel + Investment banker

### ReAct Pattern

#### Research Tasks

- Multi-step information gathering

- Iterative refinement of understanding

#### Complex Decision Making

- Evaluate multiple options

- Gather perspectives from different domains

#### Data Analysis

- Query data → Analyze → Refine query → Re-analyze

## Trade-offs

### Benefits

#### Higher Quality

- Expert knowledge \> generic responses

- Structured reasoning \> direct answer

- Multi-perspective synthesis

#### Handles Complexity

- Multi-step problems

- Cross-domain questions

- Nuanced analysis

#### Transparent Reasoning

- See coordinator’s thought process

- Understand which experts consulted

- Debug by inspecting messages

### Costs

#### More LLM Calls

- Each expert = separate LLM invoke

- Coordinator also uses LLM

- Can be 3-5x cost vs single call

#### Higher Latency

- Sequential expert consultations

- Multiple iterations

- Network round-trips

#### Token Usage

- Full conversation history in each call

- System prompts for each expert

- Can hit context limits on long conversations

### When to Use

**Use Multi-Expert ReAct when:**

- Question requires domain expertise

- Need multiple perspectives

- Quality \> Speed/Cost

- Analysis \> Simple facts

**Use Simple Agent when:**

- Straightforward questions

- Single domain

- Speed/Cost critical

- Recent facts (use web search)

## Summary

### Key Takeaways

1.  **Multi-Expert \> Web Search** for analysis questions

2.  **ReAct Pattern** provides structured reasoning

3.  **Prebuilt ToolNode** simplifies implementation

4.  **Message History** is source of truth

5.  **Iteration Control** prevents infinite loops

### Practice: Multi-Expert Research Agent

## Practice: Multi-Expert ReAct Agent with Messages

### Use Case

Agent automatically researches by consulting expert LLMs across multiple iterations:

1.  **User asks question** (HumanMessage)

2.  **Coordinator LLM reasoning** (AIMessage with tool_calls)

3.  **Expert LLMs respond** (ToolMessage from ai_research_expert)

4.  **Coordinator analyzes** (ToolMessage) and reviews

5.  Coordinator **Loops** or **returns final answer**

### Architecture Pattern: ReAct (Reasoning + Acting)

``` mermaid
        graph TB
    A["User Question<br/>(HumanMessage)"]
    B["Coordinator LLM<br/>(thinks &amp; decides)"]
    C["AI Research Expert LLM<br/>(tool)"]
    D["Coordinator observes &amp; reviews"]
    E{More info<br/>needed?}
    F(["END — Final Answer"])

    A --> B --> C --> D --> E
    E -- "Yes: call tool again" --> B
    E -- "No" --> F

    style A fill:#1a2a3a,stroke:#42a5f5,color:#e0e0e0
    style B fill:#1a3a1a,stroke:#4caf50,color:#e0e0e0
    style C fill:#3a2a1a,stroke:#ff9800,color:#e0e0e0
    style D fill:#1a3a1a,stroke:#4caf50,color:#e0e0e0
    style E fill:#3a1a1a,stroke:#f44336,color:#e0e0e0
    style F fill:#2a1a3a,stroke:#ab47bc,color:#e0e0e0
```

### Key Components

#### 1. **State Management**

#### 2. **LLM-based Tools** (not web search!)

- `ai_research_expert(query:`` ``str)` → Call specialized AI Research LLM

- `financial_analyst(query:`` ``str)` → Call specialized Financial LLM

- Each tool is 1 LLM invoke with specialized system prompt

#### 3. **Coordinator Node**

- Main reasoning LLM

- Bind with tools: `coordinator_llm.bind_tools([tool1,`` ``tool2])`

- Decides: which expert to call? or enough info to answer?

#### 4. **Tool Execution**

- Use **prebuilt `ToolNode`** instead of custom implementation

- `ToolNode(tools)` automatically handles tool execution & creates ToolMessages

### Routing Logic

``` python
def should_continue(state: ResearchState) -> str:
    last_message = state["messages"][-1]

    # Has tool calls? → Execute
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "continue"

    # Max iterations? → End
    if state["current_iteration"] >= state["max_iterations"]:
        return "end"

    # Has final answer → End
    return "end"
```

### Best Practices

1.  **System Prompts**: Set clear specialized roles for each expert LLM

2.  **Tool Naming**: Clear descriptive names (`ai_research_expert` not `tool1`)

3.  **Max Iterations**: Set limit (3-5) to avoid infinite loops

4.  **Use ToolNode**: Don’t write custom tool execution - use prebuilt

5.  **State Simplicity**: Only track essentials (messages, iterations)

### Next Steps

#### Add More Experts

``` python
@tool
def legal_expert(query: str) -> str: ...

@tool
def medical_expert(query: str) -> str: ...
```

#### Add Helper Agent (Split out task from Coordinator)

With the current ReAct Agent, Coordinator is doing three tasks:

- Guess and analyze user’s message

- call actions

- Review and return final answer. =\> This can greatly impact the quality of the system since 1 agent/node should not do more than 2 tasks

**Solution:** =\> Introduce **Planning Agent** to handle Guess and analyze user’s message for coordinator, this agent gives out action and reason for coordinator to follow

=\> Introduce **Synthesizer Agent** to generate answer to user if coordinator decided to end.

#### Production Readiness

- Add comprehensive error handling

- Implement rate limiting

- Add caching for expensive calls

- Monitor costs and performance

- A/B test vs simpler approaches

## Agentic RAG Patterns (2025-2026) NEW

The most significant evolution in RAG architecture is the shift from static pipelines to **agentic** patterns where an AI agent decides when, how, and whether to retrieve.

### Corrective RAG (CRAG)

Adds a **retrieval evaluator** after the retrieval step:

``` mermaid
        graph LR
    Q["Query"] --> R["Retrieve"]
    R --> EVAL["Evaluate Relevance"]
    EVAL -->|"Relevant"| GEN["Generate"]
    EVAL -->|"Not Relevant"| WEB["Web Search<br/>(fallback)"]
    WEB --> GEN
    EVAL -->|"Ambiguous"| BOTH["Combine both<br/>sources"]
    BOTH --> GEN
```

If retrieved documents score below a confidence threshold, the agent falls back to web search or re-retrieves with a different query. Production-ready via LangGraph.

### Self-RAG

Embeds **reflection tokens** into the generation process. The model critiques its own output:

1.  `[Retrieve]` — Should I retrieve? (yes/no)

2.  `[IsRel]` — Is the retrieved context relevant?

3.  `[IsSup]` — Is my answer supported by the context?

4.  `[IsUse]` — Is my answer useful to the user?

The model iterates: generate → self-critique → revise → generate again. This produces higher-quality answers but requires more LLM calls.

### Adaptive RAG

A front-end **query classifier** routes queries to different processing paths:

``` mermaid
        graph TD
    Q["Query"] --> CLS["Query Classifier"]
    CLS -->|"Simple factual"| NAIVE["Single-pass RAG"]
    CLS -->|"Multi-hop complex"| AGENTIC["Full Agentic RAG<br/>(iterate, tools, re-rank)"]
    CLS -->|"Conversational"| CONV["Context from history<br/>+ light retrieval"]
```

This avoids wasting resources on simple queries that don’t need the full agentic loop, while still handling complex queries thoroughly.

### Choosing the Right Pattern

| Pattern | Latency | Cost | Quality | Best For |
|----|----|----|----|----|
| Naive RAG | Low | Low | Baseline | Simple Q&A |
| Corrective RAG | Medium | Medium | +15-20% | Systems that need fallback |
| Self-RAG | High | High | +20-30% | High-stakes domains (legal, medical) |
| Adaptive RAG | Varies | Optimized | Best overall | Production systems with mixed query types |

### Resources

- LangGraph ReAct: <https://langchain-ai.github.io/langgraph/how-tos/react-agent-from-scratch/>

- Prebuilt Components: <https://langchain-ai.github.io/langgraph/reference/prebuilt/>

- Message Types: <https://docs.langchain.com/oss/python/langchain/overview>

- Andrew Ng Agentic Patterns: <https://www.youtube.com/watch?v=e2zIr_2JMbE>
