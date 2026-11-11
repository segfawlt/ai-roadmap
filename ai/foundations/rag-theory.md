# Introduction to RAG and Theoretical Foundations

The explosion of Large Language Models (LLMs) like ChatGPT, Gemini, Claude, or Llama has reshaped the NLP field. However, despite their impressive generalization and reasoning capabilities, these models still face inherent limitations: restricted knowledge at the time of training, hallucinations when encountering questions outside their knowledge domain, and especially a lack of knowledge about private enterprise data.

To address this issue, Retrieval Augmented Generation (RAG) was created. RAG allows LLMs to access external data sources without the need for expensive fine-tuning or retraining. To introduce RAG, this article delves into its architecture and implementation, including:

1.  Deep analysis of RAG concepts, architecture, and basic pipeline.

2.  Introduction to the LangChain framework, a powerful tool for LLM applications.

3.  Building a QA System on academic PDF documents.

Figure 1: Illustration of LLM with (green line) and without RAG (red line).

``` mermaid
        graph LR
    Q[/"Khoá học AIO2025<br>có bao nhiêu module?"/]
    Q -->|without RAG| LLM[Qwen LLM]
    Q -->|with RAG| RAG[RAG]
    RAG -->|retrieves from| DB[(Dữ liệu AIVN)]
    RAG --> LLM
    LLM -->|correct| A1["✓ 12 module"]
    LLM -->|without RAG| A2["✗ 10 module"]
```

| Glossary | Glossary |
|----|----|
| Term | Description |
| Hallucination | The phenomenon where the model generates false, fabricated, or non-existent information but with a confident tone. |
| Knowledge Cutoff | The time limit of training data, making the model unaware of events occurring afterwards. |
| Fine-tuning | The process of further training a pre-trained model on a specialized dataset to update weights. |
| In-Context Learning | The ability of an LLM to learn and perform tasks based on context or examples provided in the prompt without parameter updates. |
| Vector Embeddings | Representation of data (text, image) as real-number vectors in n-dimensional space. |
| Semantic Search | Search based on meaning similarity rather than just keyword matching. |
| Chunking | The technique of splitting long text into short segments to optimize encoding and fit Context Window limits. |
| Context Window | The maximum number of tokens (text units) that an LLM can receive and process in a single prompt. |
| Grounding | The technique of ‘anchoring’ the model’s answer to provided real-world data to ensure authenticity. |

## Theoretical Foundations of RAG

### Origins

The concept of RAG was first officially proposed in the scientific paper **‘Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks’** by Patrick Lewis and colleagues at Facebook AI Research (FAIR) in 2020[^1].

``` mermaid
        graph LR
    Q[Question] --> QE[Query Encoder<br>DPR]
    QE --> R[Retriever]
    R -->|Top-K docs| G[Generator BART]
    KB[(Non-Parametric Memory<br>Dense Wikipedia Index)] --> R
    Q --> G
    G --> A[Answer]

    style KB fill:#fff3cd
    style G fill:#d4edda
    style QE fill:#cce5ff
```

*Figure 2: RAG architecture overview in Patrick Lewis’s original paper (2020).*

In this work, the authors defined RAG as a hybrid probabilistic model combining two memory types to overcome the drawbacks of traditional Pre-trained Seq2Seq models:

- **Parametric Memory**: Implicit knowledge stored in the weights of a sequence generation model (Pre-trained Seq2Seq Transformer). Specifically in the paper, the authors used the BART (Bidirectional and Auto-Regressive Transformers) model as the Generator.

- **Non-Parametric Memory**: Explicit external knowledge, specifically a dense vector index containing Wikipedia text segments. This component is accessed via a Neural Retriever based on the Dense Passage Retriever architecture.

The mechanism of the original RAG allows the Generator (BART) to use input combined with latent documents found by the Retriever to generate text. A key feature is that the entire architecture is fine-tuned end-to-end, enabling weight updates for both the Query Encoder and Generator to optimize the target task.

### The Shift to In-Context RAG

Although the term RAG remains, the implementation mindset has fundamentally changed with the development of LLMs:

- **Original RAG (2020)**: An approach based on fine-tuning. As mentioned above, the original model required simultaneous training of both the retriever and the text generation model so they could learn to coordinate. Model weights changed during this process.

- **Modern RAG (Current)**: An approach based on In-Context Learning. With the explosion of massive LLMs capable of understanding broad contexts, modern RAG typically refers to a **‘Retrieve and Prompt’** process.

In the modern model, we typically keep the LLM weights fixed and focus only on optimizing data retrieval, then feeding this data into the input (Prompt) for the model to process. This approach is flexible, low-cost, and easily applicable to private data without complex training procedures.

## The RAG Landscape in 2026 NEW

### Long-Context Models vs RAG

