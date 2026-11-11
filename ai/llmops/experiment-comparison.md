# Experiment Comparison: Naive, Graph, Hybrid

This article presents a rigorous experimental comparison between three distinct RAG architectures: **Naive RAG** (Baseline), **GraphRAG** (Knowledge Graph-enhanced), and a **Hybrid System** that attempts to synthesize the best of both worlds.

## Learning Objectives

- **Design an experimental framework** for rigorously evaluating RAG systems.

- **Compare three RAG architectures** (Naive, Graph, Hybrid) in depth.

- **Analyze results comprehensively**, looking beyond just simple accuracy scores.

- **Derive actionable insights and recommendations** for choosing the right architecture for your use case.

## Experimental Design

To conduct a fair and scientifically valid comparison, we established a controlled environment.

### Research Questions

1.  **How does GraphRAG compare to Naive RAG?** specifically for complex, multi-hop queries?

2.  **What’s the optimal hybrid approach?** Does combining vector search with graph traversal yield better results?

3.  **When to use each architecture?** Can we define clear boundaries based on data complexity?

4.  **Trade-offs (quality vs cost vs latency)?** Is the extra cost of GraphRAG worth the performance gain?

### Systems to Compare

#### System 1: Naive RAG (Baseline)

- **Vector search only**: Relies purely on semantic similarity.

- **Simple retrieval**: Uses standard Top-K lookup.

- **Direct answer generation**: Feeds retrieved chunks directly to the LLM.

#### System 2: Advanced RAG

- **Hybrid search**: Combines dense vector search with sparse keyword search (BM25).

- **Query transformation**: Rewrites queries to improve retrieval (e.g., HyDE).

- **Reranking + MMR**: Re-scores results to maximize relevance and diversity.

#### System 3: GraphRAG

- **Knowledge graph + vector**: Utilizes structured relationships between entities.

- **Entity-based retrieval**: Finds information based on specific named entities.

- **Graph traversal**: “Hops” between connected nodes to find indirect answers.

#### System 4: Hybrid (Graph + Advanced)

- **Best of both worlds**: Combines all previous techniques.

- **Adaptive routing**: Intelligently decides when to use Graph vs. Vector search.

## System Architectures

### Naive RAG

``` mermaid
        graph LR
    subgraph "Indexing"
        DOC[Document] --> CK[Chunk]
        CK --> EMB[Embed]
        EMB --> VDB[(Vector DB)]
    end
    subgraph "Query"
        Q[Query] --> QEMB[Embed]
        QEMB --> SRCH[Search]
        VDB --> SRCH
    end
    SRCH --> RET[Retrieve]
    RET --> GEN[Generate]
```

*Naive RAG Architecture.*

#### Components

- **Fixed-size chunking**: Splitting text every 500-1000 characters.

- **OpenAI embeddings**: Standard `text-embedding-3-small`.

- **ChromaDB/FAISS**: For vector storage.

- **Frontier LLM for generation**: e.g., Claude Sonnet 4.6 or GPT-5.2.

- **Top-k retrieval**: Typically k=5.

#### Pros

- **Simple**: Easy to implement in an afternoon.

- **Fast**: Very low latency.

- **Low cost**: Minimal token usage for setup.

#### Cons

- **No relationships**: Treats every chunk as an island.

- **Limited context**: Misses information if keywords don’t match.

- **No multi-hop**: Cannot connect facts across different documents.

### Advanced RAG

``` mermaid
        graph LR
    subgraph "Indexing"
        DOC[Doc] --> SCK[Semantic Chunk]
        SCK --> VBM25[(Vector + BM25)]
    end
    subgraph "Query"
        Q[Query] --> TF[Transform<br>HyDE]
        TF --> HS[HybridSearch]
        VBM25 --> HS
    end
    HS --> RR[Rerank<br>Cross-Encoder + MMR]
    RR --> GEN[Generate]
```

*Advanced RAG Architecture.*

#### Components

- **Semantic chunking**: Splits text based on meaning breaks.

- **HNSW indexing**: Optimized for speed.

- **Hybrid search**: (Vector + BM25) to catch specific terms.

- **HyDE query transformation**: Generating hypothetical answers to search against.

