# Observability: LangFuse & LangSmith

This page explains how to instrument LLM applications for production using LangFuse and LangSmith, covering trace collection, cost tracking, latency debugging, and dataset-based evaluation so teams can maintain quality and control spending at scale.

## Learning Objectives

- Understand the critical role of Observability in LLM applications.

- Master the setup and integration of **LangFuse** (Open Source) and **LangSmith** (LangChain Native).

- Implement efficient tracing, cost tracking, and dataset evaluation.

## 1. The Challenge of LLM Observability

Building a demo is easy; running an LLM app in production is hard. Unlike traditional software where you can step through code reliably, LLM applications involve **probabilistic components** (the AI models) that act as “black boxes”.

### Why do we need Observability tools?

1.  **Black Box execution**: You give an input, you get an output. What happened in between? Did the RAG retriever find the right documents? Did the LLM ignore the system prompt?

2.  **Cost Management**: LLM API calls are expensive. A single runaway loop can cost hundreds of dollars. You need real-time cost tracking per user/feature.

3.  **Latency Debugging**: If a response takes 10 seconds, is it the Vector DB search or the OpenAI generation that is slow?

4.  **Quality Assurance**: How do you know if the new prompt version is actually better? You need to track usage and user feedback (thumbs up/down).

------------------------------------------------------------------------

## 2. LangFuse: Open Source Observability

