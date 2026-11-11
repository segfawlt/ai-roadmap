# Final Exam: Production-Ready RAG Evaluation System

## overview

| Field             | Value                 |
|-------------------|-----------------------|
| **Course**        | LLMOps and Evaluation |
| **Duration**      | 240 minutes (4 hours) |
| **Passing Score** | 70%                   |
| **Total Points**  | 100                   |

------------------------------------------------------------------------

## Description

You have been hired as an **MLOps Engineer** at **AI Solutions Corp.**, a company that builds enterprise AI assistants. Your task is to build a **Production-Ready RAG Evaluation System** that combines automated quality assessment, comprehensive observability, and rigorous architecture comparison.

The current system lacks:

- Automated evaluation metrics to measure answer quality

- Observability into LLM execution, costs, and latency

- Data-driven architecture selection based on experiments

You must apply knowledge from **RAGAS Evaluation Metrics**, **LLM Observability (LangFuse/LangSmith)**, and **RAG Architecture Comparison** to build a comprehensive evaluation and monitoring platform.

------------------------------------------------------------------------

## Objectives

By completing this exam, you will demonstrate mastery of:

- Implementing **RAGAS metrics** (Faithfulness, Answer Relevancy, Context Precision, Context Recall)

- Integrating **LangFuse and/or LangSmith** for comprehensive LLM tracing and cost tracking

- Designing and executing **RAG architecture experiments** with scientific rigor

- Building an **end-to-end evaluation pipeline** that combines all three components

- Making **data-driven architecture recommendations** based on experimental results

------------------------------------------------------------------------

## Problem Description

Build a **Production-Ready RAG Evaluation System** named `rag-evaluation-platform` that:

1.  **Evaluates RAG quality** using RAGAS metrics on generated responses

2.  **Traces all LLM operations** with full observability (tokens, costs, latency)

3.  **Compares multiple RAG architectures** systematically

4.  **Produces actionable reports** for architecture selection

The system should serve as a complete toolkit for evaluating, monitoring, and optimizing RAG systems in production.

------------------------------------------------------------------------

## Assumptions

- You have completed the assignments on RAGAS, Observability, and Experiment Comparison

- OpenAI API key or compatible LLM endpoint is available

- LangFuse Cloud account OR local Docker setup for self-hosted LangFuse

- LangSmith account (free tier)

- Python 3.11+ environment with necessary packages installed

- Sample documents and test questions are provided or created

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.11 or higher

- Required packages:

  - `ragas` \>= 0.2

  - `langfuse` \>= 2.0.0

  - `langchain[openai]` \>= 1.0

  - `langsmith` \>= 0.1.0

  - `chromadb` \>= 0.4.0 OR `qdrant-client` \>= 1.7.0

  - `sentence-transformers` \>= 2.2.0

  - `pandas` \>= 2.0.0

  - `matplotlib` \>= 3.7.0

### Infrastructure

- Vector Database: ChromaDB or Qdrant

- Observability: LangFuse (required) + LangSmith (optional)

- Embedding Model: `text-embedding-3-small` or equivalent

- LLM: A frontier model such as Claude Sonnet 4.6, GPT-5.2, or Gemini 3

------------------------------------------------------------------------

## Tasks

### Task 1: RAGAS Evaluation Pipeline (25 points)

**Time Allocation**: 60 minutes

Build a comprehensive evaluation pipeline using all four RAGAS metrics.

#### Requirements:

1.  **Implement RAGAS Evaluation Module**

    - Create functions to calculate Faithfulness, Answer Relevancy, Context Precision, and Context Recall

    - Support batch evaluation on datasets

    - Handle edge cases (empty contexts, very short answers)

2.  **Create Evaluation Dataset**

    - Prepare at least 30 test questions with ground truth answers

    - Categorize questions: Factual (40%), Relational (30%), Multi-hop (20%), Analytical (10%)

    - Include retrieved contexts for each question

