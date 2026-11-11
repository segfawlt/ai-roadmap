# Quiz

## Post-Retrieval Processing

**Question 1:** Why is the Top-K list often not optimal enough to feed directly into the LLM?

- **A.** Because the documents are too short.

- **B.** Due to Limited Semantic Accuracy and Information Noise.

- **C.** Because it contains too many images.

- **D.** Because it bypasses the embedding model completely.

**Answer:** B

**Question 2:** What is the primary role of Re-ranking in the retrieval process?

- **A.** To act as a filter in the final step to select the best documents.

- **B.** To generate hypothetical answers.

- **C.** To split documents into chunks.

- **D.** To combine vector and keyword search.

**Answer:** A

**Question 3:** What architecture is used when a system processes a question and a document separately?

- **A.** Cross-Encoder

- **B.** BM25

- **C.** Bi-Encoder

- **D.** HyDE

**Answer:** C

**Question 4:** What does the MMR algorithm aim to balance?

- **A.** Relevance and Diversity

- **B.** Speed and Cost

- **C.** Latency and Accuracy

- **D.** Keywords and Vectors.

**Answer:** A

**Question 5:** Which encoding method is known for high speed because calculations can be pre-computed?

- **A.** Cross-Encoder

- **B.** Bi-Encoder

- **C.** MMR

- **D.** Semantic Chunking.

**Answer:** B

**Question 6:** What is a major consequence of Embedding models prioritizing retrieval speed on large amounts of data?

- **A.** They consume too much RAM.

- **B.** They are forced to trade off the ability to understand complex semantic relationships.

- **C.** They can only process English text.

- **D.** They require manual keyword tagging.

**Answer:** B

**Question 7:** How does a Cross-Encoder process the question and the document?

- **A.** It converts them into separate numerical IDs.

- **B.** It concatenates them into a single text sequence fed into the model simultaneously.

- **C.** It translates the question before comparing it.

- **D.** It strictly uses term frequency algorithms.

**Answer:** B

**Question 8:** What is a significant disadvantage of using a Cross-Encoder?

- **A.** It cannot handle negation.

- **B.** It is very slow and resource-consuming, cannot be used across the entire database.

- **C.** It only relies on keyword matching.

- **D.** It splits complete ideas into meaningless chunks.

**Answer:** B

**Question 9:** What is the first step in the ‘Funnel Strategy’ for re-ranking?

- **A.** Use Bi-Encoder to quickly get Top 50 documents from millions.

- **B.** Apply MMR to all documents.

- **C.** Use Cross-Encoder to re-score millions of documents.

- **D.** Generate a hypothetical response using HyDE.

**Answer:** A

**Question 10:** In MMR, what is the purpose of the ‘Diversity’ factor?

- **A.** To ensure the document is related to the question.

- **B.** To ensure the document is different from previously selected documents.

- **C.** To ensure multiple languages are included.

- **D.** To increase the retrieval speed.

**Answer:** B

**Question 11:** What does a Bi-Encoder fail to capture that a Cross-Encoder excels at?

- **A.** The length of the document.

- **B.** The detailed interaction information between each word in the question and each word in the document.

- **C.** The exact number of keyword occurrences.

- **D.** The file type of the retrieved document.

**Answer:** B

**Question 12:** When querying ‘What does Python not eat?’, why might a Bi-Encoder return irrelevant results?

- **A.** It cannot process the word ‘Python’.

- **B.** It may search with wrong intent because it only catches keywords and ignores negation.

- **C.** It automatically translates ‘Python’ to a programming language.

- **D.** It requires a Graph Database to function.

**Answer:** B

**Question 13:** According to the Simplified Formula for MMR, what does a smaller lambda value prioritize?

- **A.** Relevance over diversity.

- **B.** Diversity more heavily.

- **C.** Execution speed over accuracy.

- **D.** The Cross-Encoder score.

**Answer:** B

**Question 14:** In the MMR calculation, what does the term ‘max Sim_2(d_i, d_j)’ aim to penalize?

- **A.** Documents that do not match the user’s keywords.

- **B.** Documents that are too long.

- **C.** Documents that are highly similar to documents already selected in the set S.

- **D.** Documents that come from different external sources.

**Answer:** C

**Question 15:** How does a Cross-Encoder understand complex interactions within text?

- **A.** By using keyword density mapping.

- **B.** By relying on a full Self-Attention mechanism to read the concatenated sequence in parallel.

- **C.** By splitting sentences and indexing them in an HNSW graph.

- **D.** By relying purely on BM25 frequency calculations.

**Answer:** B

**Question 16:** Why does Information Noise occur in the initial Retrieval step?

- **A.** Documents may contain matching keywords but deviate in context or true intent.

- **B.** The embedding model compresses text too heavily.

- **C.** The user’s internet connection was unstable.

- **D.** The query was rewritten poorly by HyDE.

**Answer:** A

**Question 17:** If you need extremely accurate answers for difficult questions, which Re-ranking method should you choose?

- **A.** Maximal Marginal Relevance (MMR)

- **B.** Hybrid Search

- **C.** Cross-Encoder

- **D.** Bi-Encoder.

**Answer:** C

**Question 18:** If your goal is to provide a general answer covering many aspects, which Re-ranking method is most appropriate?

- **A.** Cross-Encoder

- **B.** Maximal Marginal Relevance (MMR)

- **C.** Bi-Encoder

- **D.** HyDE.

**Answer:** B

**Question 19:** In the VF8 car example, why is Doc 2 selected when using MMR?

- **A.** Because it has a higher Bi-Encoder score than Doc 1.

- **B.** Because it contains the word ‘VF8’ more frequently.

- **C.** Because its content (ADAS system) differs from Doc 1 (electric motor).

- **D.** Because it is the longest document available.

**Answer:** C

**Question 20:** What specific structural element of the query ‘What does Python not eat?’ is successfully recognized by a Cross-Encoder but often missed by a Bi-Encoder?

- **A.** The biological context and negation structure ‘not eat’.

- **B.** The capital letter ‘P’.

- **C.** The interrogative word ‘What’.

- **D.** The length of the query string.

**Answer:** A
