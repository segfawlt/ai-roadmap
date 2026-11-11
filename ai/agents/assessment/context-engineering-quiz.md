# Quiz

## Context Engineering

**Question 1:** What is the key distinction between prompt engineering and context engineering?

- **A.** Prompt engineering focuses on model selection; context engineering focuses on temperature settings.

- **B.** Prompt engineering focuses on the instruction text; context engineering treats the entire context window as an engineered artifact.

- **C.** They are the same thing with different names.

- **D.** Context engineering only applies to RAG systems.

**Answer:** B

**Question 2:** In a context budget for a 128k token window, why should you reserve tokens for the response?

- **A.** The model needs tokens for its output; without a reserve, it may truncate its response.

- **B.** Reserved tokens are used for caching.

- **C.** The API requires a minimum response allocation.

- **D.** Response tokens are cheaper than input tokens.

**Answer:** A

**Question 3:** What is “write context” in Karpathy’s framing?

- **A.** Context that the model writes during generation.

- **B.** Information authored and controlled offline — system prompts, few-shot examples, output schemas.

- **C.** The portion of context that gets cached.

- **D.** User messages and conversation history.

**Answer:** B

**Question 4:** What is the “lost-in-the-middle” problem?

- **A.** LLMs forget the system prompt after 10 conversation turns.

- **B.** Information in the middle of the context window receives less attention than information at the beginning or end.

- **C.** Retrieved documents lose relevance after being cached.

- **D.** Token counts are inaccurate for content in the middle of long prompts.

**Answer:** B

**Question 5:** Which truncation strategy is best for conversation history that has grown beyond its token budget?

- **A.** Delete the system prompt to free tokens.

- **B.** Summarize older messages while preserving recent exchanges.

- **C.** Randomly remove messages until under budget.

- **D.** Switch to a model with a larger context window.

**Answer:** B

**Question 6:** How should you structure a context window to maximize cache hit rate?

- **A.** Place dynamic content (user query, tool outputs) first, followed by stable content.

- **B.** Randomly shuffle all components on each request.

- **C.** Place stable content (system prompt, few-shot examples) first, followed by dynamic content.

- **D.** Send each component as a separate API call.

**Answer:** C

**Question 7:** Why does injecting irrelevant retrieved documents increase hallucination?

- **A.** Irrelevant documents consume the model’s attention budget, and the model may treat irrelevant content as factual context.

- **B.** Irrelevant documents cause the API to return errors.

- **C.** The model always ignores irrelevant documents, so it has no effect.

- **D.** Irrelevant documents only affect latency, not quality.

**Answer:** A

**Question 8:** You are building a customer support agent. The retrieved documentation, customer profile, and ticket history together exceed the context budget. What is the correct approach?

- **A.** Remove the system prompt to make room.

- **B.** Apply priority-based truncation: reduce the lowest-priority component first (e.g., limit ticket history to the 3 most recent tickets).

- **C.** Send the request anyway and hope the model handles it.

- **D.** Switch to a model with unlimited context.

**Answer:** B

**Question 9:** What is source attribution in context engineering?

- **A.** Crediting the authors of the LLM training data.

- **B.** Labeling each retrieved document chunk with a source identifier so the model can cite it in responses.

- **C.** Tracking which API key was used for the request.

- **D.** Recording the timestamp of each retrieval.

**Answer:** B

**Question 10:** A production context assembly pipeline should include a “context validation” step. What does this mean?

- **A.** Validating the user’s API key before sending the request.

- **B.** Logging and monitoring the actual content the model receives, to debug quality issues.

- **C.** Running spell-check on the context.

- **D.** Verifying the model’s response before returning it.

**Answer:** B

**Question 11:** Why is “context stuffing” (dumping all available information into the window) an anti-pattern?

- **A.** It is too expensive to send large contexts.

- **B.** It degrades output quality because irrelevant information dilutes the signal and the model may attend to noise instead of the relevant content.

- **C.** LLMs reject requests that are too long.

- **D.** It only affects latency, not quality.

**Answer:** B

**Question 12:** In progressive disclosure, how should a context engineering pipeline handle a complex user request?

- **A.** Retrieve and inject all possible documents upfront.

- **B.** Start with summaries of relevant topics, then fetch detailed content only for the specific aspect the user asks about.

- **C.** Only use the system prompt and skip retrieval entirely.

- **D.** Ask the user to simplify their question.

**Answer:** B
