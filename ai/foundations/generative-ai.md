# Introduction to AI & Generative AI

Artificial Intelligence (AI) has transitioned from a futuristic concept to an integral part of our daily lives. From the recommendation algorithms on Netflix to the voice assistants in our phones, AI is everywhere. However, the most significant shift in recent years has been the emergence of **Generative AI**.

## What is Artificial Intelligence?

At its core, AI is a branch of computer science that aims to create systems capable of performing tasks that typically require human intelligence. This includes:

- **Reasoning**: Making decisions based on data.

- **Problem-solving**: Finding solutions to complex challenges.

- **Learning**: Improving performance over time through experience.

``` mermaid
        graph TB
    AI["Artificial Intelligence"]
    AI --> ML["Machine Learning"]
    AI --> RULE["Rule-based Systems"]
    ML --> DL["Deep Learning"]
    ML --> TRAD["Traditional ML<br/>Decision Trees, SVM, etc."]
    DL --> DISC["Discriminative Models<br/>Classification, Prediction"]
    DL --> GEN["Generative Models<br/>Content Creation"]
    GEN --> LLM["Large Language Models"]
    GEN --> DIFF["Diffusion Models<br/>Image, Video"]
    GEN --> MM["Multimodal Models<br/>Text + Image + Audio"]
```

## The Rise of Generative AI

While traditional AI focused on analyzing data and making predictions (Discriminative AI), **Generative AI** goes a step further by creating *new* content. This content can be text, images, code, or even music.

| Aspect  | Discriminative AI              | Generative AI            |
|---------|--------------------------------|--------------------------|
| Goal    | Classify or predict from input | Create new content       |
| Example | Spam filter, image classifier  | ChatGPT, DALL-E, Suno    |
| Learns  | Decision boundaries            | Data distribution        |
| Output  | Label or score                 | Text, image, code, audio |

The “engine” behind Generative AI often consists of **Large Language Models (LLMs)**. These models are trained on massive datasets to understand and generate human-like language.

## How LLMs Work: The Transformer Architecture NEW

Modern LLMs are built on the **Transformer** architecture, introduced in the landmark paper “Attention Is All You Need” (Vaswani et al., 2017). The core innovation is the **self-attention mechanism**, which allows the model to weigh the importance of different words relative to each other.

``` mermaid
        graph LR
    IN["Input Tokens"] --> EMB["Token + Position<br/>Embeddings"]
    EMB --> ATT["Self-Attention<br/>(which words matter?)"]
    ATT --> FFN["Feed-Forward<br/>Network"]
    FFN --> NEXT["Next Token<br/>Prediction"]

    style ATT fill:#cce5ff
```

Key concepts:

- **Tokenization**: Text is split into sub-word units called tokens. For example, “understanding” might become \[“under”, “standing”\].

- **Self-Attention**: Each token “attends to” every other token in the context, learning relationships like “the word ‘it’ refers to ‘the cat’.”

- **Next-Token Prediction**: LLMs are trained to predict the next token given all previous tokens. This simple objective, applied at massive scale, produces emergent capabilities like reasoning, translation, and coding.

- **Context Window**: The maximum number of tokens the model can process in a single call. In 2026, this ranges from 8K tokens (small models) to over 1M tokens (Claude Opus, Gemini 2.5).

## The LLM Landscape in 2026 NEW

The field has evolved rapidly. Key model families include:

| Provider | Models | Notable Capabilities |
|----|----|----|
| Anthropic | Claude 4.6 (Opus, Sonnet, Haiku) | 1M context, extended thinking, tool use, MCP |
| OpenAI | GPT-4o, o3 | Multimodal, reasoning chains |
| Google | Gemini 2.5 Pro/Flash | 1M context, multimodal (text+image+video+audio) |
| Meta | Llama 4 | Open-weight, strong multilingual |
| Mistral | Mistral Large, Codestral | Code-specialized, open-weight options |

### Key Capabilities of LLMs

1.  **Text Generation**: Writing essays, emails, reports, or stories.

2.  **Summarization**: Condensing long articles into key points.

3.  **Translation**: Converting text between different languages.

4.  **Coding Assistance**: Writing, debugging, and explaining code — the foundation of tools like Claude Code and Cursor.

5.  **Reasoning**: Solving multi-step math, logic, and planning problems (especially reasoning models like o3 and Claude extended thinking).

6.  **Multimodal Understanding**: Analyzing images, PDFs, charts, and even video alongside text.

7.  **Tool Use**: Calling external APIs, databases, and services to take actions in the real world.

### Reasoning Models NEW

A major development in 2025-2026 is the emergence of **reasoning models** that explicitly “think” before answering:

- **Chain-of-Thought (CoT)**: Models generate step-by-step reasoning, improving accuracy on complex problems.

- **Extended Thinking**: Claude’s approach where the model uses a dedicated thinking budget (up to 128K tokens) for internal reasoning before producing a response.

- **OpenAI o-series (o1, o3)**: Models trained specifically for multi-step reasoning with hidden chains of thought.

These models excel at math, coding, scientific analysis, and planning tasks where step-by-step logic is critical.

## Limitations of LLMs

Despite their power, LLMs have inherent “blind spots”:

- **Knowledge Cutoff**: They only know what was in their training data. Events after the cutoff date are unknown.

- **Hallucinations**: They can confidently state facts that are entirely false — the model generates plausible-sounding but fabricated information.

- **Lack of Private Data**: They don’t have access to your internal company documents, proprietary databases, or real-time information.

- **Context Window Limits**: Even 1M-token models cannot process an entire enterprise knowledge base in a single call.

- **No Persistent Memory**: By default, each conversation starts fresh — the model does not remember prior sessions.

## Bridging the Gap with RAG

To overcome these limitations without the massive cost of retraining a model, we use a technique called **Retrieval-Augmented Generation (RAG)**.

RAG allows the model to “look up” information from external sources before generating an answer, ensuring that its responses are grounded in factual, up-to-date, or private data.

``` mermaid
        graph LR
    Q["User Question"] --> R["Retriever"]
    KB[("Knowledge Base<br/>Documents, DBs")] --> R
    R --> CTX["Retrieved Context"]
    CTX --> LLM["LLM"]
    Q --> LLM
    LLM --> A["Grounded Answer<br/>with citations"]
```

### When to Use RAG vs Other Approaches NEW

| Approach | Best For | Cost | Knowledge Freshness |
|----|----|----|----|
| **Prompt Engineering** | Formatting, persona, simple tasks | Very low | Static (training data) |
| **RAG** | Dynamic knowledge, citations, private data | Medium | Real-time (updated externally) |
| **Fine-tuning** | Behavioral changes, specialized reasoning | High | Frozen at fine-tune time |
| **Combined** | Production systems | Varies | Best of all worlds |

Most production AI systems in 2026 layer all three: a fine-tuned model (for tone and format) + RAG (for live knowledge) + prompt engineering (for task framing and guardrails).

In the next section, we will dive deeper into how RAG works and why it has become the standard for building production-grade AI applications.