The arrival of models with 1M+ token context windows — such as Claude and Gemini — has made the “stuff it all in” strategy genuinely viable for small corpora under roughly 1 million tokens. If your entire knowledge base fits in a single prompt, you can skip retrieval entirely and let the model reason over everything at once.

That said, RAG remains essential in the vast majority of real-world deployments:

- **Large knowledge bases**: Most enterprise corpora far exceed any context window. A legal firm’s document repository, a software company’s internal wiki, or a hospital’s patient records cannot be squeezed into a single prompt.

- **Real-time and frequently updated data**: Model context is static once the prompt is sent. RAG can pull from live databases, APIs, or document stores refreshed by the minute.

- **Cost efficiency**: Retrieving the 5 most relevant chunks is dramatically cheaper than sending 500K tokens on every query. At scale, this difference is significant.

- **Citation and attribution**: RAG returns source metadata alongside content, making it straightforward to show users exactly which document or passage the answer came from — a hard requirement in regulated industries.

- **Access control**: A retrieval layer can enforce per-user or per-role document permissions before anything reaches the model. Stuffing everything into context makes this nearly impossible.

The emerging production pattern in 2026 is **hybrid**: a long-context LLM receives the top-k retrieved chunks alongside full relevant document sections, combining the precision of retrieval with the broad reasoning of a large context window.

### The RAG Maturity Model

RAG has evolved significantly since the original 2020 paper. The diagram below traces the four-generation arc from simple pipelines to fully agentic systems.

``` mermaid
        graph LR
    A["Naive RAG\n(2023)"] --> B["Advanced RAG\n(2024)"]
    B --> C["Modular RAG\n(2025)"]
    C --> D["Agentic RAG\n(2026)"]

    style A fill:#cce5ff
    style B fill:#d4edda
    style C fill:#fff3cd
    style D fill:#f8d7da
```

Each generation addresses the limitations of the previous one:

- **Naive RAG (2023)**: The baseline “retrieve then generate” pipeline. A query is embedded, the nearest chunks are fetched from a vector store, and the result is appended to the prompt. Simple and fast, but brittle — poorly phrased queries return irrelevant chunks, and there is no mechanism to catch or correct bad retrievals.

- **Advanced RAG (2024)**: Adds intelligence before and after retrieval. Query transformation (rewriting, HyDE, step-back prompting) improves what gets sent to the retriever. Re-ranking models (e.g., cross-encoders) re-score candidates after initial retrieval. Hybrid search combines dense vector similarity with sparse BM25 keyword matching to improve recall.

- **Modular RAG (2025)**: Decouples the pipeline into composable, swappable components. Any retriever (vector DB, graph DB, SQL), any reranker, and any generator can be mixed and matched. Routing logic directs queries to the most appropriate retrieval strategy. This architecture made RAG systems far easier to maintain and extend.

- **Agentic RAG (2026)**: The agent itself decides when to retrieve, what to retrieve, and whether the results are sufficient. It can issue multiple retrieval calls, reformulate queries after inspecting intermediate results, call external tools (calculators, APIs, code interpreters), and self-correct before producing a final answer. This closes the loop between reasoning and retrieval in a way that earlier generations could not.

### Fine-tuning vs RAG vs Prompt Engineering

Choosing the right approach — or combination — depends on the problem. The table below maps each technique to its ideal use case.

| Approach | Best When | Cost | Knowledge Freshness | Example |
|----|----|----|----|----|
| Prompt Engineering | Formatting, persona, simple tasks | Very low | Static | System prompt to enforce a formal writing tone |
| RAG | Dynamic knowledge, citations needed | Medium | Real-time | Enterprise internal Q&A bot over a live document store |
| Fine-tuning | Behavioral changes, consistent style | High | Frozen at fine-tune time | Brand-specific writing assistant trained on company copy |
| Combined (production) | Most real-world systems | Varies | Best of all | Fine-tuned model + RAG pipeline + carefully engineered system prompt |

A few practical notes on this decision:

- **Start with prompt engineering**. It costs nothing to iterate on a system prompt, and it often gets you 80% of the way there.

- **Add RAG when the knowledge is too large, too dynamic, or too sensitive** to bake into training data or a static prompt.

- **Fine-tune when you need to change how the model behaves**, not just what it knows. Teaching a model to follow a specific output schema, adopt a brand voice, or consistently apply domain-specific reasoning patterns are all behavioral changes that fine-tuning handles better than RAG.

- **Combine all three in production**. The most capable systems use a fine-tuned base model (for domain behavior), a RAG pipeline (for current knowledge), and a structured system prompt (for output format and safety guardrails).

------------------------------------------------------------------------

\[[1](#id1)\]

See details at: <https://arxiv.org/abs/2005.11401>

[^1]:
