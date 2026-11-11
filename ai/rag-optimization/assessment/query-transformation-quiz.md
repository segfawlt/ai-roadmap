# Quiz

## Query Transformation

**Question 1:** Why do raw user questions often yield poor search results in a RAG system?

- **A.** The questions are too long.

- **B.** The question’s vector does not match the vector of detailed legal documents.

- **C.** The database is too small.

- **D.** The system uses keyword search instead of vectors.

**Answer:** B

**Question 2:** What is the core idea of the Query Transformation technique?

- **A.** To compress the database size.

- **B.** To manually review user queries.

- **C.** To use an LLM to rewrite, expand, or break down the user’s question.

- **D.** To translate questions into multiple languages.

**Answer:** C

**Question 3:** What are the three steps of the HyDE process?

- **A.** Encode, Decode, Generate

- **B.** Generate, Encode, Retrieve

- **C.** Retrieve, Read, Rank

- **D.** Breakdown, Retrieval, Synthesis

**Answer:** B

**Question 4:** In the HyDE generation step, what does the system ask the LLM to do?

- **A.** Write a hypothetical answer paragraph for the user’s question.

- **B.** Search the internet for an answer.

- **C.** Break the question into sub-queries.

- **D.** Translate the question.

**Answer:** A

**Question 5:** What is a common characteristic of user questions compared to documents?

- **A.** They are longer and descriptive.

- **B.** They are identical in length.

- **C.** They are often short and interrogative.

- **D.** They use more complex technical vocabulary.

**Answer:** C

**Question 6:** What might happen if a user asks a very short question like ‘How to handle blue screen error’ without HyDE?

- **A.** The system crashes.

- **B.** The vector might mistakenly match documents describing screen colors.

- **C.** The system automatically downloads new drivers.

- **D.** The query is rejected.

**Answer:** B

**Question 7:** In the HyDE process, why is the ‘fake answer’ useful for retrieval?

- **A.** Its vector is closer to the vector of the ‘real answer’ than the question’s vector.

- **B.** It is always factually 100% accurate.

- **C.** It bypasses the embedding model completely.

- **D.** It relies purely on BM25 keyword matching.

**Answer:** A

**Question 8:** Which statement is true regarding the factual accuracy of the paragraph generated in the HyDE Generate step?

- **A.** It is guaranteed to be factually perfect.

- **B.** It must be verified by a human.

- **C.** The information may be factually incorrect, but the style resembles the actual document.

- **D.** It only contains information already present in the user’s prompt.

**Answer:** C

**Question 9:** When is the Query Decomposition technique particularly useful?

- **A.** When the database is empty.

- **B.** For simple, one-word queries.

- **C.** For complex questions where a single text passage cannot contain enough information to answer.

- **D.** When the user wants a hypothetical answer.

**Answer:** C

**Question 10:** What happens to a simple search vector if a question requires aggregating information from multiple sources?

- **A.** It finds the perfect document immediately.

- **B.** It hangs in between different topics.

- **C.** It triggers a system timeout.

- **D.** It switches to keyword search automatically.

**Answer:** B

**Question 11:** What is the first step in the Query Decomposition strategy?

- **A.** Split multi-intent questions into single-intent questions.

- **B.** Aggregate all text segments.

- **C.** Generate a hypothetical answer.

- **D.** Translate the query.

**Answer:** A

**Question 12:** What does the Synthesis step do in Query Decomposition?

- **A.** Creates hypothetical answers.

- **B.** Aggregates text segments found from all steps and gives them to the LLM.

- **C.** Splits the question into multiple intents.

- **D.** Generates a new vector for the question.

**Answer:** B

**Question 13:** How does Query Transformation ensure the system understands the true intent behind concise commands?

- **A.** By skipping the embedding step.

- **B.** By acting as an intelligent editor to edit and reorient user questions.

- **C.** By asking the user to retype their question.

- **D.** By only searching for the exact keywords provided.

**Answer:** B

**Question 14:** In the context of HyDE, what specifically helps the system find the exact technical instruction document for a ‘blue screen’ query?

- **A.** The presence of technical keywords like ‘BSOD’, ‘driver’, and ‘Safe Mode’ appearing in the LLM’s draft.

- **B.** The user explicitly typing ‘BSOD’.

- **C.** A fallback to exact BM25 matching.

- **D.** The decomposition of the query into three sub-queries.

**Answer:** A

**Question 15:** During Query Decomposition, why is performing a document search for each separate sub-question necessary?

- **A.** To ensure each search has a clear goal and high accuracy.

- **B.** To reduce the number of tokens used.

- **C.** To bypass the vector database entirely.

- **D.** Because the LLM cannot read long documents.

**Answer:** A

**Question 16:** If a user asks to ‘Compare the revenue of iPhone 15 and Samsung S24 in Q1 2024’, why does simple searching fail?

- **A.** The LLM refuses to compare competitors.

- **B.** The query length exceeds the maximum token limit.

- **C.** There is no single document containing this comparison table, information is scattered.

- **D.** The query lacks technical keywords.

**Answer:** C

**Question 17:** What is the ‘Encode’ step’s specific function in the HyDE process?

- **A.** To compress the final response.

- **B.** To pass the hypothetical paragraph through the embedding model to create a vector.

- **C.** To encrypt user data for privacy.

- **D.** To split the text into manageable chunks.

**Answer:** B

**Question 18:** Which technique is best to overcome semantic asymmetry by utilizing the creativity of the LLM?

- **A.** BM25

- **B.** Query Decomposition

- **C.** HNSW

- **D.** HyDE

**Answer:** D

**Question 19:** What ultimate role does Query Transformation play in the overall RAG architecture?

- **A.** It replaces the vector database.

- **B.** It acts as an intelligent editor, editing and reorienting questions before lookup.

- **C.** It generates the final answer displayed to the user.

- **D.** It handles the user interface.

**Answer:** B

**Question 20:** In a complex query scenario, what does the LLM do in the Final Generation stage of Query Decomposition?

- **A.** It receives figures from the searches and self-aggregates them into a complete comparison answer.

- **B.** It asks the user for clarification.

- **C.** It generates a hypothetical document to search with.

- **D.** It runs a Cypher query against a graph database.

**Answer:** A
