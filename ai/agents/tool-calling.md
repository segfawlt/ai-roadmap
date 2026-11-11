# Tool Calling & Tavily Search

This page explains how LLMs decide when and how to invoke external tools using structured function calling, with practical examples using the Tavily Search API to give agents access to real-time web information.

## Learning Objectives

- Understand Tool/Function Calling in LLMs and how it works

- Integrate external tools with agents effectively

- Use Tavily Search API for optimized web search

- Build agent with multiple tools and handle tool orchestration

------------------------------------------------------------------------

## 1. What is Tool Calling?

### 1.1 Concept

Tool Calling (or Function Calling) is the ability of LLM to:

- **LLM decides when to use tools**: Model automatically decides when to call tool based on user query

- **Structured tool invocation**: Call tool with parameters formatted in standard format (JSON schema)

- **Parse tool results**: Receive and process results returned from tool

- **Continue reasoning**: Continue reasoning with new information to create final response

**Basic Flow:**

``` mermaid
        graph LR
    UQ["User Query"] --> LLM["LLM Analyzes"]
    LLM --> DEC["Decides to Use Tool"]
    DEC --> CALL["Calls Tool<br/>with Params"]
    CALL --> EXEC["Tool Executes"]
    EXEC --> RES["Returns Result"]
    RES --> PROC["LLM Processes"]
    PROC --> FINAL["Final Response"]
```

### 1.2 Why Tool Calling?

- **Extend LLM capabilities**: Overcome knowledge cutoff and training data limitations

- **Access real-time data**: Get real-time information (weather, stock prices, news)

- **Perform actions**: Perform real actions (send email, create ticket, update database)

- **Integrate with APIs**: Connect with external services and third-party APIs

### 1.3 Function Calling vs Tool Use

| **Function Calling**            | **Tool Use**                    |
|---------------------------------|---------------------------------|
| OpenAI terminology              | LangChain/Anthropic terminology |
| JSON schema for functions       | Tool interface with description |
| Returns a function call object  | Returns a tool invocation       |
| Commonly used for OpenAI models | Framework-agnostic approach     |

------------------------------------------------------------------------

## 2. OpenAI Tool Calling (Modern SDK)

### 2.1 API overview

Define tools with JSON schema. Note: OpenAI replaced the old `functions=` parameter with a unified `tools=` parameter; each tool is wrapped in a `{"type":`` ``"function",`` ``"function":`` ``{...}}` envelope:

``` python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "Search the web for current information about a topic",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "The search query"
                    },
                    "num_results": {
                        "type": "integer",
                        "description": "Number of results to return",
                        "default": 5
                    }
                },
                "required": ["query"]
            }
        }
    }
]
```

### 2.2 Request

``` python
from openai import OpenAI

client = OpenAI()  # picks up OPENAI_API_KEY from the environment

response = client.chat.completions.create(
    model="gpt-5.2",
    messages=[
        {"role": "user", "content": "What's the weather in Hanoi?"}
    ],
    tools=tools,
    tool_choice="auto",  # "auto", "none", "required", or {"type": "function", "function": {"name": "..."}}
)
```

### 2.3 Response

``` python
# response.choices[0].message looks like:
{
    "role": "assistant",
    "content": None,
    "tool_calls": [
        {
            "id": "call_abc123",
            "type": "function",
            "function": {
                "name": "search_web",
                "arguments": '{"query": "weather Hanoi today"}'
            }
        }
    ]
}
```

### 2.4 Executing the tool

``` python
import json

assistant_msg = response.choices[0].message
tool_call = assistant_msg.tool_calls[0]
function_name = tool_call.function.name
arguments = json.loads(tool_call.function.arguments)

if function_name == "search_web":
    result = search_web(**arguments)
```

### 2.5 Continuation — send the tool result back

