# Embeddings & Vector Search

## What is an embedding?

An **embedding** is a dense vector representation of text (or any modality) in a continuous high-dimensional space, such that semantically similar inputs land near each other. Modern embedding models produce vectors of 384–3072 dimensions.

Nearest-neighbor search over embeddings is the foundation of semantic search, RAG retrieval, recommendation, and deduplication.

## Embedding model shortlist (2026)

| Provider | Model | Dim | Context | Notes |
|----|----|----|----|----|
| OpenAI | `text-embedding-3-small` | 1536 | 8191 | Fast, cheap default |
| OpenAI | `text-embedding-3-large` | 3072 | 8191 | Best quality |
| Voyage | `voyage-3` | 1024 | 32000 | Strong on retrieval benchmarks |
| Cohere | `embed-v4.0` | 1536 | 128000 | Multilingual, long context |
| Open source | `BAAI/bge-m3` | 1024 | 8192 | Free, multilingual, hybrid |
| Open source | `sentence-transformers/all-MiniLM-L6-v2` | 384 | 512 | Tiny, fast, weaker recall |
| Jina AI | `jina-colbert-v2` | Multi-vector | 8192 | Late interaction (ColBERT), 89 languages |
| Google | `gemini-embedding-002` | 3072 | 2048 | Multimodal (text+image+video+audio) |

Rule of thumb: **start with `text-embedding-3-small`**. If quality is not enough, switch to `text-embedding-3-large` or `voyage-3`. Only drop to MiniLM when you need to run locally or at zero cost.

## Modern embedding techniques NEW

### Matryoshka Representation Learning (MRL)

Models trained with MRL (e.g., `text-embedding-3-large`, `gemini-embedding-002`) produce embeddings where the first *N* dimensions are a valid lower-dimensional embedding. Truncate from 3072 → 256 dims to cut storage ~92% with \<2% quality loss.

``` python
# OpenAI: request fewer dimensions
embeddings = client.embeddings.create(
    model="text-embedding-3-large",
    input="Hello world",
    dimensions=256,  # truncated MRL embedding
)
```

### Late interaction (ColBERT)

Instead of a single vector per document, ColBERT produces one vector per token. Retrieval uses MaxSim: for each query token, find the max similarity across all document tokens, then sum. Much higher retrieval quality at the cost of more storage.

### Multimodal embeddings

`gemini-embedding-002` (March 2026) maps text, images, video, audio, and PDFs into a single vector space. No OCR or captioning pipeline needed for document understanding.

## Similarity metrics

| Metric | Formula | When to use |
|----|----|----|
| Cosine | `dot(a,b)`` ``/`` ``(‖a‖‖b‖)` | Default for text embeddings. Range: -1..1. |
| Dot product | `dot(a,b)` | When vectors are already normalized (faster). |
| Euclidean (L2) | `‖a-b‖` | Image embeddings; rarely for text. |

For text, always prefer **cosine** unless your embedding model specifies otherwise. Most modern text embeddings are L2-normalized, making cosine and dot product equivalent.

## Chunking strategy

Embeddings operate on chunks, not whole documents. Poor chunking is the single most common cause of bad RAG.

**Baseline (works most of the time):**

``` python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=120,
    separators=["\n\n", "\n", ". ", " ", ""],
)
chunks = splitter.split_documents(docs)
```

**Advanced:** semantic chunking (split on sentence-boundary similarity drops) or structure-aware chunking (split on Markdown headings, HTML sections). See [Advanced Indexing](../rag-optimization/advanced-indexing.md).

## Vector stores

### pgvector (PostgreSQL + `langchain-postgres`)

Best choice when you already have Postgres. Supports rich metadata filtering via SQL.

``` bash
pip install -qU langchain-postgres
```

**Important:** the package requires psycopg3. Connection string format: `postgresql+psycopg://user:pass@host:5432/db` (not `psycopg2://`).

