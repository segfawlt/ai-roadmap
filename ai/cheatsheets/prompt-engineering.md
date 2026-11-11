# Prompt Engineering Quick Reference

## Core principles

1.  **Be specific.** Vague asks (“summarize this”) yield vague answers. State the audience, length, format, and success criteria.

2.  **Show, don’t tell.** A single well-chosen example is worth a paragraph of instructions.

3.  **Separate concerns.** Use XML-style tags or Markdown headings to isolate instructions, context, and examples.

4.  **Constrain the output.** Ask for JSON / a specific schema / a fixed structure whenever downstream code will parse the response.

5.  **Let the model think.** For any non-trivial reasoning task, allow intermediate reasoning before the final answer.

## Role assignment

Open with a system message describing the persona and scope. This is the single highest-leverage prompt technique.

```
You are a senior Python code reviewer at a fintech company.
Your job is to flag security issues (OWASP Top 10), then style problems.
Only comment on things that would block a merge. Be terse.
```

## Few-shot examples

Provide 2–5 input→output pairs that cover the edge cases you care about. The model will generalize from the pattern.

```
Convert each product description to a JSON tag list.

Example 1:
Input: "Wireless noise-cancelling over-ear headphones."
Output: ["wireless", "noise-cancelling", "over-ear", "headphones"]

Example 2:
Input: "Stainless steel insulated water bottle, 500ml."
Output: ["stainless-steel", "insulated", "water-bottle", "500ml"]

Now:
Input: "{user_input}"
Output:
```

## Chain-of-thought (CoT)

Ask the model to reason step-by-step before answering. On Claude 4.6 and other reasoning models, this is often implicit via extended/adaptive thinking, but the prompt still helps for older or cheaper models.

```
Think through the problem carefully, then give the final answer.

Problem: A train leaves station A at 3pm going 60 mph. Another leaves
station B at 4pm going 80 mph toward A. Stations are 280 miles apart.
When do they meet?

Reasoning:
```

For models that expose reasoning tokens natively (Claude Opus 4.6, Sonnet 4.6 with adaptive thinking; OpenAI o-series; Gemini 3 with `ThinkingConfig`), you usually do not need to add “think step by step” manually — the model handles it internally.

## Structured output

Always prefer **schema-constrained output** over regex-parsing free text.

### With the Anthropic SDK

``` python
import anthropic

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=[{
        "name": "extract_contact",
        "description": "Extract structured contact info",
        "input_schema": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "email": {"type": "string"},
                "phone": {"type": "string"},
            },
            "required": ["name", "email"],
        },
        "strict": True,  # guarantees schema conformance
    }],
    tool_choice={"type": "tool", "name": "extract_contact"},
    messages=[{"role": "user", "content": "Jane Doe, [email protected], 555-0100"}],
)
```

### With LangChain `create_agent`

``` python
from pydantic import BaseModel, Field
from langchain.agents import create_agent

class ContactInfo(BaseModel):
    name: str = Field(description="Full name")
    email: str = Field(description="Email address")
    phone: str | None = Field(default=None, description="Phone if present")

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[],
    response_format=ContactInfo,
)
result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
print(result["structured_response"])  # ContactInfo instance
```

## XML-style tagging (Anthropic-preferred)

Claude is trained to respect XML-ish tags in prompts. Use them to isolate sections that the model should treat as data rather than instructions.

```
Review the pull request below and return a list of blocking issues.

<diff>
{pull_request_diff}
</diff>

<style_guide>
{internal_style_guide}
</style_guide>

Respond with a JSON array of {"line": int, "severity": str, "message": str}.
```

## Negative instructions — use sparingly

LLMs can interpret negative instructions (“don’t do X”) as emphasis on X. Rephrase as positive directives whenever possible.

- ❌ “Don’t use passive voice.”

- ✅ “Write every sentence in active voice.”

## Grounding and anti-hallucination

When the model must cite sources, be explicit:

```
Answer only using information contained in <context>. If the answer is
not in the context, reply exactly: "I don't know based on the provided
sources." Cite the source ID in square brackets after each claim.

<context>
[src-1] ...
[src-2] ...
</context>

Question: {question}
```

## Temperature and sampling

- `temperature=0.0` — deterministic, best for extraction, classification, code generation.

- `temperature=0.7` — balanced, good for most conversational tasks.

- `temperature=1.0` — Gemini 3 default and recommended; unusual to change.

For OpenAI Responses API and Anthropic messages with thinking enabled, temperature effects are muted — the model already has internal variance.

## Prompt caching (long static context)

If you send the same 50KB system prompt on every request, enable prompt caching so you pay the cache-read rate (typically 0.1× input) instead of the full input rate.

``` python
client.messages.create(
    model="claude-sonnet-4-6",
    system=[
        {"type": "text", "text": "You are..."},
        {
            "type": "text",
            "text": long_style_guide_text,
            "cache_control": {"type": "ephemeral"},
        },
    ],
    messages=[...],
)
```

See [Observability: LangFuse & LangSmith](../llmops/llmops-observability.md) for tracing these patterns in production.

## Practice

### 1. Rewrite a vague prompt

Take the prompt: `"summarize`` ``this`` ``article"`.

Rewrite it with:

