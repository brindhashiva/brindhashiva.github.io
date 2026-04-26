Title: Hybrid Search in RAG Systems
Date: 2026-04-26
Category: GenAI
Tags: RAG, Hybrid Search, BM25, Vector Search, LangChain, Retrieval
Slug: hybrid-search-in-rag-systems
Status: published

Pure vector search has a quiet failure mode: it ranks by meaning, so the exact tokens users actually paste into a search box (error codes, SKUs, function names, version strings) get smoothed into "something vaguely technical." Keyword search has the opposite blind spot: it can't connect "car" to "automobile." Hybrid search runs both and fuses the rankings, and it is the sane baseline for retrieval before you reach for anything fancier. The catch is that "just combine them" hides three real decisions: how to fuse, how to weight, and where the keyword index lives.

## Two Retrievers, Two Blind Spots

**Dense retrieval** — Embeds the query and every chunk into vectors and ranks by similarity. It handles paraphrase and intent well ("reset my login" matches "password recovery steps"), but it is weak on rare identifiers, because the embedding of `ERR_2043` lands near every other error-code-shaped string.

**Sparse retrieval (BM25)** — Scores chunks by how often the query terms appear, how rare those terms are across the corpus, and the chunk's length. Exact matches on rare tokens dominate. It has no notion of synonyms, and it is only as good as your tokenization.

**Reciprocal Rank Fusion (RRF)** — Merges ranked lists using ranks only: each document earns `1 / (k + rank)` from every list it appears in, and the contributions are summed. Because it ignores raw scores, you never have to normalize BM25 scores against cosine similarities, which live on completely different scales.

## Choosing a Fusion Strategy

The fusion step is where most hybrid implementations quietly go wrong. Your options:

- **RRF** — Rank-based, no normalization, robust default. The downside: it discards score magnitude, so a dominant #1 and a marginal #1 look identical.
- **Weighted score blending** — Keeps magnitude, but you must normalize each list per query (min-max or z-score), and normalization is fragile when the candidate set is small.
- **Learned fusion or a cross-encoder reranker** — Best quality, but needs labeled data or an extra model call. That is a separate topic (the next post covers it).

RRF is short enough to write yourself, which is worth doing once so you know exactly what your framework is doing:

```python
from collections import defaultdict

def rrf(rankings: list[list[str]], k: int = 60) -> list[tuple[str, float]]:
    """Fuse ranked lists of doc IDs. Higher score = better."""
    scores: dict[str, float] = defaultdict(float)
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] += 1.0 / (k + rank)
    return sorted(scores.items(), key=lambda kv: kv[1], reverse=True)

bm25_ids   = ["d7", "d2", "d9", "d4"]
vector_ids = ["d2", "d5", "d7", "d1"]
print(rrf([bm25_ids, vector_ids])[:3])
# d2 and d7 rise to the top: both retrievers agree on them.
```

The constant `k` (60 is the conventional value) damps the influence of top ranks. Raising it flattens the difference between rank 1 and rank 5; lowering it makes the top of each list count for more.

## Implementing It in LangChain

One migration note first: since LangChain 1.0, the legacy `langchain.retrievers` module is gone and `EnsembleRetriever` lives in the `langchain-classic` package. Older tutorials that import from `langchain.retrievers` will raise `ModuleNotFoundError`.

```python
# pip install langchain-classic langchain-community langchain-openai faiss-cpu rank_bm25
from langchain_classic.retrievers.ensemble import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_community.vectorstores import FAISS
from langchain_core.documents import Document
from langchain_openai import OpenAIEmbeddings

docs = [
    Document(page_content="ERR_2043: connection pool exhausted. Increase max_connections.",
             metadata={"id": "kb-101"}),
    Document(page_content="If you cannot sign in, use the password recovery flow.",
             metadata={"id": "kb-102"}),
    Document(page_content="Rotate API credentials from the security settings page.",
             metadata={"id": "kb-103"}),
]

bm25 = BM25Retriever.from_documents(docs, k=5)

vector = FAISS.from_documents(
    docs, OpenAIEmbeddings(model="text-embedding-3-small")
).as_retriever(search_kwargs={"k": 5})

hybrid = EnsembleRetriever(
    retrievers=[bm25, vector],
    weights=[0.5, 0.5],
    id_key="id",          # dedupe on metadata["id"] instead of page_content
)

for d in hybrid.invoke("ERR_2043"):
    print(d.metadata["id"], d.page_content[:60])
```

<!-- VERIFY before publishing: EnsembleRetriever's RRF constant `c` — I believe the default is 60, but confirm in the langchain_classic reference for your installed version. -->

Under the hood, `EnsembleRetriever` runs each retriever and fuses the results with weighted RRF. The `weights` scale each retriever's RRF contribution; they are not score weights.

## Where Hybrid Search Goes Wrong

- **Tokenization.** BM25's default preprocessing splits on whitespace, which mangles `get_user_by_id`, `v2.3.1`, and `ERR_2043`. If identifiers matter in your domain, pass a custom `preprocess_func` to `BM25Retriever` and test that your identifiers survive intact.
- **Mismatched chunks.** Both retrievers must index the same chunks under the same IDs. If they don't, fusion can't deduplicate and the same passage shows up twice with different IDs.
- **A frozen 50/50 split.** Equal weights are a starting point, not a tuned value. The right balance depends on your query mix: identifier-heavy queries want more BM25, conversational ones want more dense. Measure it (see the evaluation post in this series).
- **In-memory BM25 doesn't scale.** `BM25Retriever` builds its index in process, which is fine for prototypes and painful for large or frequently updated corpora. In production, prefer your search engine or vector database's native hybrid query, and check what your store supports before wiring this up by hand.
- **Too-small candidate pools.** Fetch generously from each retriever (20-50) before fusion, then trim. Fusing two top-3 lists throws away exactly the documents that fusion exists to surface.

> Hybrid search isn't about combining two retrievers. It's admitting your embedding model has a blind spot for the exact strings users copy and paste.

---
*If someone on your team is still shipping RAG on pure vector search, send them this before the next incident review.*