``` python
from langchain_postgres import PGVector
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

store = PGVector(
    embeddings=embeddings,
    collection_name="my_docs",
    connection="postgresql+psycopg://user:pass@localhost:5432/mydb",
)

store.add_documents(docs, ids=doc_ids)
results = store.similarity_search("query", k=5)

# Metadata filter operators: $eq, $ne, $lt, $in, $and, $or, $like, $ilike
results = store.similarity_search(
    "query",
    k=5,
    filter={"category": {"$eq": "policy"}, "year": {"$lt": 2025}},
)
```

### Qdrant (`langchain-qdrant`)

Best for dense+sparse hybrid retrieval and when you want a dedicated vector database.

``` bash
pip install -qU langchain-qdrant
```

``` python
from langchain_qdrant import QdrantVectorStore
from qdrant_client import QdrantClient
from qdrant_client.http.models import Distance, VectorParams

client = QdrantClient(":memory:")  # or url="http://localhost:6333"
client.create_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)

store = QdrantVectorStore(
    client=client,
    collection_name="docs",
    embedding=embeddings,
)
store.add_documents(docs, ids=uuids)

# retrieval_mode options: "dense" (default), "sparse", "hybrid"
results = store.similarity_search("query", k=5)
```

### Chroma (`langchain-chroma`)

Best for quick prototyping and local-first workflows.

``` bash
pip install -qU langchain-chroma
```

``` python
from langchain_chroma import Chroma

store = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    persist_directory="./chroma_db",
)
results = store.similarity_search("query", k=5)
```

## Hybrid search: dense + sparse

Dense retrieval (vector similarity) excels at semantic understanding. Sparse retrieval (BM25) excels at exact keyword matching. Combining both via **Reciprocal Rank Fusion (RRF)** typically outperforms either alone — especially on queries with proper names, SKUs, or error codes.

``` python
# RRF pseudocode
def rrf(rank_lists: list[list[str]], k: int = 60) -> list[str]:
    scores: dict[str, float] = {}
    for ranks in rank_lists:
        for r, doc_id in enumerate(ranks):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + r + 1)
    return sorted(scores, key=scores.get, reverse=True)
```

See [Advanced Retrieval Strategies](../rag-optimization/hybrid-search.md) for the full walkthrough.

## Re-ranking

