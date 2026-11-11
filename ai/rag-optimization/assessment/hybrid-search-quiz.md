# Quiz

## Hybrid Search

**Question 1:** What type of query reveals a weakness in pure Vector Search?

- **A.** Queries asking for summaries.

- **B.** Queries requiring absolute accuracy in wording, like proper names or product codes.

- **C.** Queries asking about complex sentiments.

- **D.** Queries written in multiple languages.

**Answer:** B

**Question 2:** What does the BM25 algorithm specialize in finding?

- **A.** Contextual synonyms.

- **B.** Hypothetical vectors.

- **C.** Precise keywords based on frequency statistics.

- **D.** Nodes in a graph database.

**Answer:** C

**Question 3:** BM25 is considered a refined upgrade of which classical information retrieval algorithm?

- **A.** HNSW

- **B.** Cosine Similarity

- **C.** TF-IDF

- **D.** MMR.

**Answer:** C

**Question 4:** What are the two parallel search streams combined in Hybrid Search?

- **A.** Cross-Encoder and Bi-Encoder

- **B.** Semantic Chunking and Recursive Chunking

- **C.** Sparse Retriever (BM25) and Dense Retriever (Vector Search)

- **D.** Neo4j and ChromaDB.

**Answer:** C

**Question 5:** Instead of caring about numerical scores, what does the RRF algorithm rely on?

- **A.** Rank

- **B.** Keyword Density

- **C.** Vector Length

- **D.** Document Chunk Size.

**Answer:** A

**Question 6:** Why might Vector Search miss a query specifically looking for ‘Error 503’?

- **A.** It ignores all numbers.

- **B.** In vector space, these numbers may not carry much specific semantic meaning, causing it to prioritize general ‘error’ synonyms.

- **C.** It automatically rounds numbers to the nearest tenth.

- **D.** It assumes 503 is a typographical error.

**Answer:** B

**Question 7:** How does the ‘TF Saturation’ mechanism in BM25 prevent keyword spamming?

- **A.** It completely ignores keywords that appear more than once.

- **B.** Unlike TF-IDF, after appearing a certain number of times, appearing again hardly adds more score, asymptoting to a limit.

- **C.** It deletes the document from the index if spam is detected.

- **D.** It divides the score by the total word count.

**Answer:** B

**Question 8:** What is the function of the IDF (Inverse Document Frequency) principle in BM25?

- **A.** It counts the total documents in the database.

- **B.** It ensures short paragraphs rank lower.

- **C.** It heavily penalizes common words and greatly rewards rare words.

- **D.** It increases the score infinitely for repeated terms.

**Answer:** C

**Question 9:** How does ‘Length Normalization’ make BM25 smarter?

- **A.** By padding short documents with empty vectors.

- **B.** By rating a keyword occurrence in a short paragraph higher than in a long novel where information is diluted.

- **C.** By only accepting documents of a fixed length.

- **D.** By translating long novels into summaries.

**Answer:** B

**Question 10:** What is the primary problem encountered when trying to fuse results from Vector Search and BM25 directly?

- **A.** They return results in different languages.

- **B.** The scoring scales are completely different (cosine similarity vs statistical formula) and cannot be directly added.

- **C.** Vector Search is too slow to wait for BM25.

- **D.** BM25 cannot run in parallel.

**Answer:** B

**Question 11:** What does the ‘Fusion’ step accomplish in the Hybrid Search Implementation Process?

- **A.** It translates the documents.

- **B.** It merges the two lists (from BM25 and Vector Search) into a single list.

- **C.** It concatenates text using a Cross-Encoder.

- **D.** It expands the query using an LLM.

**Answer:** B

**Question 12:** What is listed as a major ‘Con’ of implementing Hybrid Search?

- **A.** It fails if the user uses synonyms.

- **B.** It is more complex to deploy and consumes more resources due to running 2 parallel streams.

- **C.** It loses the ability to match exact keywords.

- **D.** It cannot handle multi-lingual queries.

**Answer:** B

**Question 13:** In the RRF formula 1 / (k + rank_r(d)), what is the typical value chosen for the smoothing constant k?

- **A.** 1

- **B.** 10

- **C.** 60

- **D.** 100

**Answer:** C

**Question 14:** What specific role does the smoothing constant k play in the Reciprocal Rank Fusion formula?

- **A.** It converts cosine similarity to a percentage.

- **B.** It acts as a hard limit on the number of retrieved documents.

- **C.** It helps reduce score disparity between very high ranks (e.g., Top 1 vs Top 2), ensuring fairness.

- **D.** It prevents BM25 from calculating saturation.

**Answer:** C

**Question 15:** In the RRF illustrative example, why does Doc B (Rank 2 Vector, Rank 3 BM25) beat Doc A (Rank 1 Vector, Rank 10 BM25)?

- **A.** Because BM25 scores are weighted double by default.

- **B.** Because RRF prioritizes documents that have high consensus from both algorithms, yielding a higher combined sum (0.0320 \> 0.0307).

- **C.** Because Doc A triggered the TF saturation penalty.

- **D.** Because Doc B was shorter, gaining a Length Normalization bonus.

**Answer:** B

**Question 16:** What is the fundamental assumption that makes Reciprocal Rank Fusion (RRF) effective?

- **A.** That cosine similarity is always more accurate than statistical frequency.

- **B.** That if a document appears at a high rank in both lists, it is certainly an important document.

- **C.** That keywords in the title are more important than in the body.

- **D.** That sparse retrievers will eventually replace dense retrievers.

**Answer:** B

**Question 17:** If a user searches for ‘symptoms of meningitis’, how does BM25 handle the word ‘of’ compared to ‘meningitis’?

- **A.** It treats them equally.

- **B.** It drops ‘meningitis’ due to complexity.

- **C.** Through IDF, it heavily penalizes ‘of’ as a common word and greatly rewards ‘meningitis’ as a rare word.

- **D.** It triggers length normalization based on the word ‘of’.

**Answer:** C

**Question 18:** In the ‘Galaxy’ keyword spam example, why does TF-IDF fail compared to BM25?

- **A.** TF-IDF cannot process the word ‘Galaxy’.

- **B.** TF-IDF scores increase linearly infinitely with repetition, giving Doc A an overwhelming win, while BM25 saturation stops the score from increasing further.

- **C.** TF-IDF strictly relies on document length normalization.

- **D.** BM25 uses vector similarity to realize it is spam.

**Answer:** B

**Question 19:** Which Search method is described as ‘Poor at matching exact keywords, hard to explain results’?

- **A.** BM25

- **B.** Graph Traversal

- **C.** Vector Search

- **D.** Hybrid Search.

**Answer:** C

**Question 20:** Which Search method ‘Does not understand context, fails if user uses different synonyms than the text’?

- **A.** Vector Search

- **B.** Hybrid Search

- **C.** MMR

- **D.** BM25.

**Answer:** D
