# Evaluation Toolkit - Ragas

To be able to evaluate a RAG system, we need a specialized evaluation toolkit. One of the chosen candidates is RAGAS, with the corresponding research work ‘Ragas: Automated Evaluation of Retrieval Augmented Generation’ in 2024.

Ragas is an automated evaluation framework designed specifically for RAG systems, helping to measure the quality of both main components: retrieval and generation. Unlike traditional evaluation methods requiring ground truth annotations created by humans, Ragas uses frontier large language models (such as Claude Sonnet 4.6, GPT-5.2, or Gemini 3) as judges to automate the evaluation process, minimizing necessary costs and time.

The framework operates on the principle of multi-dimensional evaluation, where each aspect of the RAG system is measured through separate metrics. The four main metrics used in this document include faithfulness, answer relevancy, context precision, and context recall.

**1. Faithfulness - Measuring Faithfulness**

The Faithfulness metric evaluates the truthfulness of the answer compared to the retrieved context, ensuring no hallucination phenomena. An answer is considered faithful if all statements in it can be supported by the retrieved context.

**Calculation Process:**

1.  **Decomposition**: Use LLM to split the answer into individual statements (claims).

2.  **Verification**: Check each statement to see if it can be inferred from the context.

3.  **Scoring**: Apply the formula to calculate the ratio of correct statements.

**Illustrative Example - Faithfulness**

``` mermaid
        graph TD
    Q["Question: 'Where and when was Einstein born?'"]
    C["Context: '...born 14 March 1879...<br>German-born physicist...'"]
    A["Answer: 'Einstein was born in Germany<br>on 20 March 1879.'"]
    A -->|"LLM decompose"| S1["Statement 1:<br>'born in Germany' ✓<br>(supported by context)"]
    A -->|"LLM decompose"| S2["Statement 2:<br>'born on 20 March 1879' ✗<br>(context says 14 March)"]
    S1 & S2 --> F["Faithfulness = 1/2 = 0.5"]
```

**Question**:

Where and when was Einstein born?

**Context**: Albert Einstein (born 14 March 1879) was a German-born theoretical physicist, widely held to be one of the greatest and most influential scientists of all time.

**Answer**:

Einstein was born in Germany on 20 March 1879.

**Analysis**: LLM splits the answer into two statements:

- Statement 1: ‘Einstein was born in Germany.’: Correct, can be inferred from context (‘German-born’).

- Statement 2: ‘Einstein was born on 20 March 1879.’: Incorrect, context says 14 March 1879 not 20 March 1879.

**Result**: Faithfulness = 1/2 = 0.5 because only one of the two statements can be verified from the context.

**2. Answer Relevancy - Measuring Relevance**

The Answer Relevancy metric evaluates the relevance between the answer and the original question, aiming to confirm whether the answer addresses the problem asked. This metric does not evaluate true-false in the sense of factuality, but focuses on completeness and avoiding redundant information.

where `\(E_{g_i}\)` is the embedding of the `\(i\)`-th question generated from the answer, `\(E_o\)` is the embedding of the original question, and `\(N\)` is the number of generated questions.

**Calculation Process:**

1.  **Reverse-engineer**: Ask LLM to generate `\(N\)` different questions from the given answer.

2.  **Embedding**: Convert the original question and generated questions into embedding vectors.

3.  **Similarity Calculation**: Calculate the average cosine similarity between the original question and the generated questions.

**Illustrative Example - Answer Relevancy**

**Question**:

Where is France and what is its capital?

**Low relevance answer**:

France is in western Europe.

**High relevance answer**:

France is in western Europe and Paris is its capital.

**Low relevance answer analysis**: LLM might generate questions like ‘Where is France located?’ or ‘In which part of Europe is France situated?’. These questions only partially match the original question because of missing information about the capital.

**High relevance answer analysis**: LLM might generate the question ‘Where is France and what is its capital?’ matching the original question, leading to higher cosine similarity.

**Result**: The complete answer has an Answer Relevancy score near 1, while the incomplete answer has a significantly lower score.

**3. Context Precision - Measuring Retrieval Accuracy**

The Context Precision metric measures the accuracy of the retrieval process by assessing the ranking of contexts. This metric checks if relevant chunks are ranked high in the list of retrieved contexts.

