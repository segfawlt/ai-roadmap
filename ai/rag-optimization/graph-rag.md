# GraphRAG Implementation

This page explains how to build a GraphRAG system that combines a graph database (Neo4j) with vector-based retrieval to represent entities, relationships, and documents as a knowledge graph, enabling richer reasoning and more accurate LLM-generated answers.

## Learning Objectives

- Design GraphRAG architecture with graph databases

- Extract entities and relationships from documents

- Build and maintain knowledge graphs

- Integrate with LLM-based answer generation

## Sample Data

📄 **FSoft_HR.pdf** - Sample HR policy document

📥 Download Sample: `FSoft_HR.pdf` \| 👁️ Preview: `FSoft_HR.pdf` (file not available online)

This PDF contains HR policies, employee commitments, stakeholder information, and compliance requirements used throughout this implementation example.

------------------------------------------------------------------------

## GraphRAG Architecture

GraphRAG combines structured graph databases with vector-based retrieval to create a comprehensive knowledge representation system. Key components include:

1.  **Entity Extraction**: Identifying key entities and relationships from documents

2.  **Graph Storage**: Storing structured data in Neo4j

3.  **Semantic Search**: Using LLMs to understand natural language queries

4.  **Graph Traversal**: Leveraging relationships for context-aware answers

## 1. Setup & Data Class Definitions

### Environment Configuration

``` python
import os
os.environ["HF_HUB_DISABLE_SYMLINK_WARNING"] = "1"
os.environ["HF_HUB_ENABLE_HF_TRANSFER"] = "0"
from dotenv import load_dotenv
load_dotenv()

from langchain_neo4j import Neo4jGraph, GraphCypherQAChain
from langchain.chat_models import init_chat_model
from langchain_text_splitters import RecursiveCharacterTextSplitter
from docling.document_converter import DocumentConverter
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import date
from enum import Enum

# Initialize Neo4j connection
graph = Neo4jGraph(url="bolt://localhost:7687", username="neo4j", password="<your-password>")
```

### Defining Domain-Specific Data Classes

Define Pydantic models to structure extracted information from policy documents:

``` python
class ConstraintUnit(str, Enum):
    hours = "hours"
    dong = "dong"
    percent = "percent"
    other = "other"


class ConstraintPeriod(str, Enum):
    month = "month"
    year = "year"
    none = "none"


class Constraint(BaseModel):
    """Represents measurable limits or requirements"""
    metric: str
    value: float
    unit: ConstraintUnit
    period: ConstraintPeriod


class Commitment(BaseModel):
    """Represents obligations or promises within a policy"""
    description: str
    measurable: bool
    constraints: List[Constraint] = []


class PolicyClauseExtraction(BaseModel):
    """Top-level policy information extraction structure"""
    clause_title: str
    clause_text: str
    stakeholders: List[str] = []
    regulations: List[str] = []
    commitments: List[Commitment] = []
```

These classes serve as validation schemas for structured output from LLMs, ensuring consistency in extracted data.

The classes shown above are tailored for HR policy extraction. **You should define your own Pydantic models based on:**

- The type of documents you’re processing (contracts, research papers, technical docs, etc.)

- The specific information you want to extract

- The relationships and constraints relevant to your use case

For example, for medical documents you might extract: `Symptoms`, `Diagnoses`, `Treatments`, and `MedicationConstraints` instead. Adapt the schema to match your domain and extraction needs.

## 2. Document Processing & Data Extraction

### Load and Split Documents

``` python
source = "FSoft_HR.pdf"
converter = DocumentConverter()
doc = converter.convert(source).document
markdown_text = doc.export_to_markdown()

# Split into manageable chunks
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = text_splitter.split_text(markdown_text)
```

### Configure LLM for Structured Extraction

