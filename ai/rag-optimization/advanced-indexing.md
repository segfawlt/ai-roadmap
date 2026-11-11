# Advanced Indexing

In the previous article, we approached the Indexing phase in the simplest way:

1.  **Fixed-size Chunking**: Text is cut mechanically every 500 or 1000 characters, regardless of whether a sentence or idea is being expressed incompletely.

2.  **Flat Indexing**: All vectors are stored in a long list. When searching, the system has to compare the question with every single vector in that list, also known as Brute-force.

This approach works well with a few hundred documents. However, when the system scales up many times over, two serious problems arise:

- **Loss of Semantics**: Mechanical chunking accidentally breaks the flow of the text. A complete idea is split into two meaningless chunks, making the LLM unable to understand the context.

- **High Latency**: Sequentially scanning through millions of vectors is too slow to meet real-time requirements.

This section will introduce two techniques to solve the above problems: **Semantic Chunking** to cut text more intelligently and **HNSW** to search faster.

## Semantic Chunking

Since the quality of input data determines the quality of output, our goal is to split the text into segments small enough for precise searching, yet large enough to fully encompass an idea.

Instead of cutting by a fixed number of characters, the Semantic Chunking method understands the content to decide split points.

### Core Idea

Suppose we are reading an article.

- When consecutive sentences discuss the same topic, their vectors will be close to each other in the representation space.

- When sentences or content shift to a new topic, vectors will abruptly change direction, creating a large distance.

- → Semantic Chunking detects this change to perform a break exactly at the intersection of two topics.

### Implementation Process

1.  **Sentence Splitting**: Split the text into complete sentences based on punctuation.

2.  **Similarity Calculation**: Calculate similarity, for example cosine similarity, between the current sentence and the next one.

3.  **Thresholding**:

    - If similarity is high: Two sentences are on the same topic → Merge into one chunk.

    - If similarity drops significantly below the threshold: The topic has changed → Break the chunk here and start a new one.

### Comparison: Recursive vs. Semantic Chunking

To better visualize the impact of chunking strategy on data quality, let’s consider a specific example below.

**Input Data**: A text passage containing two distinct streams of information:

- First part: Introduction to hardware (NVIDIA H100 GPU).

- Second part: Introduction to language model (Llama-3).

``` mermaid
        graph TD
    subgraph "Recursive Chunking (by punctuation)"
        T1["Full text: NVIDIA H100 intro + Llama-3 intro"]
        T1 --> RC1["Chunk 1: NVIDIA H100 ..."]
        T1 --> RC2["Chunk 2: helps reduce training time<br>(subject missing!)"]
        T1 --> RC3["Chunk 3: ... Hopper architecture"]
        T1 --> RC4["Chunk 4: Llama-3 intro ..."]
    end
    subgraph "Semantic Chunking (by meaning)"
        T2["Full text: NVIDIA H100 intro + Llama-3 intro"]
        T2 --> SC1["Chunk 1: All info about NVIDIA H100<br>(complete idea)"]
        T2 --> SC2["Chunk 2: All info about Llama-3<br>(complete idea)"]
    end
```

*Figure 2: Comparison of fragmentation results: Recursive shreds information (left) while Semantic preserves semantics (right).*

### Analyzing Results

- **Recursive Chunking (Left)**: Due to breaking by punctuation, the text is split into 4 small chunks.

  > **Consequence**: Information is broken. For example, Chunk 2 contains information ‘helps reduce training time’, but the subject ‘NVIDIA H100’ is in Chunk 1. If the system only finds Chunk 2, the LLM will not be able to know what object is being referred to, leading to inaccurate answers.

- **Semantic Chunking (Right)**: The algorithm detects the semantic break point and cuts at the right time.

  > **Consequence**: We obtain 2 complete chunks. The first chunk contains all information about H100, the second chunk contains all information about Llama-3. Data integrity is ensured.

From the visual example above, we can compare the pros and cons of these two methods in the table below:

