# Quiz

## RAGAS Evaluation Metrics

**Question 1:** Scenario: The LLM gives a brilliant, factually correct answer based on its pre-trained knowledge, but the retrieved context from your database contained entirely unrelated text. What is the Faithfulness score?

- **A.** 1.0, because the answer is factually true.

- **B.** 0.8, because it ignored the prompt.

- **C.** 0.5, because the context was ignored.

- **D.** 0.0, because none of the statements can be inferred from the retrieved context.

**Answer:** D

**Question 2:** If a user asks ‘What is the capital of Japan?’ and the LLM responds ‘Tokyo is a city in Japan with a large population, famous for cherry blossoms, and it serves as the capital.’, which metric might flag this answer as suboptimal?

- **A.** Faithfulness (due to hallucination).

- **B.** Context Recall (due to missing info).

- **C.** Answer Relevancy (due to redundant/extra information not directly addressing only the prompt).

- **D.** Context Precision (due to bad ranking).

**Answer:** C

**Question 3:** How does Context Precision handle irrelevant chunks that appear high up in the retrieved results (e.g., Position 1)?

- **A.** It significantly penalizes the score because it calculates the ratio of relevant contexts at each top-k position.

- **B.** It ignores them as long as a relevant chunk is at Position 5.

- **C.** It boosts the score to encourage diversity.

- **D.** It forces the LLM to rewrite the context.

**Answer:** A

**Question 4:** Scenario: You have an expert reference answer containing 4 key claims. Your retrieval system pulls contexts that only support 1 of those claims. What is the Context Recall score?

- **A.** 1

- **B.** 0.25 (1/4)

- **C.** 0.5

- **D.** 0

**Answer:** B

**Question 5:** Why does Ragas use a frontier LLM (e.g., Claude Sonnet 4.6 or GPT-5.2) as a ‘Judge’ for its metrics?

- **A.** Because humans are incapable of reading RAG outputs.

- **B.** To automate the evaluation process, minimizing the high costs and time associated with human ground-truth annotation.

- **C.** To generate hypothetical vectors.

- **D.** Because it is required by the Neo4j database.

**Answer:** B

**Question 6:** In the calculation of Answer Relevancy, why are ‘reverse-engineered’ questions generated?

- **A.** To compare their embedding similarity against the original user question; high similarity means the answer directly addressed the prompt.

- **B.** To train a new embedding model.

- **C.** To populate the Graph database.

- **D.** To ask the user for clarification.

**Answer:** A

**Question 7:** Which two Ragas metrics are specifically focused on evaluating the ‘Retrieval’ performance of a RAG system?

- **A.** Faithfulness and Answer Relevancy

- **B.** Answer Correctness and Faithfulness

- **C.** Context Precision and Context Recall

- **D.** Latency and Cost

**Answer:** C

**Question 8:** Which two Ragas metrics are specifically focused on evaluating the ‘Generation’ performance of a RAG system?

- **A.** Faithfulness and Answer Relevancy

- **B.** Context Precision and Context Recall

- **C.** Retrieval Latency and Token Cost

- **D.** Context Recall and Faithfulness

**Answer:** A

**Question 9:** If a RAG system has High Context Recall but Low Context Precision, what does this indicate about the retrieved chunks?

- **A.** It found no useful information.

- **B.** It found all the necessary information, but buried it among a lot of irrelevant noise (poor ranking).

- **C.** It hallucinated the answer.

- **D.** It ranked the exact right answer at position 1, but missed everything else.

**Answer:** B

**Question 10:** What is the first step in the calculation process for Context Recall?

- **A.** Reverse-engineering questions.

- **B.** Calculating cosine similarity.

- **C.** Splitting the ‘reference answer’ (ground truth) into individual sentences/claims.

- **D.** Generating an answer using a frontier LLM.

**Answer:** C

**Question 11:** In the ‘Green Tea’ Context Precision example, why does an irrelevant context at position 2 lower the final score?

- **A.** Because Precision@2 drops to 0.5, pulling down the weighted average for subsequent relevant chunks.

- **B.** Because the LLM deletes the irrelevant chunk.

- **C.** Because it triggers a Faithfulness penalty.

- **D.** Because it changes the user’s original question.

**Answer:** A

**Question 12:** What does it mean if Faithfulness evaluates to exactly 1.0?

- **A.** The answer is 100% factually accurate to the real world.

- **B.** The answer contains exactly 100 words.

- **C.** Every single statement made in the generated answer can be directly supported by the retrieved context.

- **D.** The retrieval process took exactly 1 second.

**Answer:** C

**Question 13:** Why is Answer Relevancy NOT considered a measure of ‘factuality’?

- **A.** Because it uses BM25 instead of vectors.

- **B.** Because it only checks if the answer conceptually aligns with what was asked, not whether the facts stated are true.

- **C.** Because the judge LLM cannot evaluate facts.

- **D.** Because it only measures the speed of the response.

**Answer:** B

**Question 14:** If your RAG system suffers from ‘hallucinations’, which metric will most directly drop?

- **A.** Context Precision

- **B.** Context Recall

- **C.** Answer Relevancy

- **D.** Faithfulness

**Answer:** D

**Question 15:** In the calculation process for Faithfulness, what happens after the answer is decomposed into claims?

- **A.** The claims are translated.

- **B.** The LLM verifies each statement to see if it can be inferred from the context.

- **C.** The claims are stored in Neo4j.

- **D.** The context is deleted.

**Answer:** B

**Question 16:** If a generated answer lacks necessary details requested in the prompt (e.g., asking for location and capital, but only giving location), what happens to Answer Relevancy?

- **A.** It increases because the answer is shorter.

- **B.** It stays the same.

- **C.** It decreases because the reverse-engineered questions will not match the full scope of the original prompt.

- **D.** It forces a re-retrieval.

**Answer:** C