where `\(K\)` is the total number of chunks in retrieved contexts and `\(v_k \in \{ 0, 1 \}\)` is the relevance indicator at position `\(k\)`.

**Calculation Process:**

1.  **Determine Relevance**: Use LLM to evaluate if each context is relevant to the question.

2.  **Calculate Precision@k**: For each position `\(k\)`, calculate the ratio of relevant contexts in the top `\(k\)`.

3.  **Weighted Average**: Calculate the weighted average of Precision@k, counting only for positions with relevant contexts.

**Illustrative Example - Context Precision**

**Question**:

What are the health benefits of green tea?

**Retrieved contexts in order**:

1.  Green tea contains antioxidants that may reduce cancer risk. - Relevant

2.  Tea plantations are common in Asia, especially China and India. - Irrelevant

3.  Green tea can boost metabolism and aid weight loss. - Relevant

4.  The history of tea dates back thousands of years. - Irrelevant

5.  Green tea improves brain function and mental alertness. - Relevant

**Calculation**:

- Precision@1 = 1/1 = 1.0, `\(v_1 = 1\)`

- Precision@2 = 1/2 = 0.5, `\(v_2 = 0\)`

{/*formula-not-decoded*/}

- Precision@4 = 2/4 = 0.5, `\(v_4 = 0\)`

- Precision@5 = 3/5 = 0.6, `\(v_5 = 1\)`

**Result**: Context Precision = (1.0 × 1 + 0.67 × 1 + 0.6 × 1) / 3 = 2.27 / 3 ≈ 0.76. The score reflects that there are irrelevant contexts interspersed between useful contexts.

**4. Context Recall - Measuring Retrieval Coverage**

The Context Recall metric evaluates the coverage of the retrieval process, measuring how much necessary information from the reference answer was found in the retrieved contexts. Formula:

**Calculation Process:**

1.  **Decomposition**: Split the reference answer into individual sentences/claims.

2.  **Attribution**: Use LLM to check if each claim can be inferred from retrieved contexts.

3.  **Ratio Calculation**: Calculate the ratio of claims supported by contexts over total claims.

**Illustrative Example - Context Recall**

**Question**:

Where is the Eiffel Tower located?

**Reference answer**:

The Eiffel Tower is located in Paris.

**Retrieved contexts**:

Paris is the capital of France.

**Analysis**: Reference answer contains the main claim: ‘The Eiffel Tower is located in Paris.’ However, retrieved context only provides information ‘Paris is the capital of France’ without mentioning the location of the Eiffel Tower. Therefore, LLM cannot infer the claim from the reference based on the existing context.

**Result**: Context Recall = 0/1 = 0, indicating the retriever failed to find context containing necessary information to answer the question.

``` mermaid
        graph LR
    Q[Question] --> RF[Ragas Evaluation Framework]
    GA[Generated Answer] --> RF
    RC[Retrieved Contexts] --> RF
    REF[Reference Answer] --> RF

    subgraph "Generation Metrics"
        RF --> F["Faithfulness<br>(Score 0–1)"]
        RF --> AR["Answer Relevancy<br>(Score 0–1)"]
    end
    subgraph "Retrieval Metrics"
        RF --> CP["Context Precision<br>(Score 0–1)"]
        RF --> CR["Context Recall<br>(Score 0–1)"]
    end
```

*Figure 6: Illustration for metrics of Ragas evaluation tool.*

Each metric gives a value from 0 to 1, with higher values indicating better quality. These four metrics complement each other: faithfulness and answer relevancy evaluate generation quality, while context precision and context recall evaluate retrieval performance.

## Quick Reference Table

| Metric | Measures | Range | What a low score means |
|----|----|----|----|
| **Faithfulness** | Are the answer’s claims supported by the retrieved context? | 0–1 | The generator is hallucinating. |
| **Answer Relevancy** | Does the answer actually address the question? | 0–1 | The answer is off-topic or incomplete. |
| **Context Precision** | Are relevant chunks ranked high in the retrieved set? | 0–1 | Retrieval is noisy or poorly ranked. |
| **Context Recall** | Does the retrieved set contain all information needed? | 0–1 | Retrieval is missing relevant documents. |

Think of it as a 2×2: **Faithfulness/Answer Relevancy** evaluate the **generator**; **Context Precision/Recall** evaluate the **retriever**.

A healthy RAG system typically targets ≥0.85 on all four. Below 0.7 on any one metric is a smell worth investigating.