After first-stage retrieval, pass the top-50 results through a **cross-encoder** (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2` or Cohere’s `rerank-v3`) to re-score them with deeper attention. This typically lifts precision by 5–15 points.

See [Re-ranking](../rag-optimization/post-retrieval.md) for details.

## Practice

### 1. Embedding comparison

Embed the following three sentences with `text-embedding-3-small` and compute pairwise cosine similarity:

1.  “A dog is chasing a ball in the park.”

2.  “A puppy plays fetch in the garden.”

3.  “The stock market crashed yesterday.”

Expected: (1,2) ≳ 0.7; (1,3) and (2,3) ≲ 0.3. Verify your intuition.

### 2. Chunking sensitivity

Take a 20-page PDF and index it twice:

- Run A: `chunk_size=400,`` ``chunk_overlap=40`

- Run B: `chunk_size=1200,`` ``chunk_overlap=180`

Write 10 test questions and measure top-5 recall on both runs. Report which chunk size works better and why.

### 3. pgvector end-to-end

Spin up Postgres with the pgvector extension (Docker one-liner works). Index 1,000 Wikipedia articles using `text-embedding-3-small` via `langchain-postgres`. Implement a metadata filter so that a query can be restricted to articles from a specific category. Verify both the unrestricted and filtered searches return reasonable results.

### 4. Hybrid search with RRF

Build a hybrid search that combines:

1.  Dense vector search (any of the three stores above).

2.  BM25 keyword search via `rank-bm25`.

Fuse the two ranked lists with Reciprocal Rank Fusion (k=60). Compare the fused top-5 against the dense-only top-5 on 10 queries that include proper names or codes. Hybrid should outperform dense-only on those queries.

### 5. Cross-encoder re-ranking

Retrieve top-50 from your best retriever. Re-rank with `cross-encoder/ms-marco-MiniLM-L-6-v2` and keep the top-5. Measure precision@5 before and after re-ranking on your test set.

Expected: precision@5 should improve by 5–15 percentage points.

## Review Questions

1.  **What similarity metric is the default choice for text embeddings?**

    - A. Cosine similarity

    - B. Manhattan distance

    - C. Jaccard index

    - D. Hamming distance

2.  **For most teams starting a RAG project, which embedding model is the recommended first choice?**

    - A. `all-MiniLM-L6-v2` (local, 384 dim)

    - B. `text-embedding-3-small` (OpenAI, 1536 dim)

    - C. A custom model trained from scratch

    - D. `text-embedding-ada-002` (legacy)

3.  **What is the pgvector connection string format required by `langchain-postgres`?**

    - A. `postgres://user:pass@host/db`

    - B. `postgresql+psycopg2://user:pass@host/db`

    - C. `postgresql+psycopg://user:pass@host/db` (psycopg3)

    - D. `jdbc:postgresql://host/db`

4.  **Why does hybrid search (dense + sparse) outperform dense-only on queries containing exact SKUs or error codes?**

    - A. Sparse retrieval is faster

    - B. BM25 excels at exact keyword matching, which pure vector similarity can miss

    - C. Hybrid uses more memory

    - D. Dense vectors don’t work at all

5.  **Which Qdrant parameter enables hybrid dense+sparse retrieval?**

    - A. `search_type="both"`

    - B. `retrieval_mode="hybrid"`

    - C. `mode="mixed"`

    - D. `use_bm25=True`

6.  **What is the primary purpose of chunk overlap when splitting documents?**

    - A. To make chunks larger for free

    - B. To avoid cutting a relevant passage exactly at a chunk boundary, which would make it unretrievable

    - C. To confuse the embedding model

    - D. To save disk space

7.  **A cross-encoder re-ranker is used after which step in a typical RAG pipeline?**

    - A. Before the first-stage retriever runs

    - B. After first-stage retrieval, to re-score the top-N candidates with deeper attention

    - C. Instead of the embedding model

    - D. On the final answer text

8.  **What is Reciprocal Rank Fusion (RRF) used for?**

    - A. Training embedding models

    - B. Combining ranked lists from multiple retrievers into a single fused ranking

    - C. Compressing vectors

    - D. Chunking documents

9.  **When would you prefer Chroma over pgvector for a vector store?**

    - A. Never — always use pgvector

    - B. For quick prototyping and local-first workflows without an existing Postgres instance

    - C. When you need SQL metadata filters

    - D. When you have \>100M vectors

10. **Most modern text embeddings are L2-normalized. What does this imply about cosine similarity vs dot product?**

    - A. They produce different rankings

    - B. They produce equivalent rankings (cosine = dot product when vectors are unit length)

    - C. Dot product is always slower

    - D. Cosine is impossible to compute

**View Answer Key**

1.  **A** — Cosine is the default for text embeddings.

2.  **B** — `text-embedding-3-small` is the recommended starting point; upgrade to `-large` or `voyage-3` if quality demands.

3.  **C** — `langchain-postgres` requires the psycopg3 driver.

4.  **B** — BM25 catches exact matches that dense vectors often miss.

5.  **B** — Qdrant’s `retrieval_mode` parameter with `"hybrid"` enables dense+sparse search.

6.  **B** — Overlap ensures relevant passages aren’t lost at chunk boundaries.

7.  **B** — Re-ranking operates on the top-N candidates from first-stage retrieval.

8.  **B** — RRF fuses multiple ranked lists, typically combining dense and sparse results.

9.  **B** — Chroma is great for local prototyping; pgvector wins when Postgres is already in your stack.

10. **B** — With unit-length vectors, cosine similarity and dot product produce identical rankings; dot product is just faster.