<table class="table">
<colgroup>
<col style="width: 33%" />
<col style="width: 33%" />
<col style="width: 33%" />
</colgroup>
<thead>
<tr class="row-odd">
<th class="head text-left"><p>Criteria</p></th>
<th class="head text-left"><p>Recursive Chunking</p></th>
<th class="head text-left"><p>Semantic Chunking</p></th>
</tr>
</thead>
<tbody>
<tr class="row-even">
<td class="text-left"><p><strong>Principle</strong></p></td>
<td class="text-left"><p>Cuts based on punctuation (newline, period) and a fixed number of characters.</p></td>
<td class="text-left"><p>Cuts based on changes in content meaning between sentences.</p></td>
</tr>
<tr class="row-odd">
<td class="text-left"><p><strong>Pros</strong></p></td>
<td class="text-left"><p>- Extremely fast speed, low cost.<br />
- Preserves original text structure.</p></td>
<td class="text-left"><p>- Preserves ideas fully, strictly follows the flow of text.<br />
- Increases accuracy when searching.</p></td>
</tr>
<tr class="row-even">
<td class="text-left"><p><strong>Cons</strong></p></td>
<td class="text-left"><p>- Easily cuts through important ideas.<br />
- Context can be noisy because it contains 2 halves of 2 different ideas.</p></td>
<td class="text-left"><p>- Consumes computational resources due to running a model to compare each sentence.</p></td>
</tr>
<tr class="row-odd">
<td class="text-left"><p><strong>Suitable for</strong></p></td>
<td class="text-left"><p>Documents with clear structure, e.g., contracts, laws, textbooks…</p></td>
<td class="text-left"><p>Narrative documents, less structured like meeting minutes, video transcripts…</p></td>
</tr>
</tbody>
</table>

## Optimizing Vector Database: HNSW Index

When working with large-scale vector datasets, brute-force searching by calculating distance to every data point becomes impossible in terms of time. To solve this problem, the standard solution is to use approximate nearest neighbor (ANN) search algorithms.

Currently, hierarchical navigable small world (HNSW) is considered one of the effective algorithms, balancing well between retrieval speed and result accuracy.

### Mechanism of Operation

HNSW organizes data in the form of a multi-layered graph structure. Instead of scanning all elements, the algorithm uses a hierarchical structure applied to the graph:

- **Layer 0**: Contains all data points, and the most detailed links between them. This is the layer containing the most complete information to find the exact target vector.

- **Higher layers**: Are reduced versions of layer 0. Nodes here are sparser and links are longer, serving as shortcuts helping the algorithm move quickly through the large data space.

``` mermaid
        graph TD
    subgraph "Layer 2 (sparse, long-range links)"
        L2A(( )) --- L2B(( ))
    end
    subgraph "Layer 1 (medium density)"
        L1A(( )) --- L1B(( ))
        L1B --- L1C(( ))
    end
    subgraph "Layer 0 (all nodes, dense links)"
        L0A(( )) --- L0B(( ))
        L0B --- L0C(( ))
        L0C --- L0D["● target"]
        L0A --- L0C
    end
    EP["Entry Point"] --> L2B
    L2B -->|"navigate down"| L1B
    L1B -->|"refine"| L0C
```

*Figure 3: Hierarchical graph structure of HNSW. Search starts from sparse links at the top layer and gradually refines accuracy at the bottom layer.*

### Important Configuration Parameters

When deploying HNSW on popular vector databases like ChromaDB, Milvus or Qdrant, system performance depends entirely on tuning three core parameters. Understanding the impact of each parameter will help us optimize the balance between hardware resources and search quality.

1.  **Parameter `M` (Max Links per Node)**: specifies the maximum number of links a node can create with other neighbor nodes in the graph.

    - **Meaning**: Decides the connection density of the graph. The larger the `M` value, the denser the network, and data points have more paths to reach each other.

    - **Impact**: A high `M` value helps the algorithm not get stuck in local extrema, thereby increasing search result accuracy. However, RAM consumption will increase significantly to store these links and the time to add new data will also be longer.

2.  **Parameter `ef_construction` (Construction Search Depth)**: controls the size of the list of potential candidates considered by the algorithm during index construction.

    - **Meaning**: When adding a new data point to the graph, the algorithm needs to find which existing nodes are the best neighbors to create links. The larger `ef_construction`, the more rigorous the algorithm, it will scan through more candidates to ensure finding the most optimal links.

    - **Impact**: High value helps create a high-quality graph structure, ensuring later search takes place smoothly and accurately. In return, the data loading process will be slower due to performing more calculations for each new data point.

    > **Search Process**:
    >
    > 1.  The algorithm starts from an entry point at the highest layer.
    >
    > 2.  At each layer, it moves to the neighbor node closest to the query vector until it reaches a local extremum at that layer.
    >
    > 3.  The found node will become the entry point for the next layer below.
    >
    > 4.  The process repeats until reaching Layer 0, where the algorithm performs exact local search to return the final result.