## Beyond RAGAS: The Evaluation Ecosystem in 2026 NEW

RAGAS is the de facto standard for RAG-specific metrics, but production teams increasingly combine it with other tools that cover agents, safety, observability, and CI/CD integration. The ecosystem has matured rapidly; here is what matters now.

### DeepEval

The fastest-growing evaluation framework with 50+ metrics and native pytest integration:

- Covers RAG, agents, multi-turn conversations, safety, and multimodal evaluation

- `pytest`-style test writing keeps evals close to the rest of the test suite: `assert_test(test_case,`` ``[FaithfulnessMetric()])`

- DAG-based metric evaluation shares intermediate LLM outputs across metrics, reducing LLM judge calls by ~40%

- Best for: engineering teams integrating evaluation into CI/CD pipelines

``` python
from deepeval import assert_test
from deepeval.test_case import LLMTestCase
from deepeval.metrics import FaithfulnessMetric

def test_rag_faithfulness():
    test_case = LLMTestCase(
        input="When was the Eiffel Tower built?",
        actual_output="The Eiffel Tower was built in 1887.",
        retrieval_context=["The Eiffel Tower was constructed between 1887 and 1889."],
    )
    assert_test(test_case, [FaithfulnessMetric(threshold=0.8)])
```

### Arize Phoenix

OpenTelemetry-based observability combined with evaluation:

- Embedding visualization (2D/3D UMAP projections) to diagnose retrieval gaps — clusters far from query embeddings reveal coverage holes

- Production monitoring with drift detection across latency, token usage, and evaluation scores

- First-class support for LlamaIndex, LangChain, and raw OpenAI traces

- Best for: teams that need observability AND evaluation in one tool without running two separate stacks

Phoenix traces arrive via the standard OpenTelemetry OTLP exporter, so any framework that supports OTEL works out of the box — no vendor lock-in.

### MLflow GenAI Evaluation

Unified API that wraps RAGAS, DeepEval, and Phoenix metrics under one interface:

- `mlflow.genai.evaluate()` accepts 70+ judges drawn from multiple frameworks

- Single experiment-tracking UI for comparing prompt versions, retriever configs, and model upgrades side by side

- Runs are stored alongside model artifacts, making it easy to link an eval result to the exact code that produced it

- Best for: teams already using MLflow for the broader ML lifecycle who want GenAI evaluation without a separate tool

``` python
import mlflow

with mlflow.start_run():
    results = mlflow.genai.evaluate(
        model="runs:/abc123/rag-pipeline",
        data=eval_dataset,
        evaluators=["faithfulness", "answer_relevancy"],
    )
    print(results.metrics)
```

### Eval-Driven Development (EDD)

The emerging best practice in 2026: write evals **before** shipping changes, treating them like tests in a test suite.

EDD closes the loop between development and quality. If you cannot measure the change, you cannot ship it confidently.

Workflow:

1.  Define an eval dataset (questions + expected answers) covering the scenarios that matter most to users

2.  Run a baseline eval against the current system and record all four metric scores

3.  Make one targeted change — new prompt, new retriever, different chunk size, etc.

4.  Run the eval again and compare scores against the baseline

5.  Only deploy if metrics improve or hold steady; roll back if any score regresses below threshold

This loop turns quality from an afterthought into a gated property. Combined with CI/CD integration (see Practice exercise 5 below), every pull request gets an automatic quality verdict before human review.

### Tool Comparison

| Tool          | Strength                   | Best For              | GitHub Stars |
|---------------|----------------------------|-----------------------|--------------|
| RAGAS         | Reference-free RAG metrics | Quick RAG evaluation  | ~25k         |
| DeepEval      | 50+ metrics, pytest CI/CD  | Engineering teams     | ~15k         |
| Arize Phoenix | Observability + eval       | Production monitoring | ~10k         |
| LangSmith     | LangChain-native tracing   | LangChain stacks      | N/A (SaaS)   |
| MLflow        | Unified wrapper            | ML teams              | ~20k         |

Stars are approximate as of early 2026 and shift quickly. Check the repos directly for current figures. Prefer tool selection based on workflow fit, not popularity alone.

## Ragas Python API

Install:

``` bash
pip install ragas
```

### Canonical pattern: class-based metrics + `llm_factory` (async)