``` python
messages.append(assistant_msg)  # preserve the tool_call in history
messages.append({
    "role": "tool",
    "tool_call_id": tool_call.id,
    "content": json.dumps(result),
})

final_response = client.chat.completions.create(
    model="gpt-5.2",
    messages=messages,
)
```

> **Note**: OpenAI’s newer **Responses API** (`client.responses.create(...)`) is now the recommended surface for agent-style workloads. The Chat Completions API shown above is still fully supported and is the simplest path for understanding the tool-calling loop.

------------------------------------------------------------------------

## 3. LangChain Tools

### 3.1 Tool Interface

``` python
from langchain.tools import Tool

def search_function(query: str) -> str:
    """Search implementation"""
    # Call actual search API
    results = tavily_client.search(query)
    return str(results)

search_tool = Tool(
    name="WebSearch",
    func=search_function,
    description="Useful for searching the web for current information. Input should be a search query string."
)
```

### 3.2 @tool Decorator

``` python
from langchain.tools import tool

@tool
def calculator(expression: str) -> str:
    """Useful for performing mathematical calculations.
    Input should be a valid Python mathematical expression."""
    try:
        result = eval(expression)
        return f"Result: {result}"
    except Exception as e:
        return f"Error: {str(e)}"

@tool
def get_current_time(timezone: str = "UTC") -> str:
    """Get current time in specified timezone.
    Input should be timezone string like 'UTC', 'Asia/Ho_Chi_Minh'."""
    from datetime import datetime
    import pytz
    tz = pytz.timezone(timezone)
    return datetime.now(tz).strftime("%Y-%m-%d %H:%M:%S %Z")
```

### 3.3 Built-in Tools

``` python
# DuckDuckGo Search
from langchain_community.tools import DuckDuckGoSearchResults
search = DuckDuckGoSearchResults()

# Wikipedia
from langchain_community.tools import WikipediaQueryRun
from langchain_community.utilities import WikipediaAPIWrapper
wikipedia = WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())

# Python REPL
from langchain_community.tools import PythonREPLTool
python_repl = PythonREPLTool()

# File Management
from langchain_community.tools import ReadFileTool, WriteFileTool
read_file = ReadFileTool()
write_file = WriteFileTool()
```

------------------------------------------------------------------------

## 4. Tavily Search API

### 4.1 Introduction to Tavily

Tavily is a search engine optimized for AI:

- **AI-optimized search engine**: Results are pre-formatted for LLMs

- **Designed for LLMs and RAG**: Easy integration with AI workflows

- **Clean, relevant results**: Filters out noise, only returns quality content

- **Real-time web search**: Information updated in real-time

### 4.2 Features

- **Web search**: Search entire web with optimized ranking

- **News search**: Specialized for latest news

- **Answer mode**: Returns direct answers instead of a list of links

- **Context optimization**: Optimize context for RAG applications

### 4.3 Getting Started

``` bash
# 1. Sign up at tavily.com
# 2. Get API key from dashboard
# 3. Install SDK
pip install tavily-python
```

### 4.4 Basic Usage

``` python
from tavily import TavilyClient

# Initialize client
client = TavilyClient(api_key="tvly-xxxxxxxxxxxxx")

# Basic search
response = client.search(
    query="LangGraph tutorial 2025"
)

print(response["results"])
```

### 4.5 Search Parameters

``` python
response = client.search(
    query="climate change solutions",           # Required
    search_depth="advanced",                    # "basic" or "advanced"
    max_results=10,                             # Max number of results
    include_domains=["edu", "gov"],             # Filter domains
    exclude_domains=["example.com"],            # Block domains
    include_answer=True,                        # Get AI-generated answer
    include_raw_content=False,                  # Include full page content
    include_images=True                         # Include image URLs
)
```

### 4.6 Response Structure

``` json
{
  "query": "climate change solutions",
  "follow_up_questions": null,
  "answer": "Several effective climate change solutions include...",
  "images": ["https://...", "https://..."],
  "results": [
    {
      "title": "10 Solutions for Climate Change",
      "url": "https://example.com/article",
      "content": "Clean summary of the page content...",
      "score": 0.98,
      "raw_content": null
    }
  ],
  "response_time": 1.23
}
```

