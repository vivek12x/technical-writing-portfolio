# How-To Guide: Evaluating GenAI Outputs for Accuracy & Groundedness

> **A practical workflow for measuring hallucination rates, retrieval precision, and answer faithfulness in production AI documentation pipelines.**

---

## The Challenge

Deploying a Generative AI assistant is straightforward; guaranteeing that its answers are accurate, grounded, and non-hallucinatory across thousands of user queries is hard.

Unlike traditional software with deterministic unit tests (`assert add(2, 2) == 4`), language models produce variable natural language outputs. To maintain high enterprise quality, documentation and AI teams must implement **LLM-assisted evaluation benchmarks**.

This guide demonstrates how to establish an automated evaluation pipeline using the **RAG Triad** framework.

---

## The Core Metric Triad

To diagnose why a system produces an incorrect answer, we measure three independent dimensions:

```
                  ┌────────────────────────┐
                  │       User Query       │
                  └───────────┬────────────┘
                              │
             Answer Relevance │ Context Precision
                              ▼
┌──────────────────┐  Faithfulness   ┌──────────────────┐
│ Generated Answer │ ◄─────────────► │ Retrieved Context│
└──────────────────┘                 └──────────────────┘
```

1. **Faithfulness (Groundedness)**: Does every claim in the generated answer stem directly from the retrieved context? (Detects hallucinations).
2. **Answer Relevance**: Does the generated answer directly address the user's prompt without introducing extraneous tangents? (Detects off-topic drift).
3. **Context Precision / Recall**: Did the retrieval engine pull the exact text needed to answer the question? (Diagnoses search vs. synthesis bottlenecks).

---

## Step 1: Create a Golden Evaluation Dataset

An evaluation pipeline requires a curated test set containing realistic questions, reference context, and verified ground-truth answers.

Create `eval_dataset.json`:

```json
[
  {
    "question": "What is the minimum bandwidth required for asynchronous cross-region replication?",
    "ground_truth": "The minimum network bandwidth requirement is 100 Mbps dedicated uplinks.",
    "context": [
      "Replication jobs run asynchronously across geographical regions. Minimum network bandwidth requirement: 100 Mbps dedicated uplinks. Recovery Point Objective target is 15 minutes."
    ]
  },
  {
    "question": "Can an administrator delete a compliance locked snapshot before expiration?",
    "ground_truth": "No, retention periods cannot be shortened or deleted by any user, including root administrators.",
    "context": [
      "When an administrator enables a Compliance Lock on a storage view, the retention period cannot be shortened by any user, including root administrators."
    ]
  }
]
```

---

## Step 2: Implement Automated LLM-as-a-Judge Evaluation

Using an automated judge model with structured evaluation prompts allows rapid, quantitative scoring on a 0.0 to 1.0 scale.

Create `evaluate.py`:

