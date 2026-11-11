# Model Context Protocol (MCP) NEW

The Model Context Protocol (MCP) is an open standard initiated by Anthropic in November 2024 and donated to the Linux Foundation’s Agentic AI Foundation in December 2025. It provides a universal interface for connecting LLMs to external tools, data sources, and services. Think of MCP as “USB-C for AI”: just as USB-C standardized device connectivity, MCP standardizes how AI applications connect to the outside world — regardless of which model or provider is involved.

## Learning Objectives

- Understand what MCP is and why it emerged as a standard

- Distinguish between MCP servers, clients, and transports

- Identify the major adopters and the current ecosystem scale

- Build a basic MCP server and use the MCP Connector API

- Compare MCP to provider-specific function calling

## 1. Why MCP?

Before MCP, every AI tool integration was a custom one-off. OpenAI had function calling. Anthropic had tool use. Each application built its own REST wrappers, schema translators, and error handlers. The result was an N×M integration problem: every combination of AI application and tool required bespoke work.

MCP solves this with a single shared interface. Servers expose capabilities once; clients consume them anywhere.

``` mermaid
        graph LR
    subgraph Without_MCP["Without MCP — N × M integrations"]
        A1[App 1] --> T1[Tool A]
        A1 --> T2[Tool B]
        A1 --> T3[Tool C]
        A2[App 2] --> T1
        A2 --> T2
        A2 --> T3
        A3[App 3] --> T1
        A3 --> T2
        A3 --> T3
    end
```

``` mermaid
        graph LR
    subgraph With_MCP["With MCP — N + M integrations"]
        B1[App 1] --> MCP[MCP Layer]
        B2[App 2] --> MCP
        B3[App 3] --> MCP
        MCP --> S1[Tool A]
        MCP --> S2[Tool B]
        MCP --> S3[Tool C]
    end
```

The combinatorial explosion collapses: add a new tool once, and every MCP-compatible client can use it. Add a new client once, and it gains access to every MCP server.

## 2. Architecture

MCP defines three components: servers, clients (hosts), and a transport layer between them.

``` mermaid
        graph LR
    HOST["MCP Client / Host\n(Claude Desktop, VS Code, Claude Code)"]
    TRANSPORT["Transport Layer\n(stdio or Streamable HTTP)"]
    SERVER["MCP Server\n(GitHub, Database, Slack…)"]

    HOST <-->|"JSON-RPC messages"| TRANSPORT
    TRANSPORT <-->|"tool calls / responses"| SERVER
```

### MCP Servers

An MCP server exposes **tools**, **resources**, and **prompts** to any connected client. Tools are callable functions (like `get_weather` or `search_database`). Resources are read-only data endpoints (configuration, file contents). Prompts are reusable prompt templates the server can offer.

As of early 2026, there are over 10,000 public MCP servers covering:

- Source control: GitHub, GitLab, Bitbucket

- Databases: PostgreSQL, MySQL, SQLite, MongoDB

- Communication: Slack, Gmail, Google Calendar

- Cloud: AWS, Cloudflare Workers, Kubernetes

- Developer tools: VS Code, Cursor, browser automation

### MCP Clients (Hosts)

The MCP client is the LLM application that initiates the connection and invokes tools on behalf of the model. Examples include Claude Desktop, Claude Code, VS Code (built-in since July 2025), Cursor, and any application built on top of the Anthropic API using the MCP Connector feature.

Clients handle the protocol handshake, capability negotiation, and routing of tool results back to the model.

### Transport Layer

MCP specifies two transports:

| Transport | Use Case | Notes |
|----|----|----|
| **stdio** | Local servers launched as subprocesses | Fast, zero network overhead, used by CLI tools |
| **Streamable HTTP** | Remote servers over the network | Replaced the deprecated HTTP+SSE transport; supports OAuth |

Local servers are the most common pattern for development tools. Remote servers enable hosted integrations that multiple users and applications can share.

## 3. What MCP Servers Expose

Each MCP server declares its capabilities during the initial handshake. The protocol defines three primitive types:

**Tools** — Callable functions the LLM can invoke:

``` json
{
  "name": "get_weather",
  "description": "Get the current weather for a city.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "city": { "type": "string" }
    },
    "required": ["city"]
  }
}
```

**Resources** — Read-only data the client can fetch:

```
config://settings
file:///project/README.md
db://schema/users
```

**Prompts** — Named prompt templates for common tasks:

``` json
{
  "name": "summarize-pr",
  "description": "Summarize a pull request for a code review",
  "arguments": [{ "name": "pr_url", "required": true }]
}
```

## 4. Adoption

MCP went from an Anthropic internal tool to an industry-wide standard in under 18 months:

| Adopter | When | How |
|----|----|----|
| Anthropic (Claude) | Nov 2024 | Native in Claude Desktop, Claude Code |
| OpenAI | Mar 2025 | Added MCP support to ChatGPT and Agents SDK |
| Google DeepMind | 2025 | Gemini MCP integration |
| VS Code | Jul 2025 | Built-in MCP with OAuth and a server marketplace |
| Cursor | Late 2025 | MCP in Cursor IDE, replacing custom tool protocol |
| Linux Foundation | Dec 2025 | MCP donated to the Agentic AI Foundation for neutral governance |

The Linux Foundation donation was significant: it means no single vendor controls the specification. MCP now has the same governance model as other foundational open standards.

## 5. Building an MCP Server

The official Python SDK (`mcp`) provides a `FastMCP` class that handles all protocol details. You declare tools with decorators and resources with URI templates.