------------------------------------------------------------------------

## 5. TavilySearch Tool

### 5.1 LangChain Integration

Tavily ships an official LangChain integration via the `langchain-tavily` package (the older `langchain_community.tools.tavily_search` path is deprecated):

``` bash
pip install -U langchain-tavily
```

``` python
import os
from langchain_tavily import TavilySearch

# Set API key
os.environ["TAVILY_API_KEY"] = "tvly-xxxxxxxxxxxxx"

# Create the tool
search = TavilySearch(
    max_results=5,
    search_depth="advanced",
    include_answer=True,
    include_raw_content=False,
    include_domains=[],
    exclude_domains=[],
)

# Use the tool
result = search.invoke({"query": "latest AI developments"})
```

Drop the resulting `search` tool into any `create_agent(...,`` ``tools=[search])` call to give the agent live web access.

## 6. Advanced Tool Patterns

### 6.1 Tool Chaining

Output of one tool becomes input for another:

``` python
@tool
def search_company(company_name: str) -> str:
    """Search for company information"""
    return search.invoke(f"{company_name} official website")

@tool
def get_stock_price(ticker: str) -> str:
    """Get stock price from ticker symbol"""
    # ticker extracted from search_company result
    return f"${price}"

# Chain: company name → search → extract ticker → get price
```

## 7. Error Handling

### 7.1 API Failures

``` python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def call_tavily_with_retry(query):
    try:
        return client.search(query)
    except Exception as e:
        print(f"API call failed: {e}")
        raise
```

### 7.2 Timeout Handling

``` python
import asyncio

async def execute_with_timeout(tool_func, timeout=30):
    try:
        result = await asyncio.wait_for(
            tool_func(),
            timeout=timeout
        )
        return result
    except asyncio.TimeoutError:
        return "Tool execution timeout"
```

------------------------------------------------------------------------

## 8. Optimization

### 8.1 Caching Tool Results

``` python
from functools import lru_cache
import hashlib

# Simple LRU cache
@lru_cache(maxsize=100)
def cached_search(query: str):
    return client.search(query)

from datetime import datetime, timedelta

cache = {}
CACHE_TTL = timedelta(hours=1)

def search_with_cache(query):
    cache_key = hashlib.md5(query.encode()).hexdigest()

    if cache_key in cache:
        result, timestamp = cache[cache_key]
        if datetime.now() - timestamp < CACHE_TTL:
            return result

    result = client.search(query)
    cache[cache_key] = (result, datetime.now())
    return result
```

### 8.2 Rate Limiting

``` python
from ratelimit import limits, sleep_and_retry

# Max 10 calls per minute
@sleep_and_retry
@limits(calls=10, period=60)
def rate_limited_search(query):
    return client.search(query)
```

### 8.3 Batching Requests

``` python
async def batch_search(queries: list[str]):
    """Execute multiple searches efficiently"""
    tasks = [client.search_async(q) for q in queries]
    results = await asyncio.gather(*tasks)
    return results
```

------------------------------------------------------------------------

## 9. Custom Tools

### 9.1 Creating Custom Tool

``` python
from langchain.tools import BaseTool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="Search query")
    domain: str = Field(description="Specific domain to search")

class DomainSearchTool(BaseTool):
    name = "domain_search"
    description = "Search within a specific domain"
    args_schema = SearchInput

    def _run(self, query: str, domain: str) -> str:
        """Synchronous implementation"""
        full_query = f"site:{domain} {query}"
        return client.search(full_query)

    async def _arun(self, query: str, domain: str) -> str:
        """Async implementation"""
        full_query = f"site:{domain} {query}"
        return await client.search_async(full_query)
```

------------------------------------------------------------------------

## 10. Security Considerations

### 10.1 API Key Management

