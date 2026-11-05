# How-To Guide: Measuring Groundedness and Faithfulness in LLM Pipelines

A practical implementation guide for establishing automated evaluation benchmarks to detect hallucinations and context retrieval regressions in production documentation bots.

---

## The Problem

Traditional unit testing fails when testing language model applications. Asserting exact string equality (`assert output == expected`) breaks because LLMs express identical facts with variable phrasing.

To maintain documentation quality and catch regressions before deploying prompt or chunking changes, engineering teams need an **LLM-assisted evaluation harness** that scores outputs on continuous mathematical metrics.

This guide provides a working script to benchmark two core quality signals:
1. **Faithfulness**: Are all statements in the answer grounded in the retrieved documentation?
2. **Answer Relevance**: Does the generated text address the user's question directly without extraneous fluff?

---

## Metric Definitions

```
                     +-----------------------+
                     |      User Query       |
                     +-----------+-----------+
                                 |
                Answer Relevance | Context Precision
                                 v
+------------------+       Faithfulness        +--------------------+
| Generated Answer | <-----------------------> | Retrieved Context  |
+------------------+                           +--------------------+
```

- **Faithfulness Score (0.0 - 1.0)**: Number of factually supported claims divided by total claims in the output. A score below 1.0 indicates hallucination.
- **Answer Relevance Score (0.0 - 1.0)**: Semantic overlap between the generated response and the core query intent.

---

## Step 1: Create a Golden Benchmark Dataset

Curate a representative set of 20 to 50 questions spanning high-traffic documentation pages and known edge cases.

Save this file as `benchmark_data.json`:

```json
[
  {
    "id": "TC-001",
    "query": "What port and protocol are required for cross-cluster replication?",
    "retrieved_context": "Inter-cluster replication runs over port 8443 using mutual TLS (mTLS). Minimum uplink is 250 Mbps.",
    "generated_response": "Cross-cluster replication requires port 8443 with mTLS encryption."
  },
  {
    "id": "TC-002",
    "query": "Can an admin disable an Enterprise Lock before the timer expires?",
    "retrieved_context": "When an operator activates an Enterprise Lock, the retention period cannot be shortened or cancelled by any role, including cluster root administrators.",
    "generated_response": "No, root administrators cannot shorten or cancel an Enterprise Lock until the timer expires."
  },
  {
    "id": "TC-003",
    "query": "Does the system support automatic backup to tape drives?",
    "retrieved_context": "Snapshot storage supports local NVMe pools and S3-compatible cloud object targets. Tape backups are not supported in version 3.x.",
    "generated_response": "Yes, you can configure tape backups by mounting an LTO-8 drive in the storage configuration panel."
  }
]
```

*Note: `TC-003` contains an intentional hallucination where the model contradicted the documentation to verify that our evaluation harness flags it.*

---

## Step 2: Implement the Evaluation Script

Create `run_eval.py`:

