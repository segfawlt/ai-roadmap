# Quiz

## GraphRAG Implementation

**Question 1:** What two technologies does GraphRAG combine to create a comprehensive knowledge representation system?

- **A.** SQL databases and BM25.

- **B.** Structured graph databases and vector-based retrieval.

- **C.** Flat indexing and HNSW graphs.

- **D.** Cross-Encoders and Bi-Encoders.

**Answer:** B

**Question 2:** In the GraphRAG architecture, what is the role of ‘Entity Extraction’?

- **A.** Converting PDFs to images.

- **B.** Identifying key entities and relationships from documents.

- **C.** Traversing the graph to find answers.

- **D.** Storing structured data in Neo4j.

**Answer:** B

**Question 3:** Which database technology is explicitly mentioned for storing the graph data?

- **A.** MongoDB

- **B.** MySQL

- **C.** Neo4j

- **D.** Redis

**Answer:** C

**Question 4:** What is defined as a ‘clear promise, obligation, or prohibition’ in the extraction rules?

- **A.** A constraint.

- **B.** A clause.

- **C.** A commitment.

- **D.** A regulation.

**Answer:** C

**Question 5:** What must the information extraction engine do if something does not exist in the text chunk?

- **A.** Invent hypothetical data.

- **B.** Return an empty list.

- **C.** Throw a system error.

- **D.** Perform a web search.

**Answer:** B

**Question 6:** Which specific Python class is used to convert natural language questions into Cypher queries?

- **A.** DocumentConverter

- **B.** Neo4jGraph

- **C.** GraphCypherQAChain

- **D.** RecursiveCharacterTextSplitter

**Answer:** C

**Question 7:** In the predefined Pydantic models, what are the four possible options for a ‘ConstraintUnit’?

- **A.** day, week, month, year

- **B.** hours, dong, percent, other

- **C.** high, medium, low, none

- **D.** true, false, yes, no

**Answer:** B

**Question 8:** What type of node represents affected parties in the knowledge graph?

- **A.** PolicyClause nodes

- **B.** Constraint nodes

- **C.** Stakeholder nodes

- **D.** Regulation nodes.

**Answer:** C

**Question 9:** Which Cypher keyword is used heavily in the ingestion script to prevent the creation of duplicate nodes?

- **A.** CREATE

- **B.** INSERT

- **C.** UPDATE

- **D.** MERGE.

**Answer:** D

**Question 10:** What relationship connects a Commitment node to a Constraint node in the Cypher queries?

- **A.** \[:AFFECTS\]

- **B.** \[:REFERENCES\]

- **C.** \[:CONTAINS\]

- **D.** \[:HAS_CONSTRAINT\]

**Answer:** D

**Question 11:** Why are Pydantic classes utilized during the document extraction phase?

- **A.** To split text into chunks.

- **B.** To connect to the Neo4j database.

- **C.** To serve as validation schemas for structured output from LLMs, ensuring consistency.

- **D.** To increase the temperature of the LLM.

**Answer:** C

**Question 12:** What does a ‘Regulation’ node specifically track?

- **A.** Measurable limits.

- **B.** Legal references.

- **C.** Policy topics.

- **D.** Employee obligations.

**Answer:** B

**Question 13:** Why might you want to create a custom agent instead of relying solely on GraphCypherQAChain?

- **A.** Because GraphCypherQAChain cannot connect to Neo4j.

- **B.** To validate and refine generated Cypher queries, apply domain-specific optimization, and implement fallback logic.

- **C.** Because GraphCypherQAChain requires a local LLM to run.

- **D.** To automatically chunk PDF documents.

**Answer:** B

**Question 14:** What specific problem does GraphRAG solve that vector similarity search alone cannot handle?

- **A.** Generating high-quality images from text.

- **B.** Answering questions that require explicit relationships to define how entities connect.

- **C.** Translating documents efficiently.

- **D.** Searching for exact BM25 keywords.

**Answer:** B

**Question 15:** In the prompt rules provided to the LLM for extraction, how should measurable numeric limits within a commitment be handled?

- **A.** They should be ignored.

- **B.** They should be extracted as constraints.

- **C.** They should be saved as separate Regulation nodes.

- **D.** They should be converted to standard SI units.

**Answer:** B

**Question 16:** How are multiple constraints connected to a single commitment within the graph schema?

- **A.** Through direct links to Stakeholders.

- **B.** By creating separate independent sub-graphs.

- **C.** Constraints are linked to commitments for complex constraints tracking via the HAS_CONSTRAINT relationship.

- **D.** They are concatenated into a single string within the Commitment node.

**Answer:** C

**Question 17:** What is listed as a potential drawback or cautionary note regarding this specific GraphRAG implementation?

- **A.** It relies heavily on specific types of structured data to build an effective knowledge base.

- **B.** It is too fast to monitor properly.

- **C.** Neo4j does not support Python integrations.

- **D.** It cannot handle HR policy documents.

**Answer:** A

**Question 18:** What is the fourth step in the answer generation process when querying the graph with natural language?

- **A.** Graph Traversal

- **B.** Question Processing

- **C.** Answer Generation: Converts query results into readable responses.

- **D.** Cypher Generation.

**Answer:** C

**Question 19:** What relationship is established between a PolicyClause node and a Stakeholder node?

- **A.** \[:REFERENCES\]

- **B.** \[:CONTAINS\]

- **C.** \[:AFFECTS\]

- **D.** \[:HAS_CONSTRAINT\]

**Answer:** C

**Question 20:** According to the tips provided, how should you adapt the Pydantic schemas if you were processing medical documents?

- **A.** Use the HR schema exactly as is

- **B.** Extract ‘Symptoms’, ‘Diagnoses’, ‘Treatments’, and ‘MedicationConstraints’ instead

- **C.** Disable structured extraction completely.

- **D.** Only extract Stakeholders and Regulations.

**Answer:** B
