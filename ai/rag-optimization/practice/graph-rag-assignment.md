# Assignment: GraphRAG Implementation

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | GraphRAG with Neo4j and Entity Extraction |
| **Course** | RAG and Optimization |
| **Project Name** | `graph-rag-system` |
| **Estimated Time** | 150 minutes |
| **Framework** | Python 3.11+, LangChain 1.x, Neo4j, Anthropic (or OpenAI) API, Pydantic |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Design** a GraphRAG architecture combining graph and vector databases

- **Implement** entity and relationship extraction from documents using LLMs

- **Build** and populate a knowledge graph in Neo4j

- **Create** Cypher queries for graph traversal and retrieval

- **Integrate** graph-based retrieval with LLM answer generation

------------------------------------------------------------------------

## Problem Description

Your organization has policy documents, contracts, and technical specifications that contain rich relationships between entities (stakeholders, regulations, commitments, etc.). Traditional vector search struggles to answer queries like:

- “Which policies affect both Employees and Partners?”

- “What commitments have measurable constraints?”

- “Show all regulations referenced by the Leave Policy”

Your task is to implement a **GraphRAG** system that extracts entities and relationships, stores them in Neo4j, and enables relationship-aware retrieval.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Neo4j Database (Desktop or Docker)

- Required packages:

  - `langchain[anthropic]` \>= 1.0 \# or `langchain[openai]`

  - `langchain-neo4j` \>= 0.1.0

  - `pydantic` \>= 2.0.0

  - `docling` or `pypdf` for document processing

### Neo4j Setup

``` bash
# Docker setup
docker run -d --name neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password \
  neo4j:latest
```

------------------------------------------------------------------------

## Tasks

### Task 1: Define Domain Schema (20 points)

1.  **Design Pydantic models** for your domain entities:

    - Identify at least 4 entity types from your documents

    - Define relationships between entities

    - Include constraints and measurable properties

2.  **Example schema structure**:

    ``` python
    class Entity(BaseModel):
        name: str
        type: str
        properties: dict

    class Relationship(BaseModel):
        source: str
        target: str
        relation_type: str
    ```

3.  **Document your schema** with:

    - Entity type descriptions

    - Relationship type definitions

    - Example instances from your domain

### Task 2: Entity and Relationship Extraction (30 points)

1.  **Implement an extraction pipeline**:

    - Load and chunk documents

    - Use LLM with structured output to extract entities

    - Extract relationships between entities

    - Handle extraction errors and edge cases

2.  **Design extraction prompts** that:

    - Provide clear instructions for entity identification

    - Include examples (few-shot learning)

    - Specify output format matching your Pydantic models

3.  **Quality checks**:

    - Validate extracted entities against schema

    - Handle duplicate entities across chunks

    - Log extraction statistics (entities/chunk, relationship types)

### Task 3: Build Knowledge Graph (25 points)

1.  **Populate Neo4j** with extracted data:

    - Create nodes for each entity type

    - Create relationships between entities

    - Use `MERGE` to prevent duplicates

    - Add properties to nodes and relationships

2.  **Implement graph queries**:

    - Count entities by type

    - Find entities with specific relationships

    - Traverse multi-hop relationships

    - Aggregate information across connected nodes

3.  **Example queries to implement**:

    ```
    // Find all entities related to a specific policy
    MATCH (p:Policy {name: $policy_name})-[r]->(e)
    RETURN p, r, e
    ```

### Task 4: GraphRAG Query Pipeline (25 points)

1.  **Implement natural language to Cypher translation**:

    - Use `GraphCypherQAChain` or custom implementation

    - Handle query validation and error recovery

    - Support common question patterns

2.  **Create a test set** with 10 queries:

    - Entity lookup queries (5)

    - Relationship traversal queries (3)

    - Aggregation queries (2)

3.  **Demonstrate answers** that:

    - Leverage graph relationships

    - Would be difficult/impossible with vector search alone

    - Combine information from multiple connected entities

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code (Jupyter notebook or Python scripts)

`README.md` with setup and usage instructions

Schema documentation (entity types, relationships)

Sample Cypher queries and results

Screenshots of Neo4j graph visualization

### Submission Checklist

Pydantic models correctly validate extracted data

Extraction pipeline processes documents without errors

Neo4j graph is populated with entities and relationships

Natural language queries return correct results

Documentation explains the graph schema design decisions

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                         | Points  |
|----------------------------------|---------|
| Schema design quality            | 15      |
| Extraction pipeline correctness  | 20      |
| Prompt engineering effectiveness | 10      |
| Graph population implementation  | 15      |
| Cypher query implementation      | 15      |
| Query pipeline integration       | 15      |
| Code quality and documentation   | 10      |
| **Total**                        | **100** |

------------------------------------------------------------------------

## Hints

- Start with a small document set (2-3 pages) to iterate on your schema

- Use `model.with_structured_output()` for reliable JSON extraction from LLMs

- Test Cypher queries in Neo4j Browser before implementing in code

- Consider the companion notebooks `05-graph_rag_v1.ipynb` and `05-graph_rag_v2.ipynb`

- The sample `FSoft_HR.pdf` provides a good starting point for HR policy extraction
