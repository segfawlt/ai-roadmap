# Assignment: Context Engineering

## Assignment Metadata

| Field               | Description                                       |
|---------------------|---------------------------------------------------|
| **Assignment Name** | Context Engineering for Production RAG            |
| **Course**          | AI Integration                                    |
| **Project Name**    | `context-engineering-lab`                         |
| **Estimated Time**  | 120 minutes                                       |
| **Framework**       | Python 3.11+, LangChain 1.x, OpenAI API, tiktoken |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Design** a token budget for a multi-component context window

- **Implement** a context assembly pipeline with retrieval and tool outputs

- **Build** a conversation history manager with summarization-based truncation

- **Optimize** context layout for caching and grounding

- **Measure** how context allocation affects output quality

------------------------------------------------------------------------

## Problem Description

You are building a technical support agent that needs to assemble context from multiple sources — retrieved documentation, customer profiles, ticket history, and conversation history — within a fixed token budget. Your task is to implement the context assembly pipeline and measure how different budget allocations affect response quality.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langchain[openai]` \>= 1.0

  - `tiktoken` \>= 0.7.0

### Dataset

Prepare a test set with:

- At least 10 support questions spanning different categories

- A mock documentation corpus (at least 20 chunks)

- Mock customer profiles and ticket histories

------------------------------------------------------------------------

## Tasks

### Task 1: Token Budget Design (20 points)

1.  **Define a token budget** for a 128k context window:

    - Allocate tokens across: system prompt, few-shot examples, retrieved docs, tool outputs, conversation history, and response reserve

    - Justify your allocation decisions

2.  **Implement a budget tracker** that:

    - Counts tokens per component using tiktoken

    - Warns when a component exceeds its allocation

    - Reports total utilization as a percentage

3.  **Test your budget** with at least 3 scenarios: minimal context, typical context, and maximum context

### Task 2: Dynamic Context Assembly (25 points)

1.  **Build an assembly pipeline** that:

    - Retrieves relevant documentation chunks (top-5 by relevance)

    - Fetches customer profile data

    - Injects recent ticket history

    - Combines all components into a single prompt

2.  **Implement relevance filtering** that:

    - Scores retrieved documents by relevance

    - Drops documents below a similarity threshold

    - Re-ranks remaining documents (most relevant first and last)

3.  **Add source attribution** so the model can cite `[Source`` ``N]` in its responses

### Task 3: Conversation History Management (25 points)

1.  **Implement a history manager** that:

    - Tracks token usage for conversation history

    - Triggers summarization when history exceeds its budget

    - Always preserves the system message and last 3 exchanges

2.  **Compare two truncation strategies**:

    - Simple drop: remove oldest messages

    - Summarize and compress: use an LLM to summarize old messages

3.  **Measure the impact** on response coherence across a 10-turn conversation

### Task 4: Caching and Grounding (30 points)

1.  **Restructure your context** for cache efficiency:

    - Identify which components are stable vs. dynamic

    - Place stable components first in the context

    - Measure cache hit rate across 10 sequential requests

2.  **Implement grounding checks**:

    - Add instructions that require source citation

    - Verify that responses reference provided context

    - Measure faithfulness (% of claims supported by context)

3.  **Run an experiment** varying retrieved context budget from 2,000 to 20,000 tokens:

    - Measure response quality (LLM-as-judge, 1–5 scale)

    - Measure faithfulness score

    - Plot the results and identify the optimal allocation

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code (Python scripts or Jupyter notebook)

`README.md` with setup and usage instructions

Token budget design document with justifications

Experiment results with charts

Analysis report (findings, recommendations)

### Submission Checklist

All code runs without errors

Token budget tracking works correctly

Context assembly produces valid prompts

History management handles overflow gracefully

Experiment results are reproducible

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                             | Points  |
|--------------------------------------|---------|
| Token budget design & implementation | 20      |
| Dynamic context assembly             | 25      |
| Conversation history management      | 25      |
| Caching and grounding experiments    | 30      |
| **Total**                            | **100** |

------------------------------------------------------------------------

## Hints

> **Tip**
>
> Tip
>
> - Use `tiktoken.encoding_for_model("gpt-4o")` for accurate token counting
>
> - Start with a simple budget and iterate — don’t over-optimize on the first pass
>
> - For the grounding experiment, use a simple LLM-as-judge prompt to score faithfulness
>
> - The lost-in-the-middle effect is real — test with most-relevant-first vs. random ordering
>
> - Log the actual tokens per component for each request so you can debug budget violations
