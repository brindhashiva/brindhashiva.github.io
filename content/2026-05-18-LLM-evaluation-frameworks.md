Title: LLM Evaluation Frameworks
Date: 2026-05-18
Category: GenAI
Tags: LLM Evaluation, DeepEval, Promptfoo, RAGAS, Testing, CI/CD
Slug: LLM-evaluation-frameworks
Status: published

The evaluating-RAG-systems post earlier in this series built retrieval and faithfulness metrics with RAGAS, and the agent-observability post covered scoring trajectories from traces. This post is about the tooling layer that sits above both: the frameworks that turn one-off eval scripts into something that runs in CI, gates deploys, and catches regressions before a user does. The RAG-specific metrics from earlier still apply; what changes here is how you run them repeatedly, at scale, as part of an actual engineering workflow.

## What This Layer Actually Needs to Do

**Test harness** — The thing that runs your eval cases, the same role pytest plays for ordinary code, but for cases where the assertion is "is this output good," not "does this equal that."

**Golden dataset** — The evaluation set from the RAG-evaluation post, generalized beyond retrieval: input, expected behavior or reference output, and enough metadata to know what a pass actually means for that case.

**Regression gate** — A threshold that fails a build when a metric drops beyond an agreed margin, turning evaluation from something a human checks occasionally into something CI enforces on every change.

**LLM-as-judge grading** — Using a model to score an output against a rubric, the same mechanism RAGAS uses for faithfulness and context recall, generalized to whatever quality dimension you're grading (helpfulness, tone, correctness against a reference).

## Two Shapes of Framework

Frameworks in this space tend to fall into one of two shapes, and the choice matters more than which specific brand you pick:

- **Pytest-style, code-first** — Tests are Python functions with assertions, run through your existing test runner, and fail your CI the way any other test failure would. DeepEval is the clearest example of this shape: an `LLMTestCase` and an `assert_test` call, with a broad library of built-in metrics (G-Eval, hallucination detection, answer relevancy, faithfulness, bias, and others) available as ready-made assertions.
- **Declarative, config-first** — Test cases and assertions live in a config file (YAML, typically), and a CLI runs them, with no Python required to write or run a basic suite. Promptfoo is the clearest example: language-agnostic, well-suited to prompt-engineering iteration where the unit under test is the prompt itself, and to teams that want eval cases reviewable by non-engineers.

Pick based on who's writing the test cases and where they need to live. A team that already thinks in pytest gets a pytest-style framework for free; a team where prompt engineers and non-engineers are writing eval cases benefits from a config format that doesn't require Python literacy to extend.

## A Pytest-Style Suite

```python
# pip install deepeval
from deepeval import assert_test
from deepeval.metrics import GEval, FaithfulnessMetric
from deepeval.test_case import LLMTestCase, LLMTestCaseParams

def test_refund_policy_answer():
    actual_output = my_agent.invoke("Can I get a refund after 45 days?")

    correctness = GEval(
        name="Correctness",
        criteria="Does the actual output correctly state the refund window "
                 "and any conditions, matching the expected output?",
        evaluation_params=[LLMTestCaseParams.ACTUAL_OUTPUT, LLMTestCaseParams.EXPECTED_OUTPUT],
        threshold=0.8,
    )
    faithfulness = FaithfulnessMetric(threshold=0.8)

    test_case = LLMTestCase(
        input="Can I get a refund after 45 days?",
        actual_output=actual_output,
        expected_output="Refunds are available within 30 days; after that, "
                         "only store credit is offered.",
        retrieval_context=retrieve_policy_context("refund window"),
    )

    assert_test(test_case, [correctness, faithfulness])
```

<!-- VERIFY before publishing: confirm DeepEval's current class and parameter names (GEval, FaithfulnessMetric, LLMTestCase, evaluation_params) against the package's own docs for your pinned version; this library iterates quickly and the constructor signatures shown here should be treated as illustrative rather than guaranteed current. -->

