Title: Multi-Source Retrieval Architectures
Date: 2026-05-01
Category: GenAI
Tags: RAG, Retrieval Architecture, Federation, Routing, LangChain, LangGraph
Slug: multi-source-retrieval-architectures
Status: published

Real knowledge bases are rarely one vector store. There's a docs corpus, a ticketing system, a SQL database, maybe a graph. Once a second source shows up, "just retrieve" stops being an answer and becomes a design question: query everything every time, or figure out where the answer probably lives first? Get this wrong and you either burn latency and money hitting sources that never had the answer, or you silently miss the one source that did.

## The Core Trade-off

**Fan-out (scatter-gather)** — Query every source in parallel, merge the results. Simple, robust to routing mistakes, and it degrades gracefully if one source is empty. The cost is linear in the number of sources: five sources means five queries (and five bills) per user question, most of which return nothing useful.

**Routing** — Classify the query first, then send it only to the source(s) that are likely to have the answer. Cheaper and faster, but now routing accuracy is a new failure mode sitting in front of retrieval, and a bad route means a confidently wrong "I don't know."

**Federation** — A middle ground: route to a shortlist of plausible sources rather than one, then fan out only across that shortlist. Most production multi-source systems converge here once they've measured that pure fan-out is too slow and pure routing is too brittle.

## Building a Router

The router is usually the cheapest, highest-leverage piece: a small, fast classification step, not a full agent loop. Structured output on a lightweight model keeps it fast and testable.

```python
# pip install langchain langchain-openai pydantic
from typing import Literal
from pydantic import BaseModel, Field
from langchain.chat_models import init_chat_model

class RouteDecision(BaseModel):
    sources: list[Literal["docs", "tickets", "sql", "graph"]] = Field(
        description="Which sources are plausibly relevant. Pick 1-2 unless "
                    "the question clearly spans more."
    )
    reasoning: str = Field(description="One sentence justification.")

router_llm = init_chat_model("gpt-4o-mini", model_provider="openai")
structured_router = router_llm.with_structured_output(RouteDecision)

def route(question: str) -> RouteDecision:
    return structured_router.invoke(
        "Classify which knowledge sources are relevant to this question.\n"
        "- docs: product documentation and how-to guides\n"
        "- tickets: past support tickets and resolutions\n"
        "- sql: structured account, usage, and billing data\n"
        "- graph: entity relationships (org structure, dependencies)\n\n"
        f"Question: {question}"
    )

decision = route("Why was my invoice higher this month than last month?")
print(decision.sources)   # likely ["sql"], maybe ["sql", "tickets"]
```

`with_structured_output` builds on tool calling under the hood, so this works across providers that support tool calls without you writing a parser.

## Building the Federated Graph

LangGraph is a natural fit here: retrieval from each source is a node, and the router decides which nodes actually run. This wires the router above to three source-specific retrieval nodes and merges whatever comes back.

```python
# pip install langgraph
import operator
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_core.documents import Document

class FederatedState(TypedDict):
    question: str
    sources: list[str]
    results: Annotated[list[Document], operator.add]   # nodes append in parallel

def plan(state: FederatedState) -> dict:
    return {"sources": route(state["question"]).sources}

def fan_out(state: FederatedState) -> list[str]:
    # One conditional edge per selected source; unselected nodes never run.
    return [s for s in state["sources"] if s in {"docs", "tickets", "sql"}]

def search_docs(state: FederatedState) -> dict:
    return {"results": docs_retriever.invoke(state["question"])}

def search_tickets(state: FederatedState) -> dict:
    return {"results": tickets_retriever.invoke(state["question"])}

def search_sql(state: FederatedState) -> dict:
    return {"results": sql_retriever.invoke(state["question"])}

builder = StateGraph(FederatedState)
builder.add_node("plan", plan)
builder.add_node("docs", search_docs)
builder.add_node("tickets", search_tickets)
builder.add_node("sql", search_sql)
builder.add_edge(START, "plan")
builder.add_conditional_edges("plan", fan_out, ["docs", "tickets", "sql"])
for node in ("docs", "tickets", "sql"):
    builder.add_edge(node, END)

graph = builder.compile()
```

<!-- VERIFY before publishing: add_conditional_edges accepting a function that returns a list of node names (fan-out to multiple branches) is current LangGraph behavior for parallel execution, but confirm the exact signature against the LangGraph version you pin, since conditional-edge fan-out semantics have shifted across releases. -->

Each selected source runs as its own node, so they execute in parallel and write into the shared `results` list via the `operator.add` reducer. Nothing downstream needs to know which sources actually ran.

## The Fusion Problem Gets Harder

Combining results across heterogeneous sources is not the same problem as fusing two rankings of the same document type (that's the earlier hybrid-search post). Here you're fusing:

- **Different relevance signals** — vector similarity from docs, recency and status from tickets, raw rows from SQL. There's no shared score to normalize against.
- **Different shapes** — a ticket has a resolution field, a SQL result is tabular, a doc chunk is prose. You often can't rank them on one list; you present them as labeled sections instead.
- **Different confidence in "nothing found."** An empty SQL result is a definitive fact. An empty vector search just means nothing scored above threshold. Treat these differently when you decide whether to answer or refuse.

A practical default: don't force a single fused ranking across sources. Keep each source's results attached to their source, cap how many each contributes to the prompt, and let the generation step reason over labeled sections rather than a merged, unlabeled list.

## When to Reach for This

- **Single source, one shape of question** — Skip all of this. A router adds latency and a new failure mode for no benefit.
- **A few sources, clearly separable by topic** ("billing questions" vs "how-to questions") — Routing is a clean win; the classifier's job is easy and errors are cheap to catch in eval.
- **Sources that frequently overlap** (the same fact lives in docs and in a ticket) — Lean toward federation over strict routing; the cost of fanning out to two sources beats the cost of a wrong single route.
- **Questions that genuinely need multiple sources to answer** ("why did my usage spike, and is that expected per the docs?") — You need federation or an agentic loop that can call sources sequentially, not a one-shot router.

> The hard part of multi-source retrieval was never adding a second source. It's admitting your relevance signals no longer live on the same scale.

---
*Pass this to whoever's about to bolt a fourth data source onto a RAG pipeline that only ever routes to the first one.*