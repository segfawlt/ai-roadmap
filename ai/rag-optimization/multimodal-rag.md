# Multimodal RAG NEW

Traditional RAG operates on text only. Multimodal RAG extends retrieval to images, PDFs with visual layouts, charts, tables, and mixed-media documents — making it essential for real-world enterprise knowledge bases where critical information lives in non-text form.

## Learning Objectives

- Understand why text-only RAG fails on visual documents

- Know the major approaches to multimodal retrieval

- Understand ColPali and vision-language retrieval models

- Choose the right approach for a given use case

- Design a multimodal RAG pipeline end-to-end

------------------------------------------------------------------------

## 1. The Problem with Text-Only RAG

Many enterprise documents contain critical information in visual formats:

- **Org charts** — reporting hierarchies, headcount

- **Financial tables** — P&L rows, footnotes anchored to layout position

- **Architecture diagrams** — component relationships that text alone cannot encode

- **Scanned PDFs** — no machine-readable text layer at all

Text-only RAG fails here in three ways:

1.  **Silent data loss** — images, charts, and diagrams are simply skipped

2.  **OCR degradation** — tables become garbled delimited strings; layout context is lost

3.  **No visual reasoning** — bar charts, heatmaps, and annotated screenshots require visual understanding that text embeddings cannot provide

The consequence is retrieval that looks correct (cosine similarity above threshold) but returns answers missing the most important part of the source document.

------------------------------------------------------------------------

## 2. Approaches to Multimodal RAG

### Approach 1: OCR + Text RAG (Legacy)

Extract text via OCR, then run standard text RAG on the extracted content.

**How it works:**

``` mermaid
        graph LR
    IN["PDF / Image"] --> OCR["OCR Engine<br/>(Tesseract / Textract)"]
    OCR --> TXT["Plain Text"]
    TXT --> EMB["Embed"]
    EMB --> RET["Retrieve"]
```

**Verdict:** Simple and cheap, but lossy. Tables become garbled comma-separated text, diagrams are ignored, and layout-dependent meaning (column headers, cell relationships) is destroyed.

------------------------------------------------------------------------

### Approach 2: Captioning + Text RAG

Use a vision LLM (GPT-4o, Claude with vision, Gemini) to generate rich text descriptions of each page or image, then embed those descriptions as if they were text chunks.

**How it works:**

``` mermaid
        graph LR
    IMG["Page Image"] --> VLM["Vision LLM"]
    VLM --> CAP["Caption:<br/>'This chart shows Q3 revenue...'"]
    CAP --> EMB["Embed"]
    EMB --> RET["Retrieve"]
```

**Verdict:** Higher quality than OCR for diagrams and charts. Still lossy — the caption is the model’s interpretation, not the ground truth. Expensive at scale: each page costs a full multimodal LLM call at ingestion time.

------------------------------------------------------------------------

### Approach 3: Native Multimodal Embeddings

Models such as `gemini-embedding-002` project text and images into a shared vector space. You can embed a document image and a text query and compare them directly.

**How it works:**

``` mermaid
        graph LR
    IMG["Page Image"] --> SPACE["Shared Embedding<br/>Space"]
    TXT["Text Query"] --> SPACE
    SPACE --> COS["Cosine Similarity"]
    COS --> RES["Ranked Results"]
```

**Verdict:** Architecturally elegant and retrieval-efficient. Quality depends heavily on the embedding model’s training distribution. Best suited for cross-modal search (e.g., find the diagram that matches this text description).

------------------------------------------------------------------------

### Approach 4: Vision-Language Retrieval — ColPali / ColQwen

The most capable development in multimodal retrieval: apply ColBERT-style late interaction directly to document page images using a vision-language model (VLM). No text extraction of any kind is required.

**How it works:**

``` mermaid
        flowchart LR
    A["Document Pages\n(images)"] --> B["Vision-Language Model\n(PaliGemma / Qwen2-VL)"]
    B --> C["Per-token patch embeddings\n(H × W token grid)"]
    Q["Query\n(text)"] --> D["Text Encoder"]
    D --> E["Query token embeddings"]
    C --> F["Late Interaction\nMaxSim score"]
    E --> F
    F --> G["Ranked pages"]
```

The VLM converts each document page into a grid of patch-level token embeddings. At query time, the query is encoded into token embeddings, and a MaxSim operation scores each patch against each query token. This retains fine-grained spatial information that a single pooled vector would collapse.

**Key models:**

| Model | Base VLM | Notes |
|----|----|----|
| ColPali | PaliGemma | Pioneered the approach; ViDoRe benchmark baseline |
| ColQwen2 | Qwen2-VL | Improved layout understanding; multilingual |
| NVIDIA Nemotron ColEmbed V2 | Various | 3B / 4B / 8B variants; SOTA on ViDoRe v2 |

