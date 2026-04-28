Title: Building Production-Ready RAG Applications
Date: 2026-04-28
Category: GenAI
Tags: RAG, Production, LangGraph, LangChain, Architecture, Observability
Slug: building-production-ready-rag-applications
Status: published

A RAG demo needs one happy path: embed a few documents, retrieve, generate. A production system needs to survive stale documents, users who shouldn't see each other's data, retrieval that returns nothing useful, and a provider that times out at 2 a.m. The gap between the two isn't a better prompt; it's everything around the model call. This post is a reference for that gap: what breaks, a working LangGraph skeleton that handles the worst of it, and the operational checklist that keeps it standing.

## What Breaks Between Demo and Production

**Ingestion pipeline** — The unglamorous half of RAG: parsing, cleaning, chunking, embedding, and writing to the index. Most "the model got it wrong" bugs trace back here: a PDF table flattened into nonsense, a boilerplate footer embedded 10,000 times, headings stripped from chunks that no longer say what they're about.

**Idempotent indexing** — Re-running ingestion must not create duplicates. Give every chunk a deterministic ID (for example a hash of source, section, and content) and upsert. Without this, your corpus slowly accretes stale and duplicate chunks, and your retriever starts returning the same paragraph three times.

**Chunk metadata** — Source, document version, updated-at, tenant or ACL fields. Metadata is what makes citation, freshness filtering, and access control possible. Retrofitting it later means re-indexing everything.

**Grounded refusal** — The behavior of saying "I couldn't find this" when retrieval comes back empty or weak. It is a first-class code path, not a prompt hope.

## A Reference Skeleton in LangGraph

This wires the pieces from the previous posts (hybrid retrieval, reranking) into a graph with tenant-scoped retrieval and an explicit refusal branch. It assumes a `vector_store` and a `compressor` (the reranker) like the ones built earlier in the series.

```python
# pip install langgraph langchain langchain-openai
import os
from typing import TypedDict

from langchain.chat_models import init_chat_model
from langchain_core.documents import Document
from langchain_core.prompts import ChatPromptTemplate
from langgraph.graph import StateGraph, START, END

llm = init_chat_model(
    os.environ["CHAT_MODEL"],
    model_provider=os.environ["CHAT_PROVIDER"],
)

prompt = ChatPromptTemplate.from_messages([
    ("system",
     "Answer using ONLY the context below. If the context does not contain "
     "the answer, say you don't know. Cite sources as [source_id].\n\n"
     "Context:\n{context}"),
    ("human", "{question}"),
])

MIN_RELEVANCE = 0.3   # tune per reranker; scores are model-specific

class RAGState(TypedDict):
    question: str
    tenant_id: str
    context: list[Document]
    answer: str

def retrieve(state: RAGState) -> dict:
    # Filter at query time, inside the index. Never fetch first and
    # filter after: that leaks other tenants' data into your candidate pool.
    # Filter syntax differs per vector store; this is the generic shape.
    candidates = vector_store.similarity_search(
        state["question"], k=40, filter={"tenant_id": state["tenant_id"]}
    )
    reranked = compressor.compress_documents(candidates, state["question"])
    strong = [
        d for d in reranked
        if d.metadata.get("relevance_score", 0.0) >= MIN_RELEVANCE
    ]
    return {"context": strong}

def route(state: RAGState) -> str:
    return "generate" if state["context"] else "refuse"

def generate(state: RAGState) -> dict:
    context = "\n\n".join(
        f"[{d.metadata['id']}] {d.page_content}" for d in state["context"]
    )
    msg = prompt.invoke({"context": context, "question": state["question"]})
    return {"answer": llm.invoke(msg).content}

def refuse(state: RAGState) -> dict:
    return {"answer": "I couldn't find that in the knowledge base."}

builder = StateGraph(RAGState)
builder.add_node("retrieve", retrieve)
builder.add_node("generate", generate)
builder.add_node("refuse", refuse)
builder.add_edge(START, "retrieve")
builder.add_conditional_edges(
    "retrieve", route, {"generate": "generate", "refuse": "refuse"}
)
builder.add_edge("generate", END)
builder.add_edge("refuse", END)

graph = builder.compile()

result = graph.invoke({"question": "How do I rotate API keys?", "tenant_id": "acme"})
print(result["answer"])
```

<!-- VERIFY before publishing: I confirmed the StateGraph / add_conditional_edges / init_chat_model shapes against current LangChain and LangGraph docs, but `relevance_score` in Document metadata is set by CohereRerank as far as I know. If you use a different compressor, check what it writes to metadata. -->

Two design choices are worth defending. First, the refusal branch is a graph edge rather than a sentence in the prompt, so it is testable and observable. Second, the relevance gate sits between retrieval and generation, which turns "the model hallucinated" into "retrieval found nothing, and we said so."

## Pipeline vs. Agentic RAG

- **Fixed pipeline (retrieve, then generate)** — Predictable latency and cost, easy to evaluate, easy to debug. The right default for support bots, docs Q&A, and anything where every query needs the knowledge base.
- **Agentic RAG (the model decides when and what to retrieve)** — Handles multi-step questions, follow-ups, and queries that don't need retrieval at all. Trade-off: variable latency, more failure modes, and evaluation gets harder because the retrieval calls are now model-chosen. LangChain's own retrieval-agent tutorial builds this in LangGraph with a retriever tool.
- **Hybrid of both** — Run a fixed pipeline by default and escalate to an agent loop only when the first pass comes back weak. This is often the best cost-to-capability ratio.

Start with the pipeline. Promote to an agent when your evals show queries a single retrieval pass cannot answer.

## The Production Checklist

- **Access control at retrieval time.** Filter inside the index by tenant or ACL metadata. Post-filtering is a data-leak bug with good manners.
- **Version and freshness.** Store document versions and updated-at, then prefer current versions in retrieval. Stale-but-confident answers are the failure users notice last and trust least.
- **Timeouts, retries, and fallbacks.** Every network hop (embedding, vector store, reranker, LLM) needs a timeout and a plan: retry with backoff, then degrade (skip the reranker, fall back to a smaller model) rather than 500.
- **Trace each stage separately.** Log the query, candidates, post-rerank scores, and final context for every request. When an answer is wrong, you need to know which stage failed, and end-to-end logging can't tell you.
- **Cache deliberately.** Embeddings of repeated queries and reranker calls on identical (query, candidate) pairs are cheap wins. Cache generation only if your answers are truly query-determined.
- **Regression-test with an eval set.** Gate deploys on retrieval and answer quality, not vibes (covered in the next post).

> Your RAG system's most dangerous failure isn't hallucination. It's a confident, well-cited answer from last quarter's version of the document.

---
*Sending this to the engineer who's about to say "we'll add access control after the pilot" would be a kindness.*