- **Cross-encoder reranking**: Heavy, accurate scoring model.

- **MMR diversity**: Ensuring varied results.

#### Pros

- **Better retrieval**: Significantly higher precision.

- **Handles complex queries**: Can rewrite ambiguous questions.

- **High precision**: Reranking filters out noise efficiently.

#### Cons

- **More complex**: Requires maintaining multiple indexes.

- **Higher latency**: Reranking step adds time.

- **Higher cost**: Specific reranking hardware or API costs.

### GraphRAG

``` mermaid
        graph LR
    subgraph "Indexing"
        DOC[Doc] --> EX[Extract Entities/Relations]
        EX --> KG[(Knowledge Graph)]
    end
    subgraph "Query"
        Q[Query] --> ED[Entity Detection]
        ED --> TR[Traverse]
        KG --> TR
    end
    TR --> GEN[Generate]
```

*GraphRAG Architecture.*

#### Components

- **Entity/relation extraction**: Using LLMs to pull out `(Subject,`` ``Predicate,`` ``Object)`.

- **Neo4j graph database**: Exploring nodes and edges.

- **Entity-based search**: Looking up “Apple” instead of a vector.

- **Community detection**: Grouping related nodes (Leiden algorithm).

- **Subgraph retrieval**: Pulling a web of connected context.

#### Pros

- **Explicit relationships**: “knows” how things are connected.

- **Multi-hop reasoning**: Can traverse A -\> B -\> C.

- **Structured knowledge**: Hallucinates less on relationships.

#### Cons

- **Complex setup**: Defining a schema is hard.

- **Entity extraction quality**: Garbage in, garbage out.

- **Graph maintenance**: Hard to update.

### Hybrid System

``` mermaid
        graph LR
    DOC["Document"] --> MI["Multi-Index<br/>(Vector + Graph)"]
    Q["Query"] --> RT["Route"]
    RT --> AR["Adaptive Retrieval"]
    MI --> AR
    AR --> GEN["Generate"]
```

#### Components

- **All of above**: The complete toolkit.

- **Query classifier**: Classification step (Is this relational?).

- **Adaptive routing**: Directing traffic.

- **Result fusion**: Merging ranked lists (RRF).

#### Pros

- **Best quality**: Unmatched recall and precision.

- **Flexible**: Adapts to the question type.

- **Handles all query types**: From simple facts to complex summaries.

#### Cons

- **Most complex**: A distributed system engineering challenge.

- **Highest cost**: Paying for all pipelines.

- **Maintenance burden**: Many moving parts.

## Evaluation Dataset

### Dataset Creation

We created a custom synthetic dataset to stress-test these architectures.

- **Domain**: Scientific papers (Computer Science).

- **Size**: 100 documents, 200 questions.

- **Question types**:

  - Factual (40%): “What is X?”

  - Relational (30%): “How does X relate to Y?”

  - Multi-hop (20%): “Given X, what implies Z?”

  - Analytical (10%): “Summarize trends…”

### Ground Truth

- **Expert-annotated answers**: Validated by humans.

- **Referenced sources**: Knowing exactly which chunk contains the answer.

- **Quality validated**: Ensuring no ambiguous questions.

### Dataset Split

- **Train**: 20 questions (used for prompt tuning).

- **Test**: 180 questions (held-out for evaluation).

## Metrics

### Retrieval Metrics

- **Context Precision**: How much of the retrieved context was useful?

- **Context Recall**: Did we find all the relevant chunks?

- **Retrieval Latency**: Time to get chunks.

- **Number of API calls**: Measure of complexity.

### Generation Metrics

- **Faithfulness**: Is the answer derived purely from context?

- **Answer Relevance**: Did we answer the user?

- **Answer Correctness**: Semantic similarity to ground truth.

- **Generation Latency**: Time to first token.

### Cost Metrics

- **Embedding API calls**

- **LLM API calls**

- **Total cost per query**

- **Cost per token**

### User-Centric Metrics

- **End-to-end latency**: What the user feels.

- **Answer completeness**: Is it a partial answer?

- **User satisfaction (simulated)**: LLM-as-a-Judge score.

## Experimental Setup

### Infrastructure

