# LLM API Cheatsheet

All examples here are verified against the canonical vendor docs as of 2026-04-10. If you are using LangChain, prefer `from`` ``langchain.chat_models`` ``import`` ``init_chat_model` with a provider-prefix string (`"anthropic:claude-sonnet-4-6"`) instead of these raw SDKs — see [LangGraph Foundations & State Management](../agents/langgraph-foundations.md).

## Anthropic (Claude)

### Install

``` bash
pip install anthropic
```

Python 3.9+ required. Set `ANTHROPIC_API_KEY`.

### Basic message

``` python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
)
print(message.content[0].text)
```

### Current model IDs

| Model      | ID                  | Context | Max out | Cost (in/out per MTok) |
|------------|---------------------|---------|---------|------------------------|
| Opus 4.6   | `claude-opus-4-6`   | 1M      | 128k    | `\(5 / \)`25           |
| Sonnet 4.6 | `claude-sonnet-4-6` | 1M      | 64k     | `\(3 / \)`15           |
| Haiku 4.5  | `claude-haiku-4-5`  | 200k    | 64k     | `\(1 / \)`5            |

### Tool use (client tools)

``` python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=[{
        "name": "get_weather",
        "description": "Get current weather for a city",
        "input_schema": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
        "strict": True,
    }],
    messages=[{"role": "user", "content": "Weather in SF?"}],
)
# response.content contains tool_use blocks — execute and send back tool_result
```

### Server tools (Anthropic runs them)

Available types: `web_search_20260209`, `code_execution`, `web_fetch`, `tool_search`.

``` python
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    tools=[{"type": "web_search_20260209", "name": "web_search"}],
    messages=[{"role": "user", "content": "Latest on Mars rover?"}],
)
```

### Extended thinking

``` python
# Opus 4.6 / Sonnet 4.6 — adaptive thinking (budget_tokens is deprecated)
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={"type": "adaptive"},
    messages=[{"role": "user", "content": "Is there an infinite number of primes p with p mod 4 == 3?"}],
)

# Older models — explicit budget
client.messages.create(
    model="claude-haiku-4-5",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 10000},
    messages=[...],
)
```

### Streaming

``` python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a haiku"}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

## OpenAI

### Install

``` bash
pip install openai
```

Set `OPENAI_API_KEY`.

### Responses API (primary in current docs)

``` python
from openai import OpenAI
client = OpenAI()

response = client.responses.create(
    model="gpt-5.2",
    input="Why do parrots talk?",
)
print(response.output_text)
```

### Chat Completions (legacy but supported)

``` python
completion = client.chat.completions.create(
    model="gpt-5.2",
    messages=[{"role": "user", "content": "Hello"}],
)
print(completion.choices[0].message.content)
```

### Tool use (Chat Completions shape)

``` python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get current weather",
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
    },
}]

response = client.chat.completions.create(
    model="gpt-5.2",
    messages=[{"role": "user", "content": "Weather in SF?"}],
    tools=tools,
    tool_choice="auto",
)
```

### Streaming

``` python
stream = client.chat.completions.create(
    model="gpt-5.2",
    messages=[{"role": "user", "content": "Count to 5"}],
    stream=True,
)
for chunk in stream:
    delta = chunk.choices[0].delta.content or ""
    print(delta, end="", flush=True)
```

## Google Gemini

### Install

``` bash
pip install -q -U google-genai
```

Use the new `google-genai` package, **not** the older `google-generativeai`. Set `GEMINI_API_KEY`.

### Basic generation

``` python
from google import genai
from google.genai import types

client = genai.Client()

response = client.models.generate_content(
    model="gemini-3-flash-preview",
    contents="Explain attention in one sentence.",
    config=types.GenerateContentConfig(
        system_instruction="You are a concise ML teacher.",
        temperature=1.0,  # keep default for Gemini 3
    ),
)
print(response.text)
```

### Streaming

``` python
for chunk in client.models.generate_content_stream(
    model="gemini-3-flash-preview",
    contents="List 3 facts about penguins.",
):
    print(chunk.text, end="", flush=True)
