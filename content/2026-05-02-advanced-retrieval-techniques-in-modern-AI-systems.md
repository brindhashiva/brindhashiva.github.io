Title: Advanced Retrieval Techniques in Modern AI Systems
Date: 2026-05-02
Category: GenAI
Tags: RAG, Query Rewriting, HyDE, Multi-Query, Parent-Document Retrieval, LangChain
Slug: advanced-retrieval-techniques-in-modern-ai-systems
Status: published

Hybrid search and reranking (covered earlier in this series) fix how you score candidates. They don't fix a bad query. A user typing "why is it slow sometimes" gives your retriever almost nothing to match against, no matter how good your fusion and reranking are. The techniques here operate upstream of retrieval and scoring: they change the query itself, or change what a "chunk" means, before any similarity math happens.

## Rewriting the Query

**Query rewriting** — Using an LLM to reformulate the user's question into a form that retrieves better, before it ever touches the index. Covers expanding abbreviations, resolving pronouns against conversation history, and stating implicit intent explicitly.

**Multi-query retrieval** — Generate several rephrasings of the same question, retrieve for each, and union the results. This hedges against any single phrasing missing the vocabulary your documents actually use.

**HyDE (Hypothetical Document Embeddings)** — Ask an LLM to write a hypothetical answer to the question, then embed that answer instead of the question, and search with it. The insight: answers and documents share vocabulary and phrasing far more than questions and documents do, so embedding a fake answer often lands closer to real answers than embedding the question itself.

**Step-back prompting** — Ask a broader, more general version of the question first ("what is connection pooling?" before "why does ERR_2043 happen under load?"), retrieve for both, and use the general context to help interpret the specific one. Useful when a question needs background the literal query never mentions.

## Multi-Query in LangChain

`MultiQueryRetriever` moved to `langchain-classic` in the LangChain 1.0 reorganization, same as the retrievers covered in earlier posts:

```python
# pip install langchain-classic langchain-openai
import logging
from langchain_classic.retrievers.multi_query import MultiQueryRetriever
from langchain.chat_models import init_chat_model

# See "logging" below: this lets you inspect the generated queries.
logging.basicConfig()
logging.getLogger("langchain_classic.retrievers.multi_query").setLevel(logging.INFO)

llm = init_chat_model("gpt-4o-mini", model_provider="openai")

multi_query = MultiQueryRetriever.from_llm(
    retriever=base_vector_retriever,   # any retriever from earlier posts
    llm=llm,
)

docs = multi_query.invoke("why is it slow sometimes")
# INFO logs typically show something like:
# Generated queries: ['What causes intermittent slowness in the system?',
#                      'What are common performance bottlenecks?',
#                      'Under what conditions does latency spike?']
```

`from_llm` uses a default prompt to generate query variants; pass your own `PromptTemplate` via `prompt_key` if your domain needs different phrasing (e.g. emphasizing product-specific vocabulary).

## HyDE, Written by Hand

There's no single blessed HyDE class across LangChain versions, and it's short enough that writing it directly is more reliable than chasing whichever wrapper is current:

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import OpenAIEmbeddings

hyde_prompt = ChatPromptTemplate.from_template(
    "Write a short, plausible-sounding passage that would answer this "
    "question, as if it came from technical documentation. Do not hedge "
    "or say you're unsure.\n\nQuestion: {question}"
)

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

def hyde_search(question: str, vector_store, k: int = 10):
    hypothetical_doc = llm.invoke(hyde_prompt.invoke({"question": question})).content
    hyde_vector = embeddings.embed_query(hypothetical_doc)
    return vector_store.similarity_search_by_vector(hyde_vector, k=k)

results = hyde_search("why does the connection pool fill up under load?", my_vector_store)
```

The hallucinated passage never reaches the user; it exists only to produce a better embedding. That said, HyDE adds a full LLM call before retrieval even starts, so measure it against your latency budget, and expect it to underperform plain retrieval on queries that are already well-specified (it mainly helps vague or underspecified ones).

## Restructuring What a Chunk Is

**Parent-document retrieval** — Embed small chunks for precise matching, but return their larger parent chunk (or the whole source document) to the model. This resolves the classic RAG tension: small chunks embed accurately but lack context; large chunks carry context but embed mushily. `ParentDocumentRetriever` (also in `langchain-classic` now) handles the split-and-map bookkeeping:

```python
# pip install langchain-classic langchain-text-splitters
from langchain_classic.retrievers import ParentDocumentRetriever
from langchain_classic.storage import InMemoryStore
from langchain_text_splitters import RecursiveCharacterTextSplitter

parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=0)

retriever = ParentDocumentRetriever(
    vectorstore=my_vector_store,     # indexes child chunks
    docstore=InMemoryStore(),        # holds parent chunks, keyed by ID
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

retriever.add_documents(source_documents)
# Search matches against the small, precise child chunks...
docs = retriever.invoke("connection pool exhaustion")
# ...but returns the larger parent chunks that give the model real context.
```

<!-- VERIFY before publishing: confirm the langchain_classic.storage import path for InMemoryStore against your pinned version; some releases keep storage helpers under langchain_core or langchain_community instead. -->

**Sentence-window retrieval** — A lighter alternative to full parent-document retrieval: embed individual sentences, but when one matches, expand the returned context to include a fixed window of surrounding sentences. Cheaper to implement than parent-document retrieval and a reasonable default when your source documents don't have a clean logical hierarchy to split on.

## Picking the Right Technique

- **Vague, conversational queries** ("why is it slow") — Multi-query or query rewriting against conversation history. The problem is missing vocabulary, and generating variants directly addresses that.
- **Queries phrased very differently from your documents' style** (user jargon vs. formal documentation) — HyDE. The mismatch is stylistic, and HyDE closes it by embedding in the target style.
- **Chunks that are either too small to embed meaningfully or too large to be precise** — Parent-document or sentence-window retrieval. The problem isn't the query; it's what a "unit" of retrieval means for your content.
- **Questions needing background the query never states** — Step-back prompting, paired with whatever base retriever you already have.

None of these are free. Multi-query and HyDE both add LLM calls before retrieval starts, multiplying your latency and cost per query. Measure the win on your eval set (from the evaluation post earlier in this series) before shipping any of them; on well-specified, in-domain queries, plain hybrid search plus reranking often already wins.

> Every one of these techniques is a way of admitting the user's question is not the query your index actually needs.

---
*Send this to whoever's about to tune chunk sizes for the fifth time instead of fixing the query.*