``` python
model = init_chat_model(
    "claude-sonnet-4-6",
    temperature=0.1,
    max_tokens=1000,
)

EXTRACTION_PROMPT = """
You are an information extraction engine.

Your task:
From the provided policy text chunk, extract structured policy information.

Rules:

1. A clause is a policy topic unit (e.g., "Working hours and overtime").
2. A commitment is a clear promise, obligation, or prohibition.
3. If the commitment contains measurable numeric limits, extract them as constraints.
4. Extract stakeholders mentioned explicitly (Employee, Partner, Board, Government, etc).
5. Extract legal or regulatory references explicitly mentioned.
6. Do NOT invent information.
7. If something does not exist, return empty list.

Text:
{chunk}
"""

model_with_structure = model.with_structured_output(PolicyClauseExtraction)
```

### Extract Policy Information from All Chunks

``` python
all_extractions = []
for i, chunk in enumerate(chunks):
    print(f"Processing chunk {i+1}/{len(chunks)}...")
    response = model_with_structure.invoke(
        EXTRACTION_PROMPT.format(chunk=chunk)
    )
    all_extractions.append(response)
    print(f"Extracted: {response.clause_title}")

print(f"\nTotal extractions: {len(all_extractions)}")
```

The LLM processes each chunk independently, returning structured `PolicyClauseExtraction` objects containing all relevant entities and relationships.

## 3. Ingesting Data into Neo4j

### Graph Schema Design

The knowledge graph consists of these node types and relationships:

- **PolicyClause** nodes store policy topics

- **Stakeholder** nodes represent affected parties

- **Regulation** nodes track legal references

- **Commitment** nodes capture obligations

- **Constraint** nodes define measurable limits

### Populate Graph Database

``` python
for extraction in all_extractions:
    # Create PolicyClause node
    clause_query = """
    MERGE (clause:PolicyClause {title: $title})
    SET clause.text = $text
    RETURN clause
    """
    graph.query(clause_query, {
        "title": extraction.clause_title,
        "text": extraction.clause_text
    })

    # Create Stakeholder nodes and relationships
    for stakeholder in extraction.stakeholders:
        stakeholder_query = """
        MERGE (s:Stakeholder {name: $name})
        WITH s
        MATCH (c:PolicyClause {title: $clause_title})
        MERGE (c)-[:AFFECTS]->(s)
        """
        graph.query(stakeholder_query, {
            "name": stakeholder,
            "clause_title": extraction.clause_title
        })

    # Create Regulation nodes and relationships
    for regulation in extraction.regulations:
        regulation_query = """
        MERGE (r:Regulation {name: $name})
        WITH r
        MATCH (c:PolicyClause {title: $clause_title})
        MERGE (c)-[:REFERENCES]->(r)
        """
        graph.query(regulation_query, {
            "name": regulation,
            "clause_title": extraction.clause_title
        })

    # Create Commitment nodes and their constraints
    for commitment in extraction.commitments:
        commitment_query = """
        MERGE (com:Commitment {description: $description})
        SET com.measurable = $measurable
        WITH com
        MATCH (c:PolicyClause {title: $clause_title})
        MERGE (c)-[:CONTAINS]->(com)
        """
        graph.query(commitment_query, {
            "description": commitment.description,
            "measurable": commitment.measurable,
            "clause_title": extraction.clause_title
        })

        # Create Constraint nodes
        for constraint in commitment.constraints:
            constraint_query = """
            MERGE (cons:Constraint {metric: $metric})
            SET cons.value = $value, cons.unit = $unit, cons.period = $period
            WITH cons
            MATCH (com:Commitment {description: $commitment_description})
            MERGE (com)-[:HAS_CONSTRAINT]->(cons)
            """
            graph.query(constraint_query, {
                "metric": constraint.metric,
                "value": constraint.value,
                "unit": constraint.unit.value,
                "period": constraint.period.value,
                "commitment_description": commitment.description
            })
```

**Key Points:**

- `MERGE` operations prevent duplicate nodes

- Relationships connect related entities

- Constraints are linked to commitments for complex constraints tracking