``` python
# ❌ Never hardcode
client = TavilyClient(api_key="tvly-xxxxx")

# ✅ Use environment variables
import os
from dotenv import load_dotenv

load_dotenv()
client = TavilyClient(api_key=os.getenv("TAVILY_API_KEY"))

# ✅ Use secret management
from azure.keyvault.secrets import SecretClient
api_key = secret_client.get_secret("tavily-api-key").value
```

## 11. Best Practices

### 11.1 Tool Descriptions

``` python
# ❌ Vague description
@tool
def search(query: str) -> str:
    """Search the web"""
    pass

# ✅ Clear, detailed description
@tool
def search(query: str) -> str:
    """
    Search the web for current information using Tavily API.
    Best used for: recent news, current events, factual information.
    Input should be a clear, specific search query.
    Returns: Top 5 relevant web results with summaries.
    """
    pass
```

## Structured Outputs & Modern Tool Patterns NEW

### Structured Outputs (JSON Schema Enforcement)

Modern LLMs now support **constrained decoding** — guaranteeing that outputs conform to a JSON schema:

- **OpenAI**: `response_format:`` ``{"type":`` ``"json_schema",`` ``"json_schema":`` ``{...},`` ``"strict":`` ``true}` — uses constrained decoding for 100% schema compliance

- **Anthropic Claude**: Uses `tool_use` with high reliability; also supports JSON mode via system prompts

- **Self-hosted (vLLM)**: XGrammar backend supports JSON Schema, Pydantic, and regex constraints at inference time

``` python
# OpenAI Structured Output with Pydantic
from pydantic import BaseModel
from openai import OpenAI

class CompanyInfo(BaseModel):
    name: str
    industry: str
    revenue_usd: float
    employee_count: int

client = OpenAI()
result = client.beta.chat.completions.parse(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Tell me about Anthropic"}],
    response_format=CompanyInfo,
)
company = result.choices[0].message.parsed  # CompanyInfo instance
```

Decision guide: pure data extraction → structured outputs; agentic multi-action → function/tool calling.

### Model Context Protocol (MCP)

MCP is an open standard (initiated by Anthropic, now governed by the Linux Foundation) that provides a universal interface for connecting LLMs to external tools and data sources.

Key concepts:

- **MCP Server**: Exposes tools and resources via a standardized protocol

- **MCP Client**: The LLM host (Claude, VS Code, Cursor) that connects to servers

- **Transport**: Communication layer (Streamable HTTP for remote, stdio for local)

``` mermaid
        graph LR
    LLM["LLM Client<br/>(Claude, Cursor, VS Code)"] <-->|"MCP Protocol"| S1["MCP Server<br/>GitHub"]
    LLM <-->|"MCP Protocol"| S2["MCP Server<br/>Database"]
    LLM <-->|"MCP Protocol"| S3["MCP Server<br/>Slack"]
```

Why MCP matters:

- **10,000+ public MCP servers** as of early 2026

- Adopted by OpenAI, Google, VS Code, Cursor, Windsurf, GitHub Copilot

- Replaces one-off API integrations with a universal standard

- Think of it as “USB-C for AI tools” — one protocol to connect everything

MCP is covered in detail in the Advanced tier.

### Parallel Tool Execution

Modern agents can invoke multiple tools simultaneously when the tools are independent:

``` python
# LangGraph parallel tool execution
from langgraph.prebuilt import ToolNode

tool_node = ToolNode([search_tool, calculator_tool, weather_tool])
# When the LLM returns multiple tool_calls in one response,
# ToolNode executes them in parallel automatically
```

This significantly reduces latency for multi-tool queries (e.g., “What’s the weather AND stock price?”).

------------------------------------------------------------------------

## 12. PRACTICE

- From Research Agent in module 02, add another agent called **Web Search tool** (Tavily) to handle task for web search request AND coordinator must advise user to use web search for research request.

  - Web search tool must be able to call in parallel along with other tools