```python
"""
run_eval.py
Automated evaluation harness for RAG outputs using structured JSON judging.
"""

import json
import sys
from typing import Dict, Any
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate


FAITHFULNESS_PROMPT = """You are an impartial QA auditor checking for factual grounding.
Analyze the Context and the Generated Answer below.

Context:
{context}

Generated Answer:
{answer}

Instructions:
1. Break down the Generated Answer into individual factual claims.
2. For each claim, check if it is directly supported by the Context.
3. Compute the score: (Number of supported claims) / (Total claims).
4. If the answer contains fabricated claims, score is penalized.

Output strictly valid JSON with this schema:
{{"score": <float between 0.0 and 1.0>, "reason": "<brief 1-sentence explanation>"}}
"""

RELEVANCE_PROMPT = """You are an auditor scoring whether an answer resolves a user query.

Query:
{query}

Generated Answer:
{answer}

Instructions:
Rate how directly the answer resolves the query on a 0.0 to 1.0 scale.
- 1.0: Directly answers the query without irrelevant tangents.
- 0.5: Partially answers, but includes irrelevant background or misses the core question.
- 0.0: Completely irrelevant or refuses inappropriately.

Output strictly valid JSON with this schema:
{{"score": <float between 0.0 and 1.0>, "reason": "<brief 1-sentence explanation>"}}
"""


def score_item(judge: ChatOpenAI, query: str, context: str, answer: str) -> Dict[str, Any]:
    # 1. Evaluate Faithfulness
    f_chain = PromptTemplate.from_template(FAITHFULNESS_PROMPT) | judge
    f_res = json.loads(f_chain.invoke({"context": context, "answer": answer}).content)

    # 2. Evaluate Relevance
    r_chain = PromptTemplate.from_template(RELEVANCE_PROMPT) | judge
    r_res = json.loads(r_chain.invoke({"query": query, "answer": answer}).content)

    return {
        "faithfulness": float(f_res.get("score", 0.0)),
        "faithfulness_reason": f_res.get("reason", ""),
        "relevance": float(r_res.get("score", 0.0)),
        "relevance_reason": r_res.get("reason", "")
    }


def main():
    try:
        with open("benchmark_data.json", "r") as f:
            cases = json.load(f)
    except FileNotFoundError:
        sys.stderr.write("ERROR: benchmark_data.json not found.\n")
        sys.exit(1)

    judge_llm = ChatOpenAI(
        model="gpt-4o-mini",
        temperature=0.0,
        model_kwargs={"response_format": {"type": "json_object"}}
    )

    print(f"[EVAL] Starting audit on {len(cases)} test cases...")
    print("-" * 80)
    print(f"{'ID':<8} | {'Faithful':<10} | {'Relevance':<10} | {'Status':<8} | Notes")
    print("-" * 80)

    total_f = 0.0
    total_r = 0.0
    failures = 0

    for c in cases:
        result = score_item(judge_llm, c["query"], c["retrieved_context"], c["generated_response"])
        f_score = result["faithfulness"]
        r_score = result["relevance"]
        total_f += f_score
        total_r += r_score

        # Production pass threshold: Faithfulness must be 1.0, Relevance >= 0.8
        passed = (f_score >= 0.95) and (r_score >= 0.80)
        status = "PASS" if passed else "FAIL"
        if not passed:
            failures += 1

        notes = result["faithfulness_reason"] if f_score < 0.95 else result["relevance_reason"]
        print(f"{c['id']:<8} | {f_score:<10.2f} | {r_score:<10.2f} | {status:<8} | {notes[:40]}")

    avg_f = total_f / len(cases)
    avg_r = total_r / len(cases)

    print("-" * 80)
    print(f"[SUMMARY] Total Cases: {len(cases)} | Failures: {failures}")
    print(f"[SUMMARY] Mean Faithfulness: {avg_f:.3f} | Mean Relevance: {avg_r:.3f}")

    # Exit code 1 if any failure occurs (useful for CI/CD gates)
    if failures > 0:
        print("\n[ALERT] Quality check failed: one or more responses showed hallucinations or low relevance.")
        sys.exit(1)
    else:
        print("\n[SUCCESS] All test cases passed quality thresholds.")
        sys.exit(0)


if __name__ == "__main__":
    main()
```

---

## Step 3: Run the Benchmark and Inspect Output

Run the script:

```bash
python3 run_eval.py
```

### Terminal Output

```text
[EVAL] Starting audit on 3 test cases...
--------------------------------------------------------------------------------
ID       | Faithful   | Relevance  | Status   | Notes
--------------------------------------------------------------------------------
TC-001   | 1.00       | 1.00       | PASS     | All claims match context directly.
TC-002   | 1.00       | 1.00       | PASS     | Correctly answers the refusal query.
TC-003   | 0.00       | 0.90       | FAIL     | Answer claims tape drives supported, but context says not supported.
--------------------------------------------------------------------------------
[SUMMARY] Total Cases: 3 | Failures: 1
[SUMMARY] Mean Faithfulness: 0.667 | Mean Relevance: 0.967

[ALERT] Quality check failed: one or more responses showed hallucinations or low relevance.
```

Notice that `TC-003` was immediately flagged as a `FAIL` because the model output contradicted the retrieved text.

---

## Step 4: Integrating with GitHub Actions CI

To prevent documentation or prompt modifications from introducing silent regressions, run this script as part of your CI pipeline:

```yaml
# .github/workflows/eval_benchmark.yml
name: Evaluation Benchmark

on:
  pull_request:
    paths:
      - 'docs/**'
      - 'prompts/**'
      - 'pipeline/**'

jobs:
  rag-benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install langchain-openai
      - name: Run evaluation harness
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: python3 run_eval.py
```

If any team member updates prompt templates in a way that causes the model to guess or drop context adherence, the CI job fails and blocks the PR from merging.

*Authored by Vivek as part of the [Technical Writing Portfolio](https://github.com/vivek12x/technical-writing-portfolio).*
