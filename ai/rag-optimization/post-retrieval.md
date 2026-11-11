# Re-ranking

After performing Indexing and Retrieval steps, we obtain a list of Top-K potential documents. However, this list is often not optimal enough to feed directly into the LLM for two main reasons:

1.  **Limited Semantic Accuracy**: Embedding models prioritize optimizing retrieval speed on large amounts of data, thus forced to trade off the ability to understand complex semantic relationships between questions and texts.

2.  **Information Noise**: Found documents may contain many keywords matching the question but deviate in context or true intent, leading to providing inaccurate information to the LLM.

Re-ranking acts as a filter in the final step. We accept spending a little more time to “read carefully” a small set of candidates, e.g., 50 documents, and select the best documents, e.g., 5 documents, to send to the LLM.

## Cross-Encoder

In the Retrieval step, we used Embedding models to encode questions and texts into vectors. Architecturally, this method is called **Bi-Encoder**. To understand why we need to add a Cross-Encoder at this step, let’s compare their mechanisms.

**Bi-Encoder vs. Cross-Encoder**

``` mermaid
        graph TD
    subgraph "Bi-Encoder"
        BQ[Query] --> BE1[BERT Encoder]
        BD[Document] --> BE2[BERT Encoder]
        BE1 --> BQE[Query Embedding]
        BE2 --> BDE[Document Embedding]
        BQE & BDE --> BS[Similarity Score]
    end
    subgraph "Cross-Encoder"
        CI["[CLS] Query [SEP] Document [SEP]"] --> CE[BERT Encoder]
        CE --> RS[Relevance Score]
    end
```

*Figure 5: Comparing basic differences between two architectures.*

- **Bi-Encoder is used when searching**: Processes question and document **separately**. It creates a vector for the question, a vector for the document, then calculates the distance.

  - **Pros**: Fast speed, calculations can be pre-computed.

  - **Cons**: Loses detailed interaction information between each word in the question and each word in the document.

- **Cross-Encoder allows in Re-ranking**: The question and document are concatenated into a single text sequence as in Figure 5. This sequence is fed into the model for processing simultaneously. The model will “read” in parallel and consider the interaction between every word of the question with every word of the document through a full Self-Attention mechanism.

  - **Pros**: High accuracy, understands nuance, negation, and complex logic.

  - **Cons**: Very slow and resource-consuming, cannot be used to search across the entire database.

**Funnel Strategy**: Because Cross-Encoder is slow, we do not apply it to all data. The standard process in practice is:

1.  **Retrieve**: Use Bi-Encoder to quickly get Top 50 documents from millions of documents.

2.  **Re-rank**: Use Cross-Encoder to re-score these Top 50 documents.

3.  **Select**: Take the Top 5 documents with the highest scores after re-scoring to put into context for the LLM.

**Example of Cross-Encoder power**

``` mermaid
        graph TD
    Q["Query: 'What does Python not eat?'"]
    Q -->|Bi-Encoder| BE["Bi-Encoder results<br>(matches keywords: Python, eat, food)"]
    BE --> W1["Rank 1: 'How to feed Python pythons rats'<br>(wrong intent)"]
    BE --> W2["Rank 2: 'Food of python species'<br>(wrong intent)"]
    Q -->|Cross-Encoder| CE["Cross-Encoder re-ranking<br>(reads negation 'not eat' + biology context)"]
    CE --> C1["Rank 1: 'Python pythons can fast for months'<br>(correct intent)"]
```

**Query**: “What does Python not eat?”

- **Bi-Encoder**: May return documents containing keywords “Python”, “eat”, “food” such as: “How to feed Python pythons rats”, “Food of python species”. May search with wrong intent because it only catches keywords.

- **Cross-Encoder**: When reading both the question and document at the same time, it recognizes the negation structure “not eat” and biological context, thereby ranking high the document: “Python pythons can fast for months…”.

## Maximal marginal relevance (MMR)

Sometimes, finding the most similar documents is not the best thing.

Suppose a user asks: *“Biography of Steve Jobs”*. If based only on similarity, the system may return 5 nearly identical text paragraphs, all talking about him founding Apple in 1976. This wastes the LLM’s context window without providing new information.

MMR solves this problem by balancing two factors:

- **Relevance**: The document must be related to the question.

- **Diversity**: The document must be different from previously selected documents.