3.  **Parameter `ef_search` (Runtime Search Depth)**: Similar to the above parameter, but applied at query execution time. It specifies the number of nearest neighbors the algorithm will keep in the priority queue when looking for the answer.

    - **Meaning**: This parameter decides the scanning scope of the algorithm.

    - **Impact**: This is a key factor to directly trade-off between speed and accuracy:

      - Increasing `ef_search` will expand the search scope, helping the system not miss correct results, but response time will slow down.

      - Decreasing `ef_search` helps the system respond extremely fast, but there is a risk of returning results not closest to the question.

**Practical Configuration Strategy**

There is no perfect set of parameters for every problem. We need to adjust based on business requirements:

**Case 1: Real-time Chatbot Application**

- Requirement: Fast response, accepts small margin of error.

- Configuration: Keep `ef_search` at a low level, e.g., 50 - 100, to optimize latency.

**Case 2: Intensive Lookup System**

- Requirement: Absolute accuracy, no missing information allowed.

- Configuration: Set `M` high, e.g., 64, and increase `ef_search` to a high level, e.g., 400 - 500. Accept consuming some RAM and slightly slower response time to ensure accuracy reaches maximum level.

## Advanced Chunking Strategies (2025-2026) NEW

Beyond recursive and semantic chunking, three production-relevant techniques have emerged:

### Late Chunking (Jina AI, 2024)

Traditional chunking embeds each chunk independently, losing cross-chunk context (e.g., pronoun references). Late chunking reverses the order:

1.  Embed the **full document** first using a long-context model to get token-level embeddings

2.  **Then** apply chunk boundaries to pool token embeddings into chunk vectors

3.  Each chunk vector retains awareness of the full document context

``` mermaid
        graph TD
    subgraph "Traditional Chunking"
        D1["Document"] --> C1["Chunk 1"] --> E1["Embed"] --> V1["Vector 1"]
        D1 --> C2["Chunk 2"] --> E2["Embed"] --> V2["Vector 2"]
    end
    subgraph "Late Chunking"
        D2["Document"] --> E3["Embed full document<br/>(token-level)"]
        E3 --> P["Pool by chunk boundaries"]
        P --> V3["Vector 1 (context-aware)"]
        P --> V4["Vector 2 (context-aware)"]
    end
```

Improves retrieval 10-12% on documents with pronoun-heavy content and cross-references. Requires a model that outputs token-level embeddings (e.g., Jina models).

### Contextual Chunking (Anthropic-style)

Use an LLM to generate a brief context prefix for each chunk before embedding:

1.  For each chunk, ask the LLM: “Given this document, write a short context for this chunk”

2.  Prepend the context to the chunk text

3.  Embed the contextualized chunk

Example:

- Original chunk: “The company reported \$2.5B in Q3.”

- Context prefix: “This document is Acme Corp’s 2025 annual report. This section discusses Q3 financial results.”

- Embedded text: context prefix + original chunk

Consistently the top performer (+2-18% vs baseline), but adds LLM call overhead per chunk. Best for static document collections where indexing cost is amortized.

### Proposition-Based Chunking

Decompose text into atomic, standalone factual statements using an LLM:

- Input: A paragraph with multiple facts

- Output: Individual propositions, each self-contained

Example:

- Input: “Einstein was born in 1879 in Germany. He developed the theory of relativity.”

- Propositions: \[“Albert Einstein was born in 1879”, “Albert Einstein was born in Germany”, “Albert Einstein developed the theory of relativity”\]

Highest retrieval precision for fact-based queries, but expensive (requires LLM call per chunk) and slow. Best for knowledge base construction, not real-time indexing.

### Choosing a Chunking Strategy

| Strategy | Quality Gain | Indexing Cost | Best For |
|----|----|----|----|
| Recursive | Baseline | Very low | General purpose, first approach |
| Semantic | +5-10% | Low | Topic-coherent documents |
| Late Chunking | +10-12% | Medium | Cross-reference heavy docs |
| Contextual | +2-18% | High (LLM calls) | Static collections, high-value docs |
| Proposition-based | Highest precision | Very high | Knowledge bases, FAQ systems |