```

### Thinking

``` python
response = client.models.generate_content(
    model="gemini-3-flash-preview",
    contents="Solve: 23 * 47",
    config=types.GenerateContentConfig(
        thinking_config=types.ThinkingConfig(thinking_budget=2048),
    ),
)
```

## Which API should I use?

- **Direct vendor SDKs** — when you need a provider-specific feature (Anthropic extended thinking with server tools, OpenAI Responses API reasoning items, Gemini 3 multimodal chaining).

- **LangChain `init_chat_model` + `create_agent`** — for everything else. Uniform interface, works with all three providers, and gives you streaming / persistence / human-in-the-loop for free.

See [LangGraph Foundations & State Management](../agents/langgraph-foundations.md) for the LangChain/LangGraph path.

## Practice

### 1. Hello World in three providers

Write a single Python file that calls `claude-sonnet-4-6`, `gpt-5.2`, and `gemini-3-flash-preview` with the same prompt (`"Name`` ``one`` ``thing`` ``parrots`` ``and`` ``dolphins`` ``have`` ``in`` ``common."`) and prints all three responses side by side. Each should live behind a function of the signature `call(prompt:`` ``str)`` ``->`` ``str`.

Use environment variables for API keys; never hardcode.

### 2. Tool use round-trip

Using the Anthropic SDK, define a `get_weather(city)` tool with `strict:`` ``true`. Implement the client-side loop:

1.  Send the user query.

2.  Parse `response.content` for a `tool_use` block.

3.  Execute the tool (return a stub `"sunny,`` ``72°F"`).

4.  Send the `tool_result` back to the same model.

5.  Print the final natural-language answer.

### 3. Streaming token counter

Stream a ~500-token response from any provider and count the tokens as they arrive (simple word count is fine). Print the count every 50 tokens to demonstrate you are receiving partial chunks rather than waiting for the whole response.

### 4. Extended thinking budget comparison

On Claude Sonnet 4.6, solve the same hard problem twice:

1.  With `thinking={"type":`` ``"adaptive"}`

2.  Without any thinking parameter

Compare:

- Latency (wall-clock)

- Quality (is the answer correct?)

- Token usage (`usage.output_tokens` and any thinking token counts)

Write a one-paragraph takeaway about when extended thinking is worth the cost.

### 5. Provider migration

Take a working Anthropic SDK script. Migrate it to use LangChain’s `init_chat_model("anthropic:claude-sonnet-4-6")` and confirm the output is equivalent. Then swap the model string to `"openai:gpt-5.2"` without changing any other code — confirm it still works. This demonstrates the value of the LangChain abstraction for multi-provider code.

## Review Questions

1.  **What is the install command for the official Anthropic Python SDK?**

    - A. `pip`` ``install`` ``claude-api`

    - B. `pip`` ``install`` ``anthropic-sdk`

    - C. `pip`` ``install`` ``anthropic`

    - D. `pip`` ``install`` ``claude-python`

2.  **Which OpenAI API is documented as the primary surface for new code as of 2026?**

    - A. Completions API (`client.completions.create`)

    - B. Chat Completions API (`client.chat.completions.create`)

    - C. Responses API (`client.responses.create`)

    - D. Assistants API

3.  **Which package should you install for Google Gemini in Python today?**

    - A. `google-generativeai` (the older package)

    - B. `google-genai` (the new SDK)

    - C. `gemini-python`

    - D. `gcloud-ai`

4.  **On Claude Opus 4.6 and Sonnet 4.6, which thinking mode is recommended?**

    - A. `{"type":`` ``"enabled",`` ``"budget_tokens":`` ``10000}`

    - B. `{"type":`` ``"adaptive"}`

    - C. `{"type":`` ``"disabled"}`

    - D. Thinking is not supported

5.  **What is the recommended temperature for Gemini 3 models?**

    - A. 0.0

    - B. 0.5

    - C. 1.0 (default)

    - D. 2.0

6.  **You need a 1M-token context window and top-tier reasoning. Which Claude model?**

    - A. `claude-haiku-4-5` (200k context)

    - B. `claude-opus-4-6` (1M context)

    - C. `claude-3-opus-20240229`

    - D. `claude-instant`

7.  **Which Anthropic server tool type enables the model to search the web?**

    - A. `web_search_20260209`

    - B. `google_search`

    - C. `web_tool`

    - D. `search_engine`

8.  **Which parameter guarantees an Anthropic tool call conforms to the declared input schema?**

    - A. `temperature:`` ``0`

    - B. `strict:`` ``true` on the tool definition

    - C. `mode:`` ``"strict"`

    - D. `json_mode:`` ``true`

9.  **You want provider-agnostic code that can swap between Claude, GPT, and Gemini without rewrites. What do you reach for?**

    - A. A custom HTTP wrapper

    - B. LangChain’s `init_chat_model` with a provider-prefix string

    - C. Three separate `if` branches per provider

    - D. `openai` package only

10. **Where should API keys come from in production code?**

    - A. Hardcoded at the top of the file

    - B. Environment variables (or a secret manager)

    - C. A public GitHub gist

    - D. The user’s prompt

**View Answer Key**

1.  **C** — `pip`` ``install`` ``anthropic`.

2.  **C** — The Responses API is the primary surface in current OpenAI docs; Chat Completions is still supported as legacy.

3.  **B** — `google-genai` is the new SDK; `google-generativeai` is the older package.

4.  **B** — Adaptive thinking is the recommended mode for Opus/Sonnet 4.6; `budget_tokens` is deprecated on those models.

5.  **C** — Google explicitly recommends keeping `temperature=1.0` default for Gemini 3.

6.  **B** — Claude Opus 4.6 has a 1M-token context and is the highest-intelligence broadly-available Claude.

7.  **A** — `web_search_20260209` (dated identifiers are how Anthropic versions server tools).

8.  **B** — `strict:`` ``true` on the tool enforces schema conformance.

9.  **B** — `init_chat_model("anthropic:claude-sonnet-4-6")` vs `"openai:gpt-5.2"` — same code, different string.

10. **B** — Environment variables or a secret manager. Never hardcode.
