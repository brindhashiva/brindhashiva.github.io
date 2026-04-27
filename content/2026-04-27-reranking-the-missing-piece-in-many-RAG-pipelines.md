Title: Reranking: The Missing Piece in Many RAG Pipelines
Date: 2026-04-27
Category: GenAI
Tags: RAG, Reranking, Cross-Encoder, Cohere, LangChain, Retrieval
Slug: reranking-the-missing-piece-in-many-rag-pipelines
Status: published

Retrieval gives you candidates; reranking gives you an order. That distinction matters because an LLM's answer is sensitive to what sits at the top of its context, and embedding similarity is a coarse instrument for deciding what belongs there. The cheapest quality upgrade in most RAG pipelines is not a bigger model or a fancier chunker. It is retrieving wide and reranking narrow. Yet many production pipelines still pass the raw top-k from a vector search straight into the prompt.

## How Reranking Works

**Bi-encoder** — Embeds the query and each document independently, which is what lets you precompute an index and run approximate nearest-neighbor search over millions of chunks. The cost is that query and document never "see" each other, so fine-grained relevance gets averaged away.

**Cross-encoder** — Reads the query and a candidate document together in a single forward pass and outputs a relevance score. It is materially more accurate because every query token can attend to every document token, but you can't run it over a whole corpus. That is why it lives in the second stage.

**Two-stage retrieval** — Stage one (vector, BM25, or hybrid) retrieves a wide candidate pool, typically 50-100 chunks. Stage two (the reranker) rescores those and keeps the few, typically 3-8, that go into the prompt.

**Relevance score** — A model-specific number, not a probability. Scores from one reranker are not comparable to another's, so any threshold you hard-code has to be re-tuned when you swap models.

## Three Ways to Rerank

- **Hosted reranker API (e.g. Cohere Rerank)** — Least effort, strong quality, multilingual. Cohere's current generation, Rerank 4, ships as `rerank-v4.0-pro` (quality-focused) and `rerank-v4.0-fast` (latency-focused), with a 32k-token context window versus 4k on Rerank 3.5. Trade-off: your text leaves your infrastructure, and you pay per call.
- **Self-hosted cross-encoder (e.g. BGE rerankers)** — Full control over data, cost, and latency profile. Trade-off: you own the serving, batching, and hardware, and a cross-encoder on CPU can eat your latency budget quickly.
- **LLM-as-reranker** — Prompt a general model to score or order candidates. Most flexible (you can encode business rules in the prompt), but slowest, most expensive, and least deterministic. Reach for it only when a purpose-built reranker can't express your relevance criteria.

## Code: Three Implementations

Calling Cohere directly:

```python
# pip install cohere
import cohere

co = cohere.ClientV2()   # reads CO_API_KEY from the environment

query = "How do I rotate API credentials?"
candidates = [
    "Rotate API credentials from the security settings page.",
    "Our pricing tiers are listed on the billing page.",
    "Credential rotation invalidates old keys after a 24-hour grace period.",
]

resp = co.rerank(
    model="rerank-v4.0-fast",
    query=query,
    documents=candidates,
    top_n=2,
)
for r in resp.results:
    print(round(r.relevance_score, 3), candidates[r.index])
```

Plugging a reranker into a LangChain retriever. `ContextualCompressionRetriever` now lives in `langchain-classic` (the old `langchain.retrievers` path was removed in 1.0), and `CohereRerank` requires you to name the model explicitly:

```python
# pip install langchain-classic langchain-cohere
from langchain_classic.retrievers.contextual_compression import (
    ContextualCompressionRetriever,
)
from langchain_cohere import CohereRerank

base_retriever = hybrid   # from the previous post; make sure k is generous (e.g. 30-50)

compressor = CohereRerank(model="rerank-v4.0-fast", top_n=5)

reranked = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever,
)

docs = reranked.invoke("How do I rotate API credentials?")
```

<!-- VERIFY before publishing: LangChain's own Cohere integration page still shows model="rerank-english-v3.0" in its example. I confirmed the Rerank 4 model names against Cohere's changelog, not against langchain_cohere's runtime validation, so run this once with a real key. -->

And the self-hosted route with an open cross-encoder:

```python
# pip install langchain-classic langchain-community sentence-transformers
from langchain_classic.retrievers.contextual_compression import (
    ContextualCompressionRetriever,
)
from langchain_classic.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

model = HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-base")
compressor = CrossEncoderReranker(model=model, top_n=5)

reranked = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever,
)
```

## Tuning It Properly

- **Respect the recall ceiling.** A reranker can only reorder what retrieval hands it. If the right chunk isn't in your top 50, no cross-encoder will conjure it. Check recall of your first stage at the candidate-pool size before blaming the reranker.
- **Budget the latency.** Reranking cost scales with the number of candidates and their length. Measure p95 at your real pool size, and cap chunk length before scoring.
- **Pick `top_n` from evidence.** More context is not free: it dilutes the prompt and costs tokens. Sweep `top_n` against your eval set rather than defaulting to "5."
- **Use scores to abstain.** A low top relevance score is a useful signal that the knowledge base doesn't contain the answer. Route those queries to a refusal or a clarifying question instead of letting the model improvise, but calibrate the threshold per model.
- **Re-tune after every model swap.** Scores shift between reranker versions. A threshold tuned on one model silently changes meaning on the next.

> A reranker doesn't fix bad retrieval; it fixes bad ordering. If your first stage misses the answer, you've built an expensive way to sort the wrong documents.

---
*Know a team whose RAG answers "almost" work? Send them this and ask what their retrieval pool size is.*