``` python
import asyncio
from openai import AsyncOpenAI
from ragas.llms import llm_factory
from ragas.metrics.collections import Faithfulness

async def evaluate():
    client = AsyncOpenAI()
    llm = llm_factory("gpt-5.2", client=client)
    scorer = Faithfulness(llm=llm)

    result = await scorer.ascore(
        user_input="When was the first Super Bowl?",
        response="The first Super Bowl was held on January 15, 1967.",
        retrieved_contexts=[
            "The First AFL-NFL World Championship Game was played "
            "on January 15, 1967, at the Los Angeles Memorial Coliseum."
        ],
    )
    print(f"Score: {result.value}")
    print(f"Reason: {result.reason}")

asyncio.run(evaluate())
```

### With a LangChain LLM wrapper

``` python
from ragas.metrics import (
    Faithfulness,
    AnswerRelevancy,
    ContextPrecision,
    ContextRecall,
)
from ragas.llms import LangchainLLMWrapper
from langchain.chat_models import init_chat_model

llm = init_chat_model("claude-sonnet-4-6")
evaluator_llm = LangchainLLMWrapper(llm)

metrics = [
    Faithfulness(llm=evaluator_llm),
    AnswerRelevancy(llm=evaluator_llm),
    ContextPrecision(llm=evaluator_llm),
    ContextRecall(llm=evaluator_llm),
]
```

### Legacy batch `evaluate()` (still supported)

``` python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_correctness,
    context_recall,
    context_precision,
)

evaluation_result = evaluate(
    dataset=ragas_eval_dataset,
    metrics=[faithfulness, answer_correctness, context_recall, context_precision],
)
eval_scores_df = evaluation_result.to_pandas()
```

### Newer metrics worth knowing

- `AnswerAccuracy` — stricter than AnswerRelevancy; checks factual correctness against a reference.

- `ResponseGroundedness` — similar to Faithfulness but with a different decomposition strategy.

- `DiscreteMetric` — build your own LLM-as-judge with a fixed enum of allowed values (`accurate`/`inaccurate`, `safe`/`unsafe`, etc.).

## LLM-as-Judge: When and How

Use an LLM as a judge when:

- Ground-truth labels do not exist at scale.

- The output is open-ended (summaries, code explanations).

- You need to measure quality dimensions that don’t have a deterministic function (coherence, helpfulness, tone).

**Pick a stronger judge than your generator.** If the agent under test is `claude-sonnet-4-6`, judge with `claude-opus-4-6` or `gpt-5.2`. A judge at the same level as the generator tends to rubber-stamp.

## Judge Bias to Watch For

1.  **Position bias** — judges favor the first option in a pairwise comparison. Randomize the order and average.

2.  **Length bias** — judges often prefer longer answers. Calibrate your rubric to penalize verbosity.

3.  **Self-preference** — a judge from the same model family often rates its own outputs higher. Cross-family judging mitigates this.

## Offline vs Online Evaluation

- **Offline**: run the full eval suite on a frozen test set before every release. RAGAS metrics + custom rubrics.

- **Online**: sample 1-5% of production traffic and evaluate in real-time (or near-real-time). Use [Observability: LangFuse & LangSmith](llmops-observability.md) patterns with LangFuse or LangSmith.

Both layers are necessary. Offline catches regressions; online catches drift from changing user behavior.

> **Note**
>
> Note
>
> The metrics on this page (RAGAS, LLM-as-judge) are specific to generative AI systems. For evaluation metrics on predictive and computer vision models — precision/recall, IoU, mAP, drift detection in the data-distribution sense — see **What Is Deep Learning & MLOps?**.

## Practice

### 1. Baseline the four metrics

Using the async `Faithfulness` / `AnswerRelevancy` / `ContextPrecision` / `ContextRecall` pattern, evaluate your RAG pipeline on a 20-question test set with known ground-truth contexts. Report all four scores as a markdown table. Identify which metric is the lowest — that is your next bottleneck.

### 2. Intentionally break one metric

Modify your pipeline to deliberately hurt Context Recall (e.g., lower `k` from 5 to 1). Re-run the eval suite. Confirm that Context Recall drops sharply while the generator-side metrics (Faithfulness, Answer Relevancy) may or may not move. This demonstrates metric orthogonality.

### 3. Write a custom `DiscreteMetric`