- An explicit audience (“a busy executive who has 30 seconds”)

- A target length (“3 bullet points, each under 15 words”)

- A format (“plain text, no Markdown”)

- A success criterion (“must name the two parties and the dollar amount”)

Run both prompts through the same model and compare the outputs.

### 2. Few-shot classification

Build a prompt that classifies customer support tickets into one of `billing`, `bug`, `feature-request`, `other`. Provide 4 examples in the prompt. Test with 10 held-out tickets and measure accuracy.

Target: ≥90% accuracy on a balanced test set using `claude-sonnet-4-6` at `temperature=0`.

### 3. Structured extraction with strict tool use

Use the Anthropic SDK with `strict:`` ``true` to extract the following from a job posting:

``` python
{
    "title": str,
    "company": str,
    "location": str,
    "remote": bool,
    "salary_min": int | None,
    "salary_max": int | None,
    "required_skills": list[str],
}
```

Verify the response parses as valid JSON on 20 real postings without any `try/except` fallbacks.

### 4. Prompt caching measurement

Set up a long system prompt (~2,000 tokens). Issue 10 identical user queries. Compare total cost with and without `cache_control` using the `usage.cache_creation_input_tokens` and `usage.cache_read_input_tokens` fields from the response.

Expected: after the first request, subsequent requests should report most input tokens as cache reads, cutting cost by ~90% on the cached portion.

### 5. Grounding and refusal

Build a RAG-style prompt that must answer **only** from a provided `<context>` block. Test with 5 questions that are answerable from the context and 5 that are not. The model should refuse exactly on the 5 unanswerable ones with the literal string `"I`` ``don't`` ``know`` ``based`` ``on`` ``the`` ``provided`` ``sources."`.

Measure refusal precision and recall.

## Review Questions

1.  **Which prompt technique consistently produces the largest quality improvement with the smallest token budget?**

    - A. Setting temperature to 1.0

    - B. Assigning a clear role in the system message

    - C. Using ALL CAPS for important instructions

    - D. Adding the word “please” to every instruction

2.  **You need to extract a strict JSON schema from unstructured text. Which option is most reliable?**

    - A. Ask the model to “return JSON” in the prompt and parse it yourself

    - B. Use tool use with `strict:`` ``true` (Anthropic) or `response_format` (OpenAI / LangChain)

    - C. Regex the output

    - D. Set temperature to 2.0 for variety

3.  **For Claude Opus 4.6 with adaptive thinking enabled, do you still need to write “think step by step” in the prompt?**

    - A. Yes, always

    - B. No — the model reasons internally; an explicit CoT instruction is usually redundant

    - C. Only on weekends

    - D. Only for math problems

4.  **What is the recommended temperature for Gemini 3 models?**

    - A. 0.0

    - B. 0.5

    - C. 1.0 (default)

    - D. 2.0

5.  **Why is XML-style tagging particularly effective with Claude?**

    - A. Claude was trained to respect XML-ish tags for isolating instructions from data

    - B. XML is faster to parse than JSON

    - C. It reduces token count

    - D. It’s required by the API

6.  **Your system prompt is 50KB and never changes between requests. How do you cut cost?**

    - A. Truncate it to 5KB

    - B. Enable prompt caching with `cache_control:`` ``{"type":`` ``"ephemeral"}`

    - C. Send it only on the first request

    - D. Use a smaller model

7.  **What is the failure mode of negative instructions like “Don’t use passive voice”?**

    - A. The model throws an error

    - B. LLMs sometimes interpret the negation as emphasis and do the opposite — prefer positive directives

    - C. They cost more tokens

    - D. They are case-sensitive

8.  **A RAG system should refuse to answer when the context is insufficient. How do you enforce this?**

    - A. Lower the temperature

    - B. Explicitly instruct the model to reply with a fixed refusal string when the answer is not in the context

    - C. Hope it works

    - D. Use a larger model

9.  **When building a few-shot classifier, how many examples are typically enough?**

    - A. Exactly 1

    - B. 2–5 examples that cover the edge cases

    - C. At least 100

    - D. As many as the context window allows

10. **Which parameter makes tool use schema-conforming on the Anthropic API?**

    - A. `temperature:`` ``0`

    - B. `strict:`` ``true` on the tool definition

    - C. `json_mode:`` ``true`

    - D. `force_schema:`` ``true`

**View Answer Key**

1.  **B** — Role assignment is the single highest-leverage technique.

2.  **B** — Schema-constrained tool use beats regex parsing every time.

3.  **B** — Reasoning models handle CoT internally; an explicit “think step by step” is redundant (and sometimes harmful).

4.  **C** — Google explicitly recommends keeping `temperature=1.0` default for Gemini 3 models.

5.  **A** — Claude is trained to recognize XML-ish tags as structural boundaries.

6.  **B** — Prompt caching cuts cached reads to ~0.1× input cost.

7.  **B** — Rephrase as positive directives (“Write in active voice”).

8.  **B** — Explicit refusal instructions with a fixed string make refusals detectable and measurable.

9.  **B** — 2–5 carefully chosen examples typically get you most of the benefit.

10. **B** — `strict:`` ``true` on the tool guarantees the model’s output matches the declared schema.
