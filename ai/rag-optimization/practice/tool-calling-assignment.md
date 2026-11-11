# Assignment: Tool Calling & Tavily Search Integration

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Tool Calling & Tavily Search Integration |
| **Course** | LangGraph and Agentic AI |
| **Project Name** | `tavily-research-agent` |
| **Estimated Time** | 90 minutes |
| **Framework** | Python 3.11+, LangGraph 1.x, LangChain 1.x, Tavily API, OpenAI API |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Understand** Tool/Function Calling mechanics in LLMs

- **Integrate** Tavily Search API for real-time web information

- **Build** agents with multiple tools working in parallel

- **Implement** error handling and retry logic for tool execution

- **Apply** caching and rate limiting for production readiness

------------------------------------------------------------------------

## Problem Description

Building on the Multi-Expert Research Agent from Assignment 02, you will add a **Web Search tool** using Tavily API. This enables the agent to:

1.  Access real-time information beyond LLM knowledge cutoff

2.  Combine expert analysis with current web data

3.  Execute multiple tools in parallel when needed

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langgraph` \>= 1.0

  - `langchain[openai]` \>= 1.0

  - `langchain-tavily` \>= 0.1.0

### API Requirements

- OpenAI API key (`OPENAI_API_KEY`)

- Tavily API key (`TAVILY_API_KEY`) - Get free at [tavily.com](https://www.tavily.com/)

------------------------------------------------------------------------

## Tasks

### Task 1: Tavily Search Tool Setup (25 points)

1.  **Configure TavilySearch** tool with:

    - `max_results`: 5

    - `search_depth`: “advanced”

    - `include_answer`: True (for AI-generated summary)

2.  **Create a wrapper tool** with proper description:

    - Clear description of when to use (current events, real-time data)

    - Specify input format expectations

    - Handle search errors gracefully

### Task 2: Multi-Tool Agent Integration (35 points)

1.  **Extend existing Research Agent** to include:

    - AI Research Expert (from Assignment 02)

    - Financial Analyst (from Assignment 02)

    - Web Search Tool (new Tavily integration)

2.  **Update coordinator prompt** to:

    - Advise when to use web search vs expert LLMs

    - Enable parallel tool calling for independent queries

    - Guide synthesis of web results with expert analysis

3.  **Ensure parallel execution** capability:

    - Coordinator can call multiple tools in single response

    - ToolNode executes all tool_calls and returns all ToolMessages

### Task 3: Error Handling & Optimization (25 points)

1.  **Implement retry logic** for Tavily API:

    - Use `tenacity` for exponential backoff

    - Handle rate limits and timeouts gracefully

2.  **Add basic caching** for search results:

    - Cache identical queries for 1 hour

    - Use hash-based cache keys

3.  **Implement rate limiting**:

    - Max 10 searches per minute

    - Queue overflow handling

### Task 4: Testing & Validation (15 points)

1.  **Test queries requiring web search**:

    - “What are the latest AI announcements this week?”

    - “Current NVIDIA stock performance and recent news”

2.  **Test combined expert + web search**:

    - “Analyze the impact of recent AI regulations on tech companies”

3.  **Verify parallel tool execution**:

    - Show both expert and search tools called simultaneously

    - Document execution time comparison (parallel vs sequential)

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code in Python script or Jupyter notebook

`README.md` with setup instructions and API key configuration

Test output showing web search integration

Comparison of response quality with/without web search

### Submission Checklist

Tavily tool configured with proper parameters

All three tools work together (2 experts + web search)

Parallel tool execution demonstrated

Error handling implemented for API failures

Code runs without errors

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                       | Points  |
|--------------------------------|---------|
| Tavily Search tool setup       | 25      |
| Multi-tool integration         | 35      |
| Error handling & optimization  | 25      |
| Testing & validation           | 10      |
| Code quality and documentation | 5       |
| **Total**                      | **100** |

------------------------------------------------------------------------

## Hints

- Use `TavilySearch` from `langchain_tavily` (the newer `langchain-tavily` package; the older `langchain_community.tools.tavily_search` path is deprecated)

- For parallel calls, the coordinator returns multiple tool_calls in one AIMessage

- Store Tavily API key in environment variable, never hardcode

- Use `@retry(stop=stop_after_attempt(3))` from tenacity for retries

------------------------------------------------------------------------

## References

- [Tavily Search Documentation](https://docs.tavily.com/welcome)

- [LangChain Tavily Integration](https://docs.langchain.com/oss/python/integrations/providers/overview)

- [Tenacity Retry Library](https://tenacity.readthedocs.io/en/latest/)