**Operating Principle of MMR**

1.  Select the document with the highest similarity to the question.

2.  Find the next document such that it is **both similar to the question** and **least similar** to the document selected in Step 1.

3.  Repeat until there are enough Top-K documents.

**Simplified Formula**:

\\ MMR = \arg \max\_{d_i \in D \setminus S} \[\lambda \cdot Sim_1(d_i, q) - (1-\lambda) \cdot \max\_{d_j \in S} Sim_2(d_i, d_j)\] \\

Where `\(\lambda\)` is the adjustment parameter, usually 0.5. If `\(\lambda\)` is smaller, the system prioritizes diversity more.

``` mermaid
        graph TD
    Q[Query] --> S1["Step 1: Select doc most similar to query → D1"]
    S1 --> S2["Step 2: Select next doc: similar to query AND least similar to D1 → D2"]
    S2 --> S3["Step 3: Repeat until Top-K docs selected"]
    S3 --> R["Result: diverse, relevant document set"]
```

### MMR Illustrative Example

**Query**: “Features of VinFast VF8 car.”

**Result without MMR**:

- Doc 1: VF8 has a powerful 402 horsepower electric motor.

- Doc 2: Maximum power of VF8 is 300kW (equivalent to 402 horsepower).

- Doc 3: VF8 engine gives impressive acceleration capabilities. → *Information is completely repeated regarding the engine.*

**Result with MMR**:

- Doc 1: VF8 has a powerful 402 horsepower electric motor. (Selected because most similar to question).

- Doc 2: ADAS driver assistance system on VF8 includes lane departure warning… (Selected because content differs from Doc 1).

- Doc 3: 10-year warranty policy for VF8 battery. (Selected because differs from Doc 1 and Doc 2). → *LLM has enough data to answer comprehensively about engine, safety, and after-sales.*

In summary, choosing a Re-ranking method depends on the goal: If you need extremely accurate answers for difficult questions, use **Cross-Encoder**. If you need general answers covering many aspects, use **MMR**.

## Additional Post-Retrieval Techniques NEW

### Contextual Compression

Instead of sending full retrieved documents to the LLM, use a smaller model (or the LLM itself) to extract only the relevant portions. This reduces token usage and focuses the LLM’s attention.

Two approaches:

1.  **Extractive compression**: Extract relevant sentences/passages from retrieved docs.

2.  **Abstractive compression**: Use a small LLM to summarize retrieved docs into concise context.

LangChain provides `ContextualCompressionRetriever` with extractors like `LLMChainExtractor`.

``` python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

compressor = LLMChainExtractor.from_llm(llm)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=retriever,
)
```

**When to use compression**

Extractive compression is faster and cheaper — prefer it when retrieved documents are long but well-structured (clear sentences, paragraphs). Abstractive compression is better when documents are dense or require synthesis across multiple passages.

### The Lost-in-the-Middle Problem

Research has shown that LLMs pay the most attention to information at the **beginning** and **end** of the context window, but often ignore information in the **middle** (Liu et al., 2023).

**Attention distribution across the context window**

``` mermaid
        graph LR
    A["[HIGH attention]\nBeginning of context"] --> B["[LOW attention]\nMiddle of context"]
    B --> C["[HIGH attention]\nEnd of context"]
```

*Figure: LLMs reliably attend to the edges of the context window but lose track of documents placed in the middle.*

Mitigation strategies:

- **U-shape reordering**: Place the most relevant documents at the start and end; less relevant documents go in the middle.

- **Fewer, better documents**: Retrieve Top-3 instead of Top-10 — less noise means less lost information.

- **Compression before stuffing**: Reduce each document to essential information, so the “middle” shrinks and every slot carries high signal.

### LLM-Based Re-ranking

Instead of (or in addition to) Cross-Encoder models, use an LLM as a re-ranker. The LLM reads the query and each candidate document, then scores or ranks them.

**Advantages**

- Can understand complex instructions (e.g., “rank by relevance to audit compliance”).

- Can explain its ranking decisions, making the pipeline auditable.

- Works with any LLM — no specialized model training required.

**Trade-offs to keep in mind**

- Much more expensive than a Cross-Encoder per document scored.

- Slower at scale due to sequential LLM calls.

- Best reserved for small candidate sets (Top-10 to Top-20) after an initial fast retrieval pass.