``` python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather-server")


@mcp.tool()
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    # In production, call a real weather API
    return f"Weather in {city}: 25°C, sunny"


@mcp.tool()
def get_forecast(city: str, days: int = 3) -> list[dict]:
    """Get a multi-day weather forecast for a city."""
    return [
        {"day": i + 1, "city": city, "temp_c": 24 + i, "condition": "partly cloudy"}
        for i in range(days)
    ]


@mcp.resource("config://settings")
def get_settings() -> str:
    """Return server configuration."""
    return "temperature_unit: celsius\nmax_forecast_days: 7"


if __name__ == "__main__":
    mcp.run()  # Defaults to stdio transport
```

To run the server with Streamable HTTP for remote access:

``` python
if __name__ == "__main__":
    mcp.run(transport="streamable-http", host="0.0.0.0", port=8080)
```

The server can then be registered in a client’s configuration:

``` json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["weather_server.py"]
    }
  }
}
```

## 6. Using the MCP Connector (Anthropic API)

Anthropic’s MCP Connector lets the Claude API call remote MCP servers directly — no separate client process required. This is the simplest way to add MCP capabilities to an application built on the Anthropic SDK.

``` python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    mcp_servers=[
        {
            "type": "url",
            "url": "https://my-mcp-server.example.com/mcp",
            "name": "my-tools",
        }
    ],
    messages=[
        {"role": "user", "content": "What's the weather in Hanoi?"}
    ],
)

print(response.content[0].text)
```

The API handles the MCP handshake, tool discovery, tool invocation, and result injection transparently. The model sees the tool results the same way it sees any other tool use response.

For servers that require authentication, the connector supports OAuth 2.0 headers:

``` python
mcp_servers=[
    {
        "type": "url",
        "url": "https://api.example.com/mcp",
        "name": "authenticated-server",
        "authorization_token": "Bearer <your-token>",
    }
]
```

## 7. Key Insight: Code Execution with MCP

Anthropic engineering discovered an important optimization when working with large MCP ecosystems. Loading the full schema of 100 tools into every system prompt is expensive — in tokens and in model attention. Instead, agents can write a small script that calls exactly the tools they need.

In benchmarks, this approach reduced token consumption by up to **98.7%** compared to loading all tool definitions upfront. The pattern works like this:

``` mermaid
        sequenceDiagram
    participant User
    participant Agent
    participant CodeExec as Code Executor
    participant MCP as MCP Server

    User->>Agent: "Summarize all open PRs from last week"
    Agent->>CodeExec: Write and run a script that calls GitHub MCP tools
    CodeExec->>MCP: list_pull_requests(state="open", since="7d ago")
    MCP-->>CodeExec: [PR data]
    CodeExec-->>Agent: Execution result
    Agent-->>User: Summary
```

Rather than describing every tool in the prompt, the agent generates code that imports and calls tools on demand. This is a significant architectural pattern for production agentic systems with large tool catalogs.

## 8. MCP vs Function Calling

MCP is not a replacement for function calling — it is a standardized layer on top of it. The comparison clarifies what each layer handles:

| Aspect | Provider Function Calling | MCP |
|----|----|----|
| **Scope** | Single provider (OpenAI, Anthropic) | Universal open standard |
| **Tool discovery** | Manual schema definitions in each request | Dynamic discovery via protocol handshake |
| **Transport** | Provider-specific API | stdio or Streamable HTTP |
| **Governance** | Vendor-controlled | Linux Foundation (Agentic AI Foundation) |
| **Ecosystem** | Provider-specific integrations | 10,000+ public servers, cross-vendor |
| **Portability** | Tied to one provider’s format | Works with any MCP-compatible client |
| **State** | Stateless per API call | Sessions can be stateful across calls |

The practical implication: if you build a tool as an MCP server, it works with Claude, ChatGPT, Gemini, VS Code, Cursor, and any future MCP-compatible client. If you build it as a vendor-specific function definition, you own the translation work when you switch or add providers.

## 9. Security Considerations

MCP servers run with the permissions of the process or credentials they are given. Before deploying any MCP server in production, verify:

- **Least privilege**: The server process should have only the permissions it needs (read-only database credentials, scoped API tokens)

- **Input validation**: Tool arguments come from the LLM, which can be manipulated via prompt injection — validate and sanitize all inputs

- **Authentication**: Remote servers must require authentication; never expose an MCP server publicly without access controls

- **Audit logging**: Log all tool invocations with the request, the caller, and the result for traceability

- **Rate limiting**: Apply rate limits per client to prevent runaway agents from exhausting external API quotas

Prompt injection is a specific risk: malicious content in a data source (a web page, a document, a database row) can instruct the LLM to call tools with attacker-controlled arguments. Defense requires treating all retrieved content as untrusted user input.

## Summary

| Concept          | Key Point                                         |
|------------------|---------------------------------------------------|
| What MCP is      | Open standard for LLM ↔ tool connectivity         |
| Origin           | Anthropic (Nov 2024), now Linux Foundation        |
| Server           | Exposes tools, resources, and prompts             |
| Client           | LLM application that calls the server             |
| Transport        | stdio (local) or Streamable HTTP (remote)         |
| Ecosystem        | 10,000+ public servers as of early 2026           |
| Token efficiency | Code-execution pattern cuts tokens by up to 98.7% |
| Governance       | Neutral, vendor-independent (Linux Foundation)    |

MCP represents a maturation of the AI tooling ecosystem: from ad-hoc integrations to a shared, governed, interoperable standard. For FPT audit interns, understanding MCP is increasingly essential — both because audit tools will be exposed via MCP and because clients like Claude Code and VS Code already use it as their primary extension mechanism.
