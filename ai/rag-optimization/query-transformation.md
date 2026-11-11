# Query Transformations

In the previous sections, we assumed that the user’s question is always clear, semantically complete, and matches the content in the document. However, reality is rarely that perfect.

Users often tend to ask short questions, lacking context, or ask multiple issues at once. For example: instead of asking “What date do remote work regulations apply from?”, they might just type “remote work”. If we feed this raw question directly into the RAG system, search results are often very poor because the question’s vector does not match the vector of detailed legal documents.

To solve this problem, we use the **Query Transformation** technique. The core idea is to use an LLM to rewrite, expand, or break down the user’s question into better versions before performing the search.

## Hypothetical document embeddings (HyDE)

One of the biggest challenges of Vector Search is the semantic asymmetry between the question and the answer. Questions are often short and interrogative, while documents are long and affirmative/descriptive.

HyDE is a technique to overcome this by improved using the creativity of the LLM. Instead of searching based on the question, we ask the LLM to provide a hypothetical answer, then use this hypothetical answer to find the real document.

**Mechanism of Operation**

The HyDE process takes place in three steps:

1.  **Generate**: The system asks the LLM to write a hypothetical answer paragraph for the user’s question. Note that the information in this paragraph may be factually incorrect, but the writing style and technical vocabulary used will resemble the actual document.

2.  **Encode**: Pass this hypothetical paragraph through the embedding model to create a vector.

3.  **Retrieve**: Use this vector to search in the database. Since the vector of the “fake answer” will be closer to the vector of the “real answer” than the vector of the “question”, search results are usually more accurate.

**Illustrative Example**

``` mermaid
        graph LR
    Q["User Question<br>'How to handle blue screen error'"]
    Q -->|"1. Generate"| LLM[LLM]
    LLM --> HD["Hypothetical Answer<br>'To fix BSOD, restart, check stop code,<br>update driver, enter Safe Mode...'"]
    HD -->|"2. Encode"| EM[Embedding Model]
    EM --> HV[Hypothetical Vector]
    HV -->|"3. Retrieve"| SS[Similarity Search]
    DB[(Vector Store)] --> SS
    SS --> RD["Real Documents<br>(technical instructions)"]
```

**User Question**: “How to handle blue screen error.”

**Problem**: The question is too short, the vector might mistakenly match documents describing screen colors.

**HyDE Generation (LLM drafting)**: “To fix the Blue Screen of Death (BSOD) error on Windows, you need to restart the computer, check the stop code, update the graphics card driver, or enter Safe Mode to remove conflicting software…”

**Result**: The system uses the draft paragraph above to search. Thanks to technical keywords like “BSOD”, “driver”, “Safe Mode” appearing in the draft, the system easily finds the exact technical instruction document in the database.

## Query Decomposition

This technique is particularly useful for complex questions where a single text passage cannot contain enough information to answer.

If a user asks a question that requires comparing or aggregating information from multiple sources, simple searching often fails because the question vector will hang in between different topics. Query Decomposition solves this by breaking the large problem into simpler sub-problems.

**Strategy**

The system uses an LLM to analyze the original question and split it into a sequence of independent sub-questions.

1.  **Breakdown**: Split multi-intent questions into single-intent questions.

2.  **Retrieval**: Perform document search for each separate sub-question. This ensures each search has a clear goal and high accuracy.

3.  **Synthesis**: Aggregate text segments found from all steps above and give them to the LLM to answer the original initial question.

``` mermaid
        graph TD
    OQ["Original Complex Question"] -->|"1. Breakdown"| LLM[LLM]
    LLM --> SQ1["Sub-query 1<br>(single intent)"]
    LLM --> SQ2["Sub-query 2<br>(single intent)"]
    SQ1 -->|"2. Retrieval"| R1["Retrieved Docs 1"]
    SQ2 -->|"2. Retrieval"| R2["Retrieved Docs 2"]
    R1 --> SYN[LLM Synthesis]
    R2 --> SYN
    SYN -->|"3. Synthesis"| ANS["Final Answer"]
```