Create a `DiscreteMetric` that classifies answers as one of `confident`, `hedging`, or `refusing`. Use it to measure the distribution of refusal behavior on 30 out-of-context questions. A well-grounded pipeline should produce mostly `refusing` answers on questions it cannot answer.

### 4. Compare judges

Pick one hard question your pipeline answers imperfectly. Evaluate the same (question, answer, context) triple with:

1.  `gpt-5.2` as judge

2.  `claude-opus-4-6` as judge

3.  `claude-sonnet-4-6` as judge

Compare the Faithfulness scores. Do the judges agree? When they disagree, which one seems more reasonable? Write a paragraph on your finding.

### 5. Tie eval to CI

Set up a GitHub Actions job that runs the eval suite on every pull request and fails the build if any of the four metrics drops below a threshold. This turns quality into a gated property, not an afterthought.

## Review Questions

1.  **Which two RAGAS metrics evaluate the retriever (not the generator)?**

    - A. Faithfulness and Answer Relevancy

    - B. Context Precision and Context Recall

    - C. Answer Relevancy and Context Recall

    - D. Faithfulness and Context Precision

2.  **A low Faithfulness score most directly indicates what problem?**

    - A. The retriever is missing documents

    - B. The generator is hallucinating — producing claims not supported by the retrieved context

    - C. The user’s question is ambiguous

    - D. The embeddings are stale

3.  **Why is it recommended to use a stronger model as the judge than the model under test?**

    - A. To save cost

    - B. Same-level judges tend to rubber-stamp; a stronger judge catches errors the tested model cannot

    - C. It runs faster

    - D. It is required by the Ragas API

4.  **Which Ragas import gives you the async class-based `Faithfulness` metric?**

    - A. `from`` ``ragas.metrics`` ``import`` ``faithfulness`

    - B. `from`` ``ragas.metrics.collections`` ``import`` ``Faithfulness`

    - C. `from`` ``ragas`` ``import`` ``Faithfulness`

    - D. `from`` ``ragas.async`` ``import`` ``Faithfulness`

5.  **What is the primary bias of an LLM-as-judge in pairwise comparisons?**

    - A. Position bias (preference for the first option)

    - B. Alphabetical bias

    - C. Time-of-day bias

    - D. There is no bias

6.  **What does a high Context Recall but low Context Precision indicate?**

    - A. The retriever is missing relevant documents entirely

    - B. The retriever is fetching all the needed information but burying it among noise (poor ranking)

    - C. The generator is hallucinating

    - D. The user’s question is too simple

7.  **When should you deploy online evaluation (sampling production traffic) in addition to offline evaluation?**

    - A. Never — offline is always enough

    - B. Always — offline catches regressions but online catches drift from changing user behavior

    - C. Only when the budget allows

    - D. Only for chatbots

8.  **What is a good target score for a healthy production RAG on all four RAGAS metrics?**

    - A. ≥0.5

    - B. ≥0.85

    - C. Exactly 1.0

    - D. ≥0.99

9.  **Which `ragas.llms` helper wraps a LangChain chat model for use as an evaluator?**

    - A. `WrapLangchain`

    - B. `LangchainLLMWrapper`

    - C. `langchain_to_ragas`

    - D. `LCEvaluator`

10. **A `DiscreteMetric` in Ragas is used for what?**

    - A. Counting tokens

    - B. Building a custom LLM-as-judge with a fixed enum of allowed output values (e.g., `accurate`/`inaccurate`)

    - C. Measuring latency

    - D. Calculating cosine similarity

**View Answer Key**

1.  **B** — Context Precision and Context Recall target the retriever; Faithfulness and Answer Relevancy target the generator.

2.  **B** — Faithfulness measures whether answer claims are supported by context; low = hallucination.

3.  **B** — Same-level judges rubber-stamp; a stronger judge catches real errors.

4.  **B** — The class-based async API lives under `ragas.metrics.collections`.

5.  **A** — Position bias is well-documented; randomize order and average to mitigate.

6.  **B** — The retriever found the right info but buried it among irrelevant chunks.

7.  **B** — Both layers are needed: offline for regressions, online for drift.

8.  **B** — ≥0.85 on all four metrics is a reasonable production target; \<0.7 on any one is a smell.

9.  **B** — `LangchainLLMWrapper`.

10. **B** — `DiscreteMetric` lets you build LLM-as-judge metrics with a closed set of valid outputs.