3.  **Run Evaluation**

    - Execute evaluation on the complete dataset

    - Calculate aggregate statistics (mean, std, min, max)

    - Identify failure cases (scores \< 0.5)

#### Deliverables:

- `evaluation/ragas_evaluator.py` - Core evaluation logic

- `evaluation/dataset.py` - Dataset loading and preparation

- `data/test_questions.json` - Test dataset with ground truth

------------------------------------------------------------------------

### Task 2: LLM Observability Integration (25 points)

**Time Allocation**: 60 minutes

Implement comprehensive tracing and monitoring for all LLM operations.

#### Requirements:

1.  **LangFuse Integration**

    - Configure LangFuse SDK with proper authentication

    - Implement `CallbackHandler` for all LangChain operations

    - Capture: input/output, token counts, latency, costs

2.  **Cost Tracking Dashboard**

    - Track token usage per query

    - Calculate costs based on model pricing

    - Generate cost breakdown reports

3.  **Production Best Practices**

    - Implement configurable sampling (100% dev, 5% prod)

    - Add PII masking for sensitive data

    - Create correlation IDs for request tracking

4.  **(Bonus) LangSmith Integration**

    - Configure auto-tracing via environment variables

    - Demonstrate Playground debugging for a failed trace

#### Deliverables:

- `observability/langfuse_handler.py` - LangFuse integration

- `observability/cost_tracker.py` - Cost calculation logic

- `observability/pii_masker.py` - PII handling

- Screenshots of LangFuse dashboard with traces

------------------------------------------------------------------------

### Task 3: RAG Architecture Comparison (25 points)

**Time Allocation**: 60 minutes

Design and execute a rigorous experiment comparing multiple RAG architectures.

#### Requirements:

1.  **Implement Two RAG Architectures**

    - **Naive RAG**: Fixed chunking, Top-K retrieval, direct generation

    - **Advanced RAG**: Semantic chunking, hybrid search, re-ranking

2.  **Run Comparative Experiments**

    - Execute both architectures on the same test dataset

    - Capture all RAGAS metrics for each architecture

    - Track latency and cost per query

3.  **Performance Analysis**

    - Break down performance by question category

    - Calculate statistical significance of differences

    - Create visualizations (bar charts, tables)

#### Deliverables:

- `architectures/naive_rag.py` - Naive RAG implementation

- `architectures/advanced_rag.py` - Advanced RAG implementation

- `experiments/runner.py` - Experiment execution

- `results/comparison_table.md` - Results summary

------------------------------------------------------------------------

### Task 4: Integrated Evaluation Platform (25 points)

**Time Allocation**: 60 minutes

Combine all components into a unified evaluation platform.

#### Requirements:

1.  **End-to-End Pipeline**

    - Single entry point to run complete evaluation

    - Automatic tracing of all operations

    - Configurable architecture selection

2.  **Comprehensive Reporting**

    - Generate evaluation report with all metrics

    - Include observability insights (cost, latency distribution)

    - Architecture comparison summary

    - Actionable recommendations

3.  **CLI Interface**

    ``` bash
    python evaluate.py --architecture naive --dataset data/test.json --output results/
    python evaluate.py --architecture advanced --dataset data/test.json --output results/
    python compare.py --results-dir results/ --output comparison_report.md
    ```

4.  **Answer Key Questions**

    - Which architecture should be used for production and why?

    - What is the cost-quality trade-off between architectures?

    - What are the top 3 failure patterns and how to address them?

#### Deliverables:

- `evaluate.py` - Main evaluation script

- `compare.py` - Architecture comparison script

- `reports/evaluation_report.md` - Complete evaluation report

- `ANSWERS.md` - Written responses to key questions

------------------------------------------------------------------------

## Questions to Answer

Include written responses to these questions in `ANSWERS.md`:

1.  **RAGAS Interpretation**: Analyze your Faithfulness and Answer Relevancy scores. What do low scores indicate about your RAG system, and how would you improve them?