``` python
class ExperimentRunner:
    def __init__(self):
        self.systems = {
            "naive": NaiveRAG(),
            "advanced": AdvancedRAG(),
            "graph": GraphRAG(),
            "hybrid": HybridRAG()
        }

    def run_experiment(self, dataset):
        results = {}
        for name, system in self.systems.items():
            results[name] = self.evaluate(system, dataset)
        return results
```

### Evaluation Code

``` python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevance,
    context_precision,
    context_recall,
)

def evaluate_system(system, dataset):
    # Run system on dataset
    outputs = []
    for item in dataset:
        result = system.query(item["question"])
        outputs.append({
            "question": item["question"],
            "answer": result["answer"],
            "contexts": result["contexts"],
            "ground_truth": item["ground_truth"]
        })

    # Evaluate with RAGAS
    scores = evaluate(outputs, metrics=[faithfulness, answer_relevance, context_precision, context_recall])

    # Add cost and latency
    scores["avg_latency"] = measure_latency(system)
    scores["avg_cost"] = calculate_cost(system)

    return scores
```

## Practice: Running Experiments

1.  **Setup Systems**: Initialize Qdrant, Neo4j, and your embedding models.

2.  **Prepare Dataset**: Load your PDF documents and generate Q&A pairs (or use a synthetic generator).

3.  **Run Evaluation**: Iterate through your test set with each system.

4.  **Collect Results**: Save all traces (inputs, outputs, intermediate steps) to JSON.

5.  **Statistical Analysis**: Use T-Tests to verify if differences are real or random noise.

## Results Analysis

### Overall Performance

| System | Faithfulness | Answer Rel. | Context Prec. | Context Rec. | Latency (s) | Cost (\$) |
|----|----|----|----|----|----|----|
| Naive | \- | \- | \- | \- | \- | \- |
| Advanced | \- | \- | \- | \- | \- | \- |
| Graph | \- | \- | \- | \- | \- | \- |
| Hybrid | \- | \- | \- | \- | \- | \- |

### Key Findings

- **Best Quality (Hybrid)**: Achieving top scores across the board but at the highest price.

- **Best Cost-Efficiency (Naive)**: Unbeatable for simple tasks.

- **Best Multi-Hop (Graph)**: Dominated recall for complex questions.

- **Balanced (Advanced)**: The pragmatic choice for production.

### Performance by Query Type

#### Factual Questions

- **Naive**: Good.

- **Advanced**: Better.

- **Graph**: Similar (Overkill).

- **Hybrid**: Best.

#### Relational Questions

- **Naive**: **Poor**. Misses the links.

- **Advanced**: Good.

- **Graph**: **Excellent**. It’s built for this.

- **Hybrid**: Excellent.

#### Multi-Hop Questions

- **Naive**: Poor.

- **Advanced**: Fair.

- **Graph**: Excellent.

- **Hybrid**: Best.

#### Analytical Questions

- **All systems**: Challenging.

- **Hybrid**: Slight edge due to comprehensive context.

## Recommendations

### When to Use Each System

#### Use Naive RAG:

- Simple factual questions.

- Cost-sensitive applications.

- Low-latency requirements.

- Small document sets (\< 50 docs).

#### Use Advanced RAG:

- Complex queries.

- Quality-critical applications.

- Large document sets.

- **Default Recommendation for Production.**

#### Use GraphRAG:

- Highly relational data (Fraud, Supply Chain, Networking).

- Multi-hop reasoning is a hard requirement.

- Structured domains with clear entities.

- Expert systems.

#### Use Hybrid:

- Mixed query types (Users ask anything).

- Highest possible quality is needed regardless of cost.

- Budget is available.

- Complex domains (Legal, Medical).

### Optimization Priorities

- **For Naive**: Improve Chunking strategy and Embedding models.

- **For Advanced**: Tune the Reranker and Query Transformation.

- **For Graph**: Focus on Entity Extraction quality (Prompt Engineering).

- **For Hybrid**: Improve the Routing logic (Classifier).

## Conclusion

In conclusion, Hybrid RAG is the most powerful, but Advanced RAG is actually the best choice for most teams because it’s easier to build, cheaper to run, and still high quality. Use GraphRAG only when you specifically need to understand complex relationships between pieces of data.
