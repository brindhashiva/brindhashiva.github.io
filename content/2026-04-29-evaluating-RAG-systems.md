Title: Evaluating RAG Systems
Date: 2026-04-29
Category: GenAI
Tags: RAG, Evaluation, RAGAS, LLM-as-Judge, Metrics, Testing
Slug: evaluating-rag-systems
Status: published

Most RAG teams tune by anecdote: try five queries, eyeball the answers, ship. That works until you swap a chunk size, a reranker, or a model and can't tell whether quality moved. RAG has two independently failing halves, retrieval and generation, and a single "does the answer look right?" check can't tell you which one broke. Evaluation is what turns your pipeline from something you demo into something you can change safely.

## Measure the Two Halves Separately

**Retrieval metrics** — Did the right chunks come back, and how high were they ranked? Hit rate@k (was any gold chunk in the top k), MRR (how early the first gold chunk appears), and recall@k. These need no LLM, are deterministic, and are cheap enough to run on every commit.

**Context precision and recall** — LLM-judged cousins of the above. Context precision asks how much of the retrieved context was actually useful; context recall asks whether the retrieved context contained what was needed to produce the reference answer.

**Faithfulness** — Whether every claim in the answer is supported by the retrieved context. This is your hallucination metric, and it is deliberately independent of whether the answer is correct: a faithful answer can still be wrong if retrieval was wrong.

**Answer correctness / relevancy** — Whether the answer matches a reference answer, or actually addresses the question asked. Needs reference answers or an LLM judge, and is the noisiest of the four.

The diagnostic value comes from the combination. Low context recall with high faithfulness means retrieval is your problem. High context recall with low faithfulness means the model is ignoring or embellishing what it was given.

## Start With a Golden Set

Every metric below is only as good as the dataset behind it.

- **Source questions from real usage**: query logs, support tickets, search history. Synthetic questions are fine to fill gaps, but they skew toward questions your documents make easy.
- **Label the gold chunk IDs**, not just reference answers. It's what lets you compute retrieval metrics without an LLM.
- **Include unanswerable questions** with an expected refusal. If every test question has an answer, you're not testing your refusal path.
- **Start small (50-200 examples) and grow it** with every production failure you fix. A failure that isn't added to the set will come back.

## Code: Retrieval Metrics With No Dependencies

This is the metric set to run first, and it works with any retriever that returns documents carrying an `id` in their metadata:

```python
def evaluate_retrieval(golden_set, retriever, k: int = 5) -> dict[str, float]:
    """golden_set: [{"question": str, "gold_ids": set[str]}, ...]"""
    hits, reciprocal_ranks = 0, 0.0

    for ex in golden_set:
        docs = retriever.invoke(ex["question"])[:k]
        ranked_ids = [d.metadata["id"] for d in docs]

        first_hit = next(
            (i for i, doc_id in enumerate(ranked_ids, start=1)
             if doc_id in ex["gold_ids"]),
            None,
        )
        if first_hit is not None:
            hits += 1
            reciprocal_ranks += 1.0 / first_hit

    n = len(golden_set)
    return {f"hit_rate@{k}": hits / n, f"mrr@{k}": reciprocal_ranks / n}

# Compare configurations on the same set:
# evaluate_retrieval(golden, vector_only_retriever)
# evaluate_retrieval(golden, hybrid)
# evaluate_retrieval(golden, reranked)
```

Run this before and after every change to chunking, hybrid weights, or reranking. It is the fastest way to find out whether "improvements" from the earlier posts in this series improved anything for your data.

## Code: LLM-Judged Metrics With RAGAS

For faithfulness and context recall you need a judge model. RAGAS is the common open-source option. Here is the classic `evaluate()` pattern with `EvaluationDataset`:

```python
# pip install ragas langchain-openai   (pin your ragas version; see note below)
import os
from langchain_openai import ChatOpenAI
from ragas import EvaluationDataset, evaluate
from ragas.llms import LangchainLLMWrapper
from ragas.metrics import Faithfulness, LLMContextRecall, FactualCorrectness

rows = []
for ex in golden_set:
    docs = retriever.invoke(ex["question"])
    rows.append({
        "user_input": ex["question"],
        "retrieved_contexts": [d.page_content for d in docs],
        "response": answer_fn(ex["question"], docs),   # your generation step
        "reference": ex["reference"],
    })

dataset = EvaluationDataset.from_list(rows)
judge = LangchainLLMWrapper(ChatOpenAI(model=os.environ["JUDGE_MODEL"]))

result = evaluate(
    dataset=dataset,
    metrics=[Faithfulness(), LLMContextRecall(), FactualCorrectness()],
    llm=judge,
)
print(result.to_pandas().describe())
```

<!-- VERIFY before publishing: RAGAS is mid-migration. The pattern above (evaluate + LangchainLLMWrapper + legacy metrics) is the documented quickstart, but 0.4.x emits deprecation warnings pointing to `ragas.metrics.collections` + `llm_factory`, and a GitHub issue from March 2026 reports evaluate() rejecting the new collections metrics. Pin an exact ragas version, run this against it, and update the snippet if you move to the new API. Also confirm the metric class names against the version you pin. -->

The version caveat is not boilerplate. RAGAS changed its API substantially between 0.1 and 0.2 (`Dataset.from_dict` is gone) and is deprecating its LangChain wrappers in the 0.4 line. If you copy an old tutorial into a new install, expect breakage. Pin the version, and store it next to your scores, because a number produced by an unrecorded judge configuration can't be reproduced.

## Judge Hygiene

- **Don't grade your own homework.** Use a different judge model than your generator where you can. Models tend to prefer their own style of output.
- **Calibrate against humans.** Hand-label 30-50 examples, run the judge on them, and check agreement. If the judge can't reproduce your labels, its scores aren't measuring what you think.
- **Track variance.** LLM judges are non-deterministic. Compare configurations by averages over the full set, and treat differences within noise as ties.
- **Gate deploys on thresholds you chose deliberately.** Fail CI when hit rate or faithfulness drops beyond an agreed margin, not when a single score wiggles.
- **Sample production traffic.** Offline sets go stale. Score a sampled slice of real queries continuously, and feed the bad ones back into the golden set.

> A metric you haven't validated against human judgment isn't measurement. It's a number with good branding.

---
*Forward this to whoever last said "the answers look better now" without showing a before-and-after number.*