**Illustrative Example**

**User Question**: “Compare the revenue of iPhone 15 and Samsung S24 in Q1 2024.”

**Problem**: There is no single document containing this comparison table. Information is scattered in Apple’s financial report and Samsung’s report.

**Decomposition Process**:

- **Sub-query 1**: “What is iPhone 15 revenue in Q1 2024?” → Found in Apple Report.

- **Sub-query 2**: “What is Samsung S24 revenue in Q1 2024?” → Found in Samsung Report.

**Final Generation**: The LLM receives both figures from the two searches and self-aggregates them into a complete comparison answer.

In summary, Query Transformation acts as an intelligent editor, helping to edit and reorient user questions before sending them to the lookup department, ensuring that the system always correctly understands the true intent behind concise commands.

## Additional Query Transformation Techniques NEW

### Multi-Query Generation

Instead of searching with one query, use the LLM to generate 3–5 different phrasings of the same question, search with all of them, then merge the results. This increases recall by covering different angles — different wordings activate different embedding neighborhoods in the vector store.

``` mermaid
        graph TD
    OQ["Original Query"] --> LLM[LLM]
    LLM --> Q1["Query 1"]
    LLM --> Q2["Query 2"]
    LLM --> Q3["Query 3"]
    Q1 --> R1["Retrieve 1"]
    Q2 --> R2["Retrieve 2"]
    Q3 --> R3["Retrieve 3"]
    R1 --> MD[Merge & Deduplicate]
    R2 --> MD
    R3 --> MD
    MD --> FR["Final Results"]
```

**Original**: “database connection issues”

**Generated queries**:

- “How to fix connection timeout error when connecting to database”

- “Handle Access Denied error for root user in MySQL/PostgreSQL”

- “Guide to check firewall blocking port 5432 or 3306”

Each query targets a different failure mode, so together they surface a broader set of relevant documents than the original terse phrase ever could.

### Step-Back Prompting

Ask the LLM to first generate a more abstract, higher-level question before retrieving. This is especially useful for highly specific questions that benefit from broader context — the retrieved documents then cover the underlying principles, which helps the LLM reason toward the specific answer.

**Example**:

| Stage | Query |
|----|----|
| Original (specific) | “What happens to pressure at 100 km altitude?” |
| Step-back (abstract) | “What are the physics principles governing atmospheric pressure at different altitudes?” |
| Retrieval | Broader physics docs covering the barometric formula, scale height, etc. |
| Final generation | LLM grounds the specific answer in the retrieved principles |

The key insight is that vector search performs better when the query matches the level of abstraction of the stored documents. Most knowledge bases contain explanatory or reference material written at a conceptual level, so an abstract step-back query is a better semantic match than a narrow, highly specific one.

### Query Routing

Query routing is not a transformation per se, but a critical pre-retrieval decision: route different query types to different retrieval backends rather than always hitting the same vector store.

``` mermaid
        graph TD
    Q["Query"] --> R["Router (LLM classifier)"]
    R -->|"factual"| VS["Vector Store"]
    R -->|"recent news"| WS["Web Search (Tavily)"]
    R -->|"structured data"| SQL["SQL Database"]
    R -->|"code"| CS["Code Search"]
```

The router is typically a lightweight LLM call that classifies the query into a category, then dispatches it to the appropriate backend. This is especially important in **Modular** and **Agentic RAG** architectures where the system has access to multiple knowledge sources with different strengths:

- A vector store excels at semantic similarity over large document corpora.

- A web search API (e.g., Tavily) handles time-sensitive or current-events queries.

- A SQL database is the right choice when the question involves aggregation or filtering over structured records.

- A code search index retrieves exact symbol definitions, function signatures, or file paths.

Without routing, every query hits a single backend and the system silently fails on query types that backend cannot serve well.