```python
"""
evaluate.py
Evaluates RAG pipeline outputs for Faithfulness and Answer Relevance.
"""

import json
from dataclasses import dataclass
from typing import List, Dict
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate


@dataclass
class EvalResult:
    question: str
    faithfulness_score: float
    relevance_score: float
    reasoning: str


def evaluate_faithfulness(judge_llm: ChatOpenAI, context: str, answer: str) -> Dict:
    """Evaluates whether all claims in the answer are supported by the context."""
    prompt = PromptTemplate.from_template(
        """You are an impartial evaluator auditing AI responses for factual grounding.
Examine the Context and the Answer.

Context:
{context}

Answer:
{answer}

Task:
1. Count the total number of factual statements made in the Answer.
2. Count how many of those statements are directly supported by the Context.
3. Calculate faithfulness = (supported statements) / (total statements).
4. Provide a brief 1-sentence justification.

Respond strictly in valid JSON format:
{{"score": <float between 0.0 and 1.0>, "reason": "<string>"}}
"""
    )
    chain = prompt | judge_llm
    response = chain.invoke({"context": context, "answer": answer})
    return json.loads(response.content)


def evaluate_relevance(judge_llm: ChatOpenAI, question: str, answer: str) -> Dict:
    """Evaluates whether the answer directly resolves the user question."""
    prompt = PromptTemplate.from_template(
        """You are an evaluator assessing answer relevance.
Question:
{question}

Answer:
{answer}

Task:
Score the relevance from 0.0 (completely irrelevant) to 1.0 (directly answers the prompt without fluff).

Respond strictly in valid JSON format:
{{"score": <float between 0.0 and 1.0>, "reason": "<string>"}}
"""
    )
    chain = prompt | judge_llm
    response = chain.invoke({"question": question, "answer": answer})
    return json.loads(response.content)


def run_benchmark(dataset_path: str = "eval_dataset.json"):
    with open(dataset_path, "r") as f:
        data = json.load(f)

    judge = ChatOpenAI(model="gpt-4o", temperature=0.0, model_kwargs={"response_format": {"type": "json_object"}})
    
    print(f"📊 Starting Evaluation on {len(data)} test scenarios...\n")
    print(f"{'Question':<50} | {'Faithfulness':<12} | {'Relevance':<10}")
    print("-" * 78)

    total_faithfulness = 0.0
    total_relevance = 0.0

    for item in data:
        q = item["question"]
        ctx = " ".join(item["context"])
        ans = item.get("generated_answer", item["ground_truth"])  # Mocking with ground truth

        f_eval = evaluate_faithfulness(judge, ctx, ans)
        r_eval = evaluate_relevance(judge, q, ans)

        f_score = f_eval["score"]
        r_score = r_eval["score"]

        total_faithfulness += f_score
        total_relevance += r_score

        short_q = (q[:47] + '...') if len(q) > 47 else q
        print(f"{short_q:<50} | {f_score:<12.2f} | {r_score:<10.2f}")

    avg_f = total_faithfulness / len(data)
    avg_r = total_relevance / len(data)
    print("-" * 78)
    print(f"📈 Aggregate Performance: Faithfulness = {avg_f:.2f} | Relevance = {avg_r:.2f}")


if __name__ == "__main__":
    run_benchmark()
```

---

## Step 3: Diagnosing Failure Modes

When your benchmark scores dip below acceptable thresholds (typically $>0.90$ for production), use the following diagnostic matrix:

| Observed Symptom | Primary Metric Failing | Root Cause | Remediation Action |
| :--- | :--- | :--- | :--- |
| Model gives an articulate answer containing fabricated facts. | **Faithfulness** (< 0.85) | Context was missing the answer, but the system prompt lacked strict negative refusal constraints. | Strengthen negative guardrails in system prompt (`"If the context does not explicitly say X, state you do not know"`). |
| Model says *"I don't know"* to questions clearly documented in the manual. | **Context Recall** (< 0.70) | Retrieval failed to pull the relevant chunk into the top-$K$. | Adjust chunking strategy (reduce chunk size, add chunk overlap), or enable hybrid search (BM25 + vectors). |
| Model quotes the entire manual page instead of answering the query. | **Answer Relevance** (< 0.80) | The LLM was instructed too conservatively or the prompt lacked conciseness directives. | Refine system prompt to instruct: *"Be direct, concise, and provide the exact answer in 2-3 sentences."* |

---

## Step 4: Automating Evaluation in CI/CD

To prevent documentation updates or prompt changes from causing regressions:

1. **Pre-commit Checks**: Run a fast 10-query sanity check before merging changes to prompt templates.
2. **Nightly Regression Runs**: Execute the full golden dataset (100+ scenarios) nightly.
3. **Alert Thresholds**: Block deployment if aggregate Faithfulness drops below `0.92`.

---

*Authored by Vivek as part of the [Technical Writing Portfolio](https://github.com/vivek12x/technical-writing-portfolio).*