## 4. Retrieving Data & Generating Answers

### Query Graph with Natural Language

Use `GraphCypherQAChain` to convert natural language questions into Cypher queries and generate answers:

``` python
chain = GraphCypherQAChain.from_llm(
    init_chat_model("claude-sonnet-4-6", temperature=0),
    graph=graph,
    verbose=True,
    allow_dangerous_requests=True,
)

response = chain.invoke("how many policies that affect Employees?")
print(response)
```

`GraphCypherQAChain` automatically generates Cypher queries from natural language questions. However, for more control over query generation, you may want to create a **custom agent** that:

- Validates and refines generated Cypher queries before execution

- Applies domain-specific query optimization rules

- Implements custom fallback logic for complex queries

- Adds query caching or rate limiting

- Logs and monitors query performance

Consider implementing a wrapper around `GraphCypherQAChain` or using LangGraph agents when you need fine-grained control over Cypher generation and execution.

### How It Works

1.  **Question Processing**: The LLM understands the natural language query

2.  **Cypher Generation**: Generates appropriate Cypher queries to traverse the graph

3.  **Graph Traversal**: Executes queries on Neo4j

4.  **Answer Generation**: Converts query results into readable responses

## Benefits of GraphRAG Architecture

- **Structured Knowledge**: Relationships explicitly define how entities connect, so it can answer questions that RAG with vector similarity search cannot handle

- **Context-Aware Retrieval**: Graph traversal provides rich contextual information

- **Compliance Tracking**: Easy to audit which commitments affect which stakeholders

- **Complex Reasoning**: Combines graph traversal with LLM reasoning for sophisticated queries

## NOTE:

This might not be the optimal way to implement GraphRAG, given its reliance on specific types of structured data for an effective graph-based knowledge base. You can experiment with alternative approaches to achieve better results. For example, use hybrid retrieval with both vector search and graph exploration

------------------------------------------------------------------------

## GraphRAG Evolution: 2025-2026 NEW

### Microsoft GraphRAG v2 → v3

Microsoft’s open-source GraphRAG has evolved significantly:

| Version | Date | Key Changes |
|----|----|----|
| v2.0 | Feb 2025 | NLP-based graph extraction alongside LLM, DRIFT search, multi-index querying |
| v2.2 | Apr 2025 | OpenAI reasoning model support |
| v3.x | Late 2025 | Removed NetworkX (DataFrame-based ops), CSVTableProvider, memory profiling |

v2.0 introduced **DRIFT search** — a hybrid of vector similarity + graph traversal + keyword search with learned ranking. Reports show 40% improvement on multi-hop questions (HotpotQA benchmark).

### Production Alternatives

The GraphRAG ecosystem is no longer Microsoft-only:

| Tool | Approach | Best For |
|----|----|----|
| Microsoft GraphRAG | Global summarization + community detection | Large document collections, “what is the theme?” questions |
| Neo4j GraphRAG | Property graph + Cypher | Structured enterprise data, relationship queries |
| LlamaIndex PropertyGraphIndex | Lightweight graph construction | Quick prototyping, smaller datasets |
| LangChain Graph Integrations | Framework-integrated graph chains | LangChain-native stacks |

### When to Use GraphRAG vs Vector RAG

| Scenario | Recommended | Why |
|----|----|----|
| “What does document X say about Y?” | Vector RAG | Simple retrieval, no relationships needed |
| “How are entities A and B related?” | GraphRAG | Relationship traversal required |
| “What are the main themes across 1000 docs?” | GraphRAG (global search) | Needs cross-document summarization |
| “Find similar documents to this one” | Vector RAG | Embedding similarity is sufficient |
| Multi-hop reasoning (“Who reports to the CEO’s direct reports?”) | GraphRAG | Requires graph traversal |

The trend in 2026 is toward **hybrid approaches**: vector search for initial retrieval + graph traversal for relationship enrichment + keyword search for exact matches.
