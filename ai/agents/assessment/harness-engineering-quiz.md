# Quiz

## Harness Engineering

**Question 1:** What is harness engineering?

- **A.** A framework for fine-tuning LLM weights.

- **B.** The practice of shaping the environment around AI agents so they can work reliably — including context, constraints, specs, evaluation, and runtime.

- **C.** A testing methodology for web applications.

- **D.** The process of building evaluation benchmarks for language models.

**Answer:** B

**Question 2:** Why are agent performance gaps “often harness problems rather than model problems”?

- **A.** Because models are always perfect and never make mistakes.

- **B.** Because infrastructure choices — context management, tool design, safety constraints — shape agent behavior as much as the model’s capabilities.

- **C.** Because harnesses replace the need for a good model.

- **D.** Because harness engineering only applies to weak models.

**Answer:** B

**Question 3:** What is context condensation in the context of long-running agents?

- **A.** Deleting the entire conversation history when it gets too long.

- **B.** Compressing completed subtasks and verbose outputs into summaries to free context budget while preserving key decisions.

- **C.** Increasing the model’s context window size.

- **D.** Splitting the context across multiple model calls without any summarization.

**Answer:** B

**Question 4:** An agent attempts to run `rm`` ``-rf`` ``/`. The harness blocks the action. Which harness component is responsible?

- **A.** Context management

- **B.** Specification files

- **C.** Constraints and safety — specifically tool validation against blocked commands

- **D.** Evaluation and observability

**Answer:** C

**Question 5:** What is the purpose of a scratchpad in agent harness engineering?

- **A.** A temporary variable in Python.

- **B.** An external file where the agent persists key decisions, findings, and progress so it can resume after context overflow or interruption.

- **C.** A logging destination for error messages.

- **D.** A UI element for the user to write notes.

**Answer:** B

**Question 6:** What should a `CLAUDE.md` specification file contain?

- **A.** The model’s training data and hyperparameters.

- **B.** Build commands, file conventions, constraints on agent behavior, and examples of desired output.

- **C.** Only the project’s README content.

- **D.** A list of all files in the repository.

**Answer:** B

**Question 7:** Why do agent benchmarks (like SWE-bench) depend heavily on the harness?

- **A.** Because benchmarks measure harness latency, not model quality.

- **B.** Because the same model scores differently with different tool sets, context strategies, and retry policies — so you are comparing harnesses as much as models.

- **C.** Because benchmarks are designed by harness vendors.

- **D.** Because benchmarks ignore the model entirely.

**Answer:** B

**Question 8:** When should a harness require human-in-the-loop approval?

- **A.** For every single tool call, to maximize safety.

- **B.** For sensitive or hard-to-reverse actions like file deletion, git operations, sending messages, or modifying infrastructure.

- **C.** Never — autonomous agents should operate without human intervention.

- **D.** Only when the model explicitly requests it.

**Answer:** B

**Question 9:** What is trajectory-level evaluation for agents?

- **A.** Evaluating only the final output of the agent.

- **B.** Scoring the entire sequence of actions — including efficiency, safety violations, and whether intermediate steps contributed to the goal.

- **C.** Measuring the model’s training loss curve.

- **D.** Counting the number of API calls.

**Answer:** B

**Question 10:** An agent working on a 50-file refactor runs out of context at step 30. What harness improvement would help?

- **A.** Switch to a larger model.

- **B.** Implement checkpointing and context condensation so the agent can persist progress to disk, compress old context, and resume.

- **C.** Reduce the number of files to refactor.

- **D.** Remove the system prompt to free tokens.

**Answer:** B

**Question 11:** What is the difference between a sandbox and a human-in-the-loop gate?

- **A.** They are the same thing.

- **B.** A sandbox isolates the agent’s execution environment (e.g., Docker, worktree) to limit blast radius; a gate pauses execution to ask a human for approval before a specific action.

- **C.** A sandbox is for production; a gate is for development.

- **D.** A sandbox blocks all actions; a gate allows all actions.

**Answer:** B

**Question 12:** Which metric best indicates that an agent is “wandering” rather than making progress?

- **A.** Total token usage

- **B.** Steps per task — a high step count relative to task complexity suggests the agent is retrying, looping, or taking unnecessary actions.

- **C.** Task completion rate

- **D.** Model temperature setting

**Answer:** B

**Question 13:** In multi-agent coordination, what role does shared state play?

- **A.** It replaces the need for individual agent context windows.

- **B.** It allows agents to exchange work products (plans, code, reviews) through files, databases, or message queues without needing to share context windows.

- **C.** It synchronizes model weights between agents.

- **D.** It stores the conversation history for all agents in one place.

**Answer:** B

**Question 14:** What is the recommended approach when a harness discovers a new failure mode?

- **A.** Retrain the model to handle the failure.

- **B.** Tighten the relevant constraint, add a test case, and update the spec — the harness evolves with the agent as you discover failure modes.

- **C.** Switch to a different model.

- **D.** Ignore it if it only happens rarely.

**Answer:** B