[LangFuse](https://langfuse.com/) is a popular open-source tool that focuses on engineering observability. It is highly valued for its ability to be **self-hosted**, ensuring data privacy.

### Key Features

- **Tracing**: Visualizing the complex execution flow of chains and agents.

- **Prompt Management**: Version controlling your prompts (CMS for prompts).

- **Scores**: Attaching quality scores (manual or automated) to traces.

- **Cost Tracking**: Automatic calculation of token usage and costs.

### Installation & Setup

You can use LangFuse Cloud or host it yourself using Docker.

**1. Install SDK**

``` bash
pip install langfuse
```

or

``` bash
uv add 'langfuse'
```

**2. Configure Environment**

``` bash
os.environ["LANGFUSE_SECRET_KEY"] = "sk-..."
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-..."
os.environ["LANGFUSE_HOST"] = "https://cloud.langfuse.com" # or your local host
```

### Integration with LangChain

LangFuse provides a `CallbackHandler` that automatically instruments your LangChain runnables and agents.

``` python
from langfuse.callback import CallbackHandler
from langchain.chat_models import init_chat_model
from langchain_core.prompts import ChatPromptTemplate

# Initialize Handler
langfuse_handler = CallbackHandler()

# Compose a simple runnable with LCEL (LLMChain is deprecated in LangChain 1.x)
llm = init_chat_model("claude-sonnet-4-6")
prompt = ChatPromptTemplate.from_template("What is the capital of {country}?")
chain = prompt | llm

# Run with the callback — the trace will appear in your LangFuse dashboard
result = chain.invoke(
    {"country": "Vietnam"},
    config={"callbacks": [langfuse_handler]},
)
```

### Prompt Management (Advanced)

Instead of hardcoding prompts in Git, manage them in LangFuse to allow non-engineers to tweak them.

``` python
from langfuse import Langfuse
langfuse = Langfuse()

# Fetch production prompt
prompt_obj = langfuse.get_prompt("customer-support", version="production")

# Use in code
langchain_prompt = PromptTemplate.from_template(prompt_obj.get_langchain_prompt())
```

------------------------------------------------------------------------

## 3. LangSmith: The LangChain Native Solution

[LangSmith](https://smith.langchain.com/) is built by the LangChain team. While paid/commercial, it offers the deepest integration and powerful “Debug” features that allow you to replay and modify steps in a chain.

### Key Features

- **Deep Tracing**: Shows every single step, including retries and internal state.

- **Playground**: “Edit and Re-run”. You can take a failed production trace, open it in the playground, change the prompt, and see if it fixes the issue.

- **Datasets & Testing**: First-class support for creating datasets from production traffic and running evaluations.

### Setup (Auto-Tracing)

LangSmith is “magic” for LangChain users. You often don’t need code changes, just environment variables.

``` bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY="lsv2_..."
export LANGSMITH_PROJECT="my-production-app"
```

> **Note**: The older `LANGCHAIN_TRACING_V2` / `LANGCHAIN_API_KEY` / `LANGCHAIN_PROJECT` env var names still work as aliases, but `LANGSMITH_*` is the canonical naming in current docs.

### Code Example: RAG with LangSmith

Once the environment variables are set, *everything* is traced — `create_agent` runs, LangGraph state transitions, chat model calls, and vector store lookups all show up automatically.

``` python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

# This execution is automatically logged to LangSmith
vectorstore = FAISS.from_texts(
    ["LangSmith is great for debugging."],
    embedding=OpenAIEmbeddings(model="text-embedding-3-small"),
)
retriever = vectorstore.as_retriever()
docs = retriever.invoke("What is LangSmith good for?")
```

------------------------------------------------------------------------

## 4. Comparison: LangFuse vs. LangSmith

| Feature | LangFuse | LangSmith |
|----|----|----|
| **Open Source** | Yes (MIT License) | No (Proprietary) |
| **Self-Hosting** | Easy (Docker Compose) | Enterprise Only (Complex) |
| **Pricing** | Generous Free Tier / Free Self-host | Paid (per trace/token) |
| **Integration** | SDKs, LangChain Callbacks | Native (Environment Variables) |
| **Debugging** | Good (Trace visualization) | Excellent (Edit & Re-run Playground) |
| **Prompt Management** | Excellent (CMS style) | Good (Hub integration) |

### Recommendation

- **Choose LangFuse** if: You need **Data Privacy** (Self-hosted), are budget-conscious, or want a platform-agnostic solution (works well with non-LangChain code too).

- **Choose LangSmith** if: You go “All-in” on **LangChain/LangGraph**, need the advanced **Playground** debugging features, and don’t mind a SaaS dependency.

------------------------------------------------------------------------

## 5. Production Best Practices

### 1. Sampling

In production, you might process millions of requests. Tracing every single one is expensive and noisy.

- **Dev/Staging**: Trace 100%.

- **Production**: Trace 1-5% of traffic, or specific “High Importance” traces (errors, negative feedback).

### 2. PII / Data Privacy

Never log sensitive data (Credit Cards, PII) to a cloud observability tool.

- **LangFuse**: Run the PII Masking function before sending data.

- **LangSmith**: Use their enterprise features for PII redaction or self-host if compliant.

### 3. Alerts

Don’t just stare at dashboards. Set up alerts for:

- **Error Rate Spike**: If errors \> 10% in 5 minutes.

- **Latency Spike**: If P99 latency \> 10 seconds.

- **Cost Anomaly**: If daily spend \> \$50.

------------------------------------------------------------------------

## 6. Practice Exercise

Task: Trace your FPT Support Chatbot using LangSmith

------------------------------------------------------------------------

## The Observability Ecosystem in 2026 NEW

### Beyond LangFuse and LangSmith

| Tool | Type | Key Strength | Free Tier | Status (2026) |
|----|----|----|----|----|
| LangFuse | Open Source | Self-hostable, GDPR/HIPAA | 50k events/mo | Active, acquired Jan 2026 |
| LangSmith | SaaS | Best agent trace visualization | 5k traces/mo | Active, renamed from LangGraph Platform |
| Helicone | Proxy-based | 1-line setup, cost dashboards | 100k req/mo | Maintenance mode (acquired by Mintlify) |
| Arize Phoenix | Open Source | OpenTelemetry, embedding viz | Unlimited (self-host) | Active |
| Braintrust | SaaS | A/B testing prompts, experiment mgmt | Free tier | Active |

**Practical tip:** Many production teams run two tools — Helicone/LangFuse for cost tracking + LangSmith for development debugging.

------------------------------------------------------------------------

### Cost Optimization Strategies

This is a critical LLMOps concern. A combined optimization stack can reduce costs 60-80%:

1.  **Prompt Caching** (highest ROI)

    - Anthropic: 25% extra for cache writes, 10% for reads (90% discount on reads)

    - OpenAI: Automatic prefix caching

    - Requires static content before dynamic content in the prompt

    - Typical savings: 45-80%

2.  **Semantic Caching**

    - Cache prior query/response pairs in a vector store

    - On new query, check if a similar query was already answered

    - 61-69% hit rates on FAQ-style workloads

    - Not suitable for creative/personalized/time-sensitive tasks

3.  **Model Routing**

    - Route 70-80% of easy queries to cheap models (e.g., Haiku at \$0.25/M input)

    - Escalate complex queries to expensive models (e.g., Opus at \$15/M input)

    - Research shows 87% cost reduction while maintaining quality

4.  **Batch Processing**

    - 50% cost discount for async, non-real-time workloads

    - Both Anthropic and OpenAI offer batch APIs

``` mermaid
        graph LR
    Q["Query"] --> SC["Semantic Cache"]
    SC -->|"Cache Hit"| R["Cached Response"]
    SC -->|"Cache Miss"| PC["Prompt Cache<br/>(prefix match)"]
    PC --> MR["Model Router"]
    MR -->|"Simple"| CHEAP["Haiku / GPT-4o-mini"]
    MR -->|"Complex"| EXPENSIVE["Opus / GPT-4o"]
```

------------------------------------------------------------------------

## Summary

Observability is the “X-Ray” for your LLM application. Without it, you are flying blind. Start by integrating **LangSmith** for deep debugging during development, and consider **LangFuse** for production deployment if cost and privacy are concerns.
