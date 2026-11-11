# Quiz

## Advanced Indexing

**Question 1:** What characterizes ‘Fixed-size Chunking’?

- **A.** Cuts based on changes in content meaning.

- **B.** Text is cut mechanically every 500 or 1000 characters, regardless of sentence completion.

- **C.** Uses an LLM to rewrite the text.

- **D.** Relies on BM25 for text division.

**Answer:** B

**Question 2:** What happens when search utilizes ‘Flat Indexing’?

- **A.** The system calculates the similarity of consecutive sentences.

- **B.** The system has to compare the question with every single vector in a long list.

- **C.** Data points are structured hierarchically.

- **D.** The semantic flow of text is perfectly preserved.

**Answer:** B

**Question 3:** What does the abbreviation ANN stand for in vector searching?

- **A.** Artificial Neural Network

- **B.** Advanced Node Navigation

- **C.** Approximate Nearest Neighbor

- **D.** Algorithm for Numerical Nodes.

**Answer:** C

**Question 4:** What determines the split points in the Semantic Chunking method?

- **A.** A fixed character limit.

- **B.** Understanding the content to decide split points.

- **C.** The number of punctuation marks.

- **D.** Random allocation by the system.

**Answer:** B

**Question 5:** What is the first step in the Semantic Chunking implementation process?

- **A.** Thresholding

- **B.** Similarity Calculation

- **C.** Generating hypothetical documents.

- **D.** Sentence Splitting: Split the text into complete sentences based on punctuation.

**Answer:** D

**Question 6:** Why does ‘Loss of Semantics’ occur with mechanical chunking?

- **A.** Because embedding models ignore large chunks.

- **B.** Because mechanical chunking accidentally breaks the flow of text, splitting complete ideas into meaningless chunks.

- **C.** Because it uses the HNSW algorithm.

- **D.** Because it relies heavily on BM25.

**Answer:** B

**Question 7:** How does Semantic Chunking detect a change in topic between sentences?

- **A.** By counting the characters.

- **B.** When vectors abruptly change direction, creating a large distance.

- **C.** By checking for new paragraph formatting.

- **D.** By querying a Graph Database.

**Answer:** B

**Question 8:** What happens in Semantic Chunking if the similarity between a sentence and the next one is high?

- **A.** The system generates a Cypher query.

- **B.** The topic has changed, so a new chunk starts.

- **C.** Two sentences are on the same topic and are merged into one chunk.

- **D.** The algorithm performs a brute-force search.

**Answer:** C

**Question 9:** What is one of the main cons of Semantic Chunking compared to Recursive Chunking?

- **A.** It easily cuts through important ideas.

- **B.** It creates noisy context.

- **C.** It consumes computational resources due to running a model to compare each sentence.

- **D.** It is only suitable for strictly structured documents like laws.

**Answer:** C

**Question 10:** What is the structure of data in the HNSW algorithm?

- **A.** A single long list.

- **B.** A flat SQL table.

- **C.** A multi-layered graph structure.

- **D.** A recursive binary tree.

**Answer:** C

**Question 11:** In the HNSW graph, what does Layer 0 contain?

- **A.** Only sparse shortcut links.

- **B.** The most recently added data only.

- **C.** All data points, and the most detailed links between them.

- **D.** Only vectors that match exactly.

**Answer:** C

**Question 12:** What role do the ‘higher layers’ serve in an HNSW graph?

- **A.** They contain the full detailed index.

- **B.** They hold backup data for failure recovery.

- **C.** They act as shortcuts helping the algorithm move quickly through the large data space.

- **D.** They are used exclusively for BM25 processing.

**Answer:** C

**Question 13:** What does the parameter ‘M’ control in HNSW, and what is the trade-off of increasing it?

- **A.** Controls search depth, increases speed but lowers accuracy.

- **B.** Specifies the maximum number of links per node, increases accuracy but RAM consumption increases significantly.

- **C.** Controls document chunk size, improves context but slows embedding.

- **D.** Specifies the threshold for semantic chunking, diversity.

**Answer:** B

**Question 14:** How does the parameter ‘ef_construction’ affect index construction?

- **A.** It sets the maximum token length for documents.

- **B.** A larger value makes the algorithm scan more candidates to find optimal links, ensuring high structure quality but slower loading.

- **C.** It limits the number of nodes at Layer 0 to save RAM.

- **D.** It determines how heavily rare words are penalized.

**Answer:** B

**Question 15:** If you are configuring HNSW for a Real-time Chatbot Application, what is the recommended practical strategy?

- **A.** Set M to 64 and ef_search to 500.

- **B.** Disable HNSW and use brute-force.

- **C.** Keep ‘ef_search’ at a low level (e.g., 50 - 100) to optimize latency, accepting a small margin of error.

- **D.** Set ef_construction to 0 to bypass indexing.

**Answer:** C

**Question 16:** What describes the search process transition between layers in HNSW?

- **A.** It scans all nodes on every layer sequentially.

- **B.** It moves to the nearest neighbor until a local extremum is reached, which becomes the entry point for the next layer down.

- **C.** It calculates the cosine similarity for all nodes on Layer 0 first.

- **D.** It performs an exact BM25 keyword match at the highest layer.

**Answer:** B

**Question 17:** In the context of evaluating Chunking Methods, why is Recursive Chunking considered ‘suitable for documents with clear structure’?

- **A.** Because it relies heavily on LLM semantic analysis.

- **B.** Because it cuts based on punctuation and a fixed number of characters, which aligns well with laws and contracts.

- **C.** Because it is the only method that supports HNSW.

- **D.** Because it automatically merges sentences on the same topic.

**Answer:** B

**Question 18:** What is the specific risk of decreasing the ‘ef_search’ parameter?

- **A.** The RAM consumption will skyrocket.

- **B.** The system will crash during index building.

- **C.** The system responds extremely fast, but there is a risk of returning results not closest to the question.

- **D.** It will trigger an endless loop in the top layer.

**Answer:** C

**Question 19:** Why did Semantic Chunking successfully keep all ‘NVIDIA H100 GPU’ information intact in the visual example, unlike Recursive Chunking?

- **A.** Because the word count was exactly 500 characters.

- **B.** Because it detected the semantic break point between H100 and Llama-3 and cut at the right time.

- **C.** Because H100 was designated as a keyword.

- **D.** Because it bypassed the punctuation rules entirely.

**Answer:** B

**Question 20:** What is an explicit consequence of a ‘brute-force’ approach when a system scales up many times over?

- **A.** The vectors lose their semantic meaning completely.

- **B.** Sequentially scanning through millions of vectors is too slow to meet real-time requirements.

- **C.** The LLM hallucinates technical terms.

- **D.** BM25 saturation limits are bypassed.

**Answer:** B