Run with `pytest`, and this fails the build exactly like a normal assertion failure would, which is the entire point: eval failures become CI failures, not something a human has to remember to check.

## A Declarative Suite

```yaml
# promptfooconfig.yaml
prompts:
  - "Answer this support question using only the provided context: {{question}}"

providers:
  - openai:gpt-4o-mini

tests:
  - vars:
      question: "Can I get a refund after 45 days?"
    assert:
      - type: llm-rubric
        value: "States that the refund window is 30 days and mentions store credit as the alternative after that."
      - type: not-contains
        value: "I don't know"
```

<!-- VERIFY before publishing: confirm the current promptfooconfig.yaml schema (field names like assert, llm-rubric) against promptfoo's own documentation for the version you pin; config-driven eval tools tend to add fields across releases. -->

Run with `promptfoo eval`, which produces a pass/fail report per test case and integrates into CI the same way, without a single line of Python needed to define or extend the suite.

## RAG and Agent Metrics Still Apply Here

Nothing here replaces the metrics from earlier in this series; this layer is where they get run repeatedly. RAGAS's retrieval and faithfulness metrics, or the trajectory scoring from the agent-observability post, plug into either shape of framework as the actual scoring logic behind a test case:

```python
# Wiring a RAGAS metric into a DeepEval-style pytest suite, rather than
# treating them as competing tools.
from ragas.metrics import Faithfulness
from ragas.llms import LangchainLLMWrapper

def test_rag_faithfulness():
    docs = retriever.invoke("refund policy after 45 days")
    answer = generate_answer("refund policy after 45 days", docs)

    faithfulness = Faithfulness()
    judge = LangchainLLMWrapper(judge_model)
    score = faithfulness.single_turn_score(
        {"user_input": "refund policy after 45 days",
         "response": answer,
         "retrieved_contexts": [d.page_content for d in docs]},
        llm=judge,
    )
    assert score >= 0.8
```

<!-- VERIFY before publishing: same RAGAS version caveat as the evaluating-RAG-systems post earlier in this series, single_turn_score's exact signature is version-dependent and worth confirming against your pinned release. -->

The test harness (pytest here) provides the CI integration and reporting; RAGAS provides the domain-specific scoring logic for retrieval quality. Neither replaces the other.

## What to Actually Gate On

Not every metric belongs in a hard CI gate. Calibrate which ones block a merge versus which ones you track as a trend:

- **Deterministic checks (schema validity, presence of required fields, banned phrases)** — Hard gate. These have no ambiguity and should block a merge on any failure.
- **Retrieval metrics (hit rate, MRR from the RAG-evaluation post)** — Hard gate with a reasonable margin, since these are cheap to compute and directly measure whether a change degraded retrieval.
- **LLM-judged quality metrics (faithfulness, correctness, tone)** — Gate on a threshold, but expect noise; treat a single borderline failure as a signal to investigate, not necessarily to block, and track the trend over multiple runs rather than reacting to one score.
- **Agent trajectory correctness (right tool, right sequence, from the observability post)** — Track as a metric on every merge, but be cautious gating hard on it early; trajectory scoring is newer and noisier than output-quality scoring, and an overly strict gate here can block legitimate architectural changes that achieve the same result via a different, equally valid path.

## Building the Habit, Not Just the Tooling

The tooling only pays off if eval cases keep growing. Every production failure caught (via the tracing from the observability post) is a candidate new test case; every prompt or model change is a reason to run the suite before merging, not after a complaint arrives. The framework you pick matters less than whether your team actually treats a failing eval the way it treats a failing unit test: as a blocker, not a suggestion.

> An eval framework that nobody's afraid to fail isn't protecting anything. The value isn't in having the metrics; it's in making a bad change impossible to merge without someone noticing.

---
*Worth forwarding to whoever's still eyeballing five example outputs before every deploy and calling that "evaluation."*