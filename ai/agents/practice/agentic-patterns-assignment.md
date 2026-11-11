# Assignment: Multi-Expert ReAct Research Agent

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Multi-Expert ReAct Research Agent |
| **Course** | LangGraph and Agentic AI |
| **Project Name** | `react-research-agent` |
| **Estimated Time** | 120 minutes |
| **Framework** | Python 3.11+, LangGraph 1.x, LangChain 1.x, Anthropic (or OpenAI) API |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Understand** the ReAct pattern (Reasoning + Acting) for agent design

- **Implement** LLM-based expert tools instead of simple web search

- **Use** LangGraph’s prebuilt ToolNode for tool execution

- **Apply** iteration control to prevent infinite loops

- **Design** coordinator prompts that enable structured reasoning

------------------------------------------------------------------------

## Problem Description

You are building a research agent that can consult multiple domain experts to answer complex questions. Instead of using web search, the agent uses specialized LLM experts:

- **AI Research Expert**: Specializes in ML papers, architectures, trends

- **Financial Analyst**: Specializes in stocks, markets, valuations

The coordinator LLM uses ReAct pattern to decide which expert(s) to consult and synthesize their responses.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langgraph` \>= 1.0

  - `langchain[anthropic]` \>= 1.0 \# or `langchain[openai]`

### API Requirements

- Anthropic API key (or OpenAI API key) configured as environment variable

------------------------------------------------------------------------

## Tasks

### Task 1: Expert Tool Definition (25 points)

1.  **Create AI Research Expert tool** that:

    - Uses a specialized system prompt for ML/AI topics

    - Invokes an LLM with research-focused instructions

    - Returns structured analysis with paper-style references

2.  **Create Financial Analyst tool** that:

    - Uses a specialized system prompt for market analysis

    - Provides data-driven insights with market context

    - Returns actionable financial information

### Task 2: Coordinator Implementation (30 points)

1.  **Define ResearchState** with:

    - `messages`: For conversation history

    - `max_iterations`: Limit for ReAct loops (default: 5)

    - `current_iteration`: Tracking current loop count

2.  **Implement coordinator_node** that:

    - Binds expert tools to the coordinator LLM

    - Includes ReAct reasoning in the prompt

    - Increments iteration counter on each call

3.  **Create routing function** (`should_continue`) that:

    - Checks for tool_calls in the last message

    - Enforces max_iterations limit

    - Routes to “tools” or “end” appropriately

### Task 3: Graph Construction (25 points)

1.  **Use prebuilt ToolNode** for tool execution (NOT custom implementation)

2.  **Build the graph** with:

    - Coordinator node as entry point

    - ToolNode for expert execution

    - Conditional edges for ReAct loop

    - Edge from tools back to coordinator

3.  **Configure checkpointer** for state persistence

### Task 4: Testing Multi-Step Reasoning (20 points)

1.  **Test with complex queries** requiring multiple experts:

    - “How are AI companies valued in the current market?”

    - “Compare NVIDIA vs AMD for AI workloads - technical and market position”

2.  **Verify ReAct flow**:

    - Iteration 1: First expert consultation

    - Iteration 2+: Additional consultations or synthesis

    - Final: Comprehensive answer combining perspectives

3.  **Document the reasoning chain** showing Think → Act → Observe steps

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code in Python script or Jupyter notebook

`README.md` with setup instructions

Test output showing multi-step ReAct reasoning

Graph visualization

### Submission Checklist

Both expert tools implemented with specialized prompts

ToolNode from langgraph.prebuilt is used (not custom)

Iteration control prevents infinite loops

Coordinator successfully synthesizes multi-expert input

Code runs without errors

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                         | Points  |
|----------------------------------|---------|
| Expert tool implementation       | 25      |
| Coordinator with ReAct prompt    | 30      |
| Graph construction with ToolNode | 25      |
| Multi-step reasoning tests       | 15      |
| Code quality and documentation   | 5       |
| **Total**                        | **100** |

------------------------------------------------------------------------

## Hints

- Use `llm.bind_tools([tool1,`` ``tool2])` to enable tool calling for the coordinator

- The prebuilt `ToolNode(tools)` handles parsing, execution, and ToolMessage creation

- Include few-shot examples in coordinator prompt to guide tool selection

- Set `temperature=0.3` for expert LLMs for consistent responses

------------------------------------------------------------------------

## References

- [LangGraph ReAct Pattern](https://langchain-ai.github.io/langgraph/how-tos/react-agent-from-scratch/)

- [Prebuilt ToolNode](https://github.com/langchain-ai/langgraph/blob/main/libs/prebuilt/langgraph/prebuilt/tool_node.py)

- [Andrew Ng Agentic Patterns](https://www.youtube.com/watch?v=e2zIr_2JMbE)