2.  **Observability Value**: How did LangFuse/LangSmith tracing help you identify issues in your RAG pipeline? Provide a specific example.

3.  **Architecture Decision**: Based on your experiments, which RAG architecture would you recommend for a customer support chatbot vs. a legal document Q&A system? Justify with data.

4.  **Cost Optimization**: If you had to reduce costs by 50% while maintaining 90% of quality, what strategies would you employ? Reference your experimental results.

5.  **Production Readiness**: What additional monitoring, alerting, or evaluation would you add before deploying this system to production?

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

Complete source code organized in the specified directory structure

`README.md` with:

- Setup instructions (dependencies, API keys, observability setup)

- Usage examples for CLI commands

- Architecture diagram of the evaluation platform

`ANSWERS.md` with written responses to the 5 questions

Test dataset with at least 30 categorized questions

Results tables and visualizations

Screenshots of observability dashboards

### Submission Checklist

All code runs without errors

RAGAS evaluation produces valid scores for all metrics

LangFuse traces are captured and visible in dashboard

Both RAG architectures are implemented and evaluated

Comparison report includes statistical analysis

All questions answered with data-backed reasoning

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria | Weight | Excellent (90-100%) | Good (70-89%) | Needs Improvement (50-69%) | Unsatisfactory (\<50%) |
|----|----|----|----|----|----|
| **RAGAS Evaluation** | 25% | All 4 metrics implemented correctly; comprehensive dataset; insightful failure analysis | Metrics implemented; adequate dataset; basic analysis | Partial metrics; small dataset; minimal analysis | Missing metrics; no dataset |
| **Observability** | 25% | Full LangFuse integration; cost tracking; PII handling; production best practices | LangFuse working; basic cost tracking; some best practices | Partial tracing; no cost tracking | No observability integration |
| **Architecture Comparison** | 25% | Both architectures implemented; rigorous experiments; statistical analysis; visualizations | Both architectures; experiments run; basic comparison | One architecture; limited experiments | No architecture comparison |
| **Integration & Reporting** | 15% | Seamless pipeline; comprehensive reports; CLI interface; actionable insights | Components integrated; adequate reports | Partial integration; basic reports | Components not connected |
| **Code Quality & Documentation** | 10% | Clean code; comprehensive docs; clear README; well-organized | Readable code; adequate docs | Messy code; minimal docs | Poor quality; no docs |

------------------------------------------------------------------------

## Estimated Time

| Task                                   | Time Allocation           |
|----------------------------------------|---------------------------|
| Task 1: RAGAS Evaluation Pipeline      | 60 minutes                |
| Task 2: LLM Observability Integration  | 60 minutes                |
| Task 3: RAG Architecture Comparison    | 60 minutes                |
| Task 4: Integrated Evaluation Platform | 60 minutes                |
| **Total**                              | **240 minutes (4 hours)** |

------------------------------------------------------------------------

## Hints

**Task 1 - RAGAS**:

- Use the companion notebook `10_RAG_Evaluation_with_Ragas.ipynb` as a reference

- Start with a small dataset (10 questions) to verify your pipeline before scaling up

- For claim decomposition in Faithfulness, pick a strong judge model (e.g., Claude Opus 4.6 or GPT-5.2) for higher accuracy

**Task 2 - Observability**:

- Set up LangFuse first since it requires explicit callback handlers (good for understanding)

- Use environment variables to switch between dev (100% tracing) and prod (5% sampling) modes

- Test PII masking with fake data before using real sensitive information

**Task 3 - Experiments**:

- Use the same embedding model for both architectures to ensure fair comparison

- Run each query multiple times if measuring latency to account for variance

- Calculate confidence intervals when comparing metric differences

**Task 4 - Integration**:

- Use Python’s `argparse` or `click` library for CLI implementation

- Generate markdown reports that can be easily shared with stakeholders

- Include both quantitative metrics and qualitative insights in recommendations