**Advantages:**

- Zero OCR, zero captioning, zero text extraction

- The model “reads” the page visually, preserving spatial layout

- Tables, charts, and annotated diagrams are retrieved correctly

- Works on scanned documents and handwritten notes

------------------------------------------------------------------------

## 3. Choosing an Approach

| Approach | Retrieval Quality | Ingestion Cost | Query Latency | Best For |
|----|----|----|----|----|
| OCR + Text RAG | Low | Low | Fast | Simple, text-heavy documents |
| Captioning + Text RAG | Medium | High (per page LLM call) | Fast | Mixed media, small corpora |
| Multimodal Embeddings | High | Medium | Fast | Cross-modal search |
| ColPali / ColQwen | Highest | Medium | Medium | Complex visual documents |

**Decision guide:**

- Fewer than 1,000 pages, budget available → Captioning + Text RAG

- Large corpus, mostly text with occasional images → Multimodal Embeddings

- Audit reports, financial filings, technical manuals → ColPali / ColQwen

- Regulatory requirement to preserve exact source → ColPali + page-level storage

------------------------------------------------------------------------

## 4. Multimodal RAG Pipeline Design

A production multimodal RAG pipeline using ColQwen looks like this:

``` mermaid
        flowchart TD
    subgraph Ingestion
        A["Source documents\n(PDF, DOCX, images)"] --> B["Page renderer\n(pdf2image / Pillow)"]
        B --> C["ColQwen encoder"]
        C --> D["Per-token patch vectors"]
        D --> E["Vector store\n(Vespa / Qdrant multi-vector)"]
    end

    subgraph Retrieval
        Q["User query"] --> F["Text encoder\n(same VLM backbone)"]
        F --> G["MaxSim retrieval\nagainst patch vectors"]
        G --> H["Top-K page images"]
    end

    subgraph Generation
        H --> I["Vision LLM\n(GPT-4o / Claude / Gemini)"]
        Q --> I
        I --> J["Answer with\npage citations"]
    end

    E --> G
```

**Key implementation notes:**

1.  **Page rendering** — Convert PDF pages to high-resolution images (300 DPI recommended). Lower resolution degrades ColPali accuracy on small text and dense tables.

2.  **Vector store choice** — ColBERT-style retrieval requires storing multiple vectors per document (one per patch token). Use Vespa, Qdrant, or Milvus with multi-vector support. Standard single-vector stores like Pinecone require approximations.

3.  **Hybrid approach for generation** — Retrieve with ColQwen (visual), then pass the retrieved page images to a vision LLM for answer generation. This separates retrieval quality from generation cost.

4.  **Re-ranking** — Add a lightweight re-ranker after MaxSim retrieval to handle ties and improve precision before sending pages to the generation LLM.

------------------------------------------------------------------------

## 5. Practical Considerations

### Token and Storage Cost

| Factor | Impact |
|----|----|
| Image tokens in LLM | 1,000–2,000 tokens per page (varies by model) |
| ColBERT vector storage | ~100–500 vectors per page vs. 1 for standard RAG |
| Batch ingestion | Use async pipelines; 1,000-page corpus takes minutes on GPU |

### Evaluation

Standard text-based RAG metrics (RAGAS faithfulness, answer relevance) do not capture visual understanding failures. Use:

- **ViDoRe benchmark** — visual document retrieval; ColPali / ColQwen leaderboard

- **Human spot-check** — sample retrieved pages, verify correct chart/table is returned

- **Visual grounding accuracy** — verify the answer references the correct page region

### Cost Optimization

- **Two-stage pipeline:** Run ColQwen retrieval (cheap), then pass only top-3 pages to the vision LLM (expensive). Avoid passing all retrieved pages to the LLM.

- **Thumbnail retrieval + full-res generation:** Retrieve on 150 DPI thumbnails, then fetch the 300 DPI original only for the generation step.

- **Caching:** Cache ColQwen page embeddings; re-embed only on document update.

------------------------------------------------------------------------

## 6. Summary

Multimodal RAG is not a single technique but a spectrum of approaches with different quality-cost-latency tradeoffs. For FPT audit work — where source evidence lives in financial statements, scanned filings, and structured reports — native visual retrieval with ColPali or ColQwen provides the most faithful grounding.

The key insight: retrieval quality sets a ceiling on generation quality. A vision LLM cannot generate a correct answer if the retrieval step failed to surface the right page. Invest in retrieval first.

**Next steps:** See [Building RAG Agent using LangChain](../llmops/building-rag-agents.md) for how to wrap a multimodal retrieval pipeline inside an agentic loop with tool use and citations.
