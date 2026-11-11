# Assignment: LLM Observability with LangFuse & LangSmith

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | LLM Observability Implementation |
| **Course** | LLMOps and Evaluation |
| **Project Name** | `llm-observability-lab` |
| **Estimated Time** | 120 minutes |
| **Framework** | Python 3.11+, LangChain 1.x, LangFuse, LangSmith, OpenAI API |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Configure** LangFuse and LangSmith for LLM application tracing

- **Implement** callback handlers to capture execution flows

- **Track** token usage, latency, and costs per request

- **Debug** LLM chains using trace visualization and playgrounds

- **Apply** production best practices for sampling, PII handling, and alerting

------------------------------------------------------------------------

## Problem Description

You are building a production-ready RAG chatbot application. Without observability, you face:

1.  **Black box execution**: No visibility into retrieval and generation steps

2.  **Cost overruns**: Inability to track spending per user or feature

3.  **Performance issues**: Difficulty identifying latency bottlenecks

4.  **Quality problems**: No systematic way to collect and analyze feedback

Your task is to instrument this application with comprehensive observability.

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `langfuse` \>= 2.0.0

  - `langchain[openai]` \>= 1.0

  - `langsmith` \>= 0.1.0

### Accounts Required

- LangFuse Cloud account (free tier) OR Docker setup for self-hosted

- LangSmith account (free tier available)

- OpenAI API key

------------------------------------------------------------------------

## Tasks

### Task 1: LangFuse Integration (30 points)

1.  **Set up LangFuse** environment:

    - Create a LangFuse Cloud account or deploy locally with Docker

    - Configure API keys and environment variables

    - Verify connectivity

2.  **Implement tracing** for a LangChain application:

    - Create a RAG chain with retrieval and generation steps

    - Add `CallbackHandler` to capture all traces

    - Verify traces appear in the LangFuse dashboard

3.  **Implement cost tracking**:

    - Capture token usage for each LLM call

    - Calculate costs based on model pricing

    - Display cost breakdown per session

4.  **Document**:

    - Screenshot of trace visualization in LangFuse

    - Cost breakdown for at least 10 queries

### Task 2: LangSmith Integration (30 points)

1.  **Configure LangSmith** auto-tracing:

    - Set environment variables for automatic instrumentation

    - Create a project for your application

    - Verify traces are captured

2.  **Build a RAG pipeline** with detailed tracing:

    - Implement document retrieval step

    - Implement LLM generation step

    - Capture intermediate states

3.  **Use the Playground** for debugging:

    - Identify a failed or low-quality response

    - Open the trace in the Playground

    - Modify the prompt and re-run

    - Document the improvement

4.  **Create a test dataset**:

    - Export 5 production traces to a dataset

    - Run evaluation on the dataset

    - Compare results across prompt versions

### Task 3: Comparison Analysis (20 points)

1.  **Compare LangFuse vs LangSmith** based on your experience:

| Feature             | LangFuse | LangSmith | Your Assessment |
|---------------------|----------|-----------|-----------------|
| Setup complexity    |          |           |                 |
| Trace visualization |          |           |                 |
| Cost tracking       |          |           |                 |
| Debugging tools     |          |           |                 |
| Self-hosting option |          |           |                 |

1.  **Write a recommendation** (200-300 words):

    - Which tool would you choose for different scenarios?

    - What are the key trade-offs?

### Task 4: Production Best Practices (20 points)

1.  **Implement sampling**:

    - Configure 100% tracing for development

    - Configure 5% sampling for production simulation

    - Add “High Importance” flag for error traces

2.  **Implement PII handling**:

    - Create a masking function for sensitive data

    - Apply to traces before sending to observability tools

    - Test with sample PII data

3.  **Design an alerting strategy**:

    - Define thresholds for error rate, latency, and cost

    - Document alert rules (pseudo-code or tool configuration)

    - Create a runbook for each alert type

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Source code (Jupyter notebook or Python scripts)

`README.md` with setup and configuration instructions

Screenshots of LangFuse traces and dashboard

Screenshots of LangSmith traces and Playground usage

Comparison analysis document

Production best practices implementation

### Submission Checklist

LangFuse traces are captured and visible

LangSmith auto-tracing is working

Cost tracking is implemented

Playground debugging is demonstrated

Comparison analysis is complete

Production best practices are documented

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria                          | Points  |
|-----------------------------------|---------|
| LangFuse integration & tracing    | 30      |
| LangSmith integration & debugging | 30      |
| Comparison analysis quality       | 20      |
| Production best practices         | 15      |
| Code quality and documentation    | 5       |
| **Total**                         | **100** |

------------------------------------------------------------------------

## Hints

- Start with LangSmith as it requires minimal code changes (just environment variables)

- Use LangFuse’s prompt management for version control of prompts

- When comparing tools, focus on real usage scenarios from your experience

- For PII masking, consider regex patterns for emails, phone numbers, and credit cards

- Set up alerts using webhook integrations or existing monitoring tools
