Title: LangGraph for Complex Agent Workflows
Date: 2026-05-15
Category: GenAI
Tags: LangGraph, AI Agents, State Machines, Subgraphs, Workflow Orchestration
Slug: langgraph-for-complex-agent-workflows
Status: published

Every LangGraph example in this series so far, RAG pipelines, multi-agent supervisors, plan-and-execute, human-in-the-loop gates, used the same three primitives: state, nodes, and edges. That's not a coincidence. Complex agent workflows aren't complex because the primitives are more elaborate; they're complex because those three primitives get composed more deeply: subgraphs nested inside supervisors, parallel branches that fan out and back in, and state schemas that carry real structure instead of a flat message list. This post is about that composition.

## Composing Graphs as Subgraphs

A graph built earlier can be embedded as a single node inside a larger graph, which is how the multi-agent supervisor pattern (from earlier in this series) actually works under the hood: each worker agent is itself a compiled graph, invoked as one node from the supervisor's perspective.

```python
# pip install langgraph
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# A self-contained subgraph: validates and enriches an order.
class OrderState(TypedDict):
    order_id: str
    validated: bool
    enriched: dict

def validate(state: OrderState) -> dict:
    return {"validated": order_exists(state["order_id"])}

def enrich(state: OrderState) -> dict:
    return {"enriched": fetch_order_details(state["order_id"])}

order_subgraph = StateGraph(OrderState)
order_subgraph.add_node("validate", validate)
order_subgraph.add_node("enrich", enrich)
order_subgraph.add_edge(START, "validate")
order_subgraph.add_edge("validate", "enrich")
order_subgraph.add_edge("enrich", END)
compiled_order_graph = order_subgraph.compile()

# A parent graph that uses the subgraph as one step among others.
class SupportState(TypedDict):
    order_id: str
    validated: bool
    enriched: dict
    response: str

parent = StateGraph(SupportState)
parent.add_node("order_check", compiled_order_graph)   # a compiled graph, used as a node
parent.add_node("respond", lambda s: {"response": build_response(s["enriched"])})
parent.add_edge(START, "order_check")
parent.add_edge("order_check", "respond")
parent.add_edge("respond", END)

graph = parent.compile()
```

<!-- VERIFY before publishing: passing a compiled StateGraph directly as a node works when the parent and subgraph state schemas share compatible keys; for schemas that diverge, current LangGraph docs use a wrapper function that translates between parent and subgraph state instead. Confirm the exact requirement against your pinned version before relying on the direct-embedding form shown here. -->

This is the actual mechanism that made the multi-agent supervisor post's code work: `create_agent`'s output is a compiled graph, and `create_supervisor` wires several of them together as nodes in one larger graph, exactly like `compiled_order_graph` above.

## Fan-Out and Fan-In

The multi-source retrieval post earlier in this series used parallel node execution to query several sources at once. The same pattern generalizes to any workflow where independent branches should run concurrently and their results merge back into shared state:

```python
import operator
from typing import Annotated

class ReviewState(TypedDict):
    document: str
    findings: Annotated[list[str], operator.add]   # each branch appends independently

def check_grammar(state: ReviewState) -> dict:
    return {"findings": [grammar_check(state["document"])]}

def check_facts(state: ReviewState) -> dict:
    return {"findings": [fact_check(state["document"])]}

def check_tone(state: ReviewState) -> dict:
    return {"findings": [tone_check(state["document"])]}

builder = StateGraph(ReviewState)
for name, fn in [("grammar", check_grammar), ("facts", check_facts), ("tone", check_tone)]:
    builder.add_node(name, fn)
    builder.add_edge(START, name)   # all three start in parallel from START
    builder.add_edge(name, "summarize")

builder.add_node("summarize", lambda s: {"findings": s["findings"]})  # fan-in point
builder.add_edge("summarize", END)

graph = builder.compile()
```

Three edges from `START` means all three checks launch concurrently rather than sequentially; the `operator.add` reducer on `findings` is what lets them write to shared state without overwriting each other, and `summarize` only runs once every branch has completed.

## Conditional Routing Beyond a Simple If

The RAG production post's retrieve-or-refuse branch was a two-way fork. Real workflows often need routing across more than two outcomes, and a `Literal` return type keeps that routing checkable:

```python
from typing import Literal

def classify_intent(state: SupportState) -> dict:
    return {"intent": intent_classifier.invoke(state["message"])}

def route_by_intent(state: SupportState) -> Literal["billing", "technical", "general", "escalate"]:
    return state["intent"]   # one of the four literal values, or a routing bug you'll catch in testing

builder.add_conditional_edges(
    "classify_intent",
    route_by_intent,
    {"billing": "billing_agent", "technical": "tech_agent",
     "general": "general_agent", "escalate": "human_review"},
)
```

The explicit mapping (a dict rather than letting the function's return value be used directly as the node name) is worth keeping even when the values already match node names: it's what turns a routing typo into an immediate, obvious error rather than a silent no-op.

## Cycles: Loops That Terminate on Purpose

A workflow that revises a draft until a reviewer is satisfied is a cycle, not a straight line, and LangGraph handles this the same way any graph does: an edge that points back to an earlier node, gated by a condition that eventually resolves to "stop."

```python
class DraftState(TypedDict):
    draft: str
    feedback: str
    revision_count: int

def write_draft(state: DraftState) -> dict:
    return {"draft": writer_llm.invoke(state.get("feedback", "")), "revision_count": state.get("revision_count", 0) + 1}

def review_draft(state: DraftState) -> dict:
    return {"feedback": reviewer_llm.invoke(state["draft"])}

def should_continue(state: DraftState) -> str:
    if "approved" in state["feedback"].lower():
        return "done"
    if state["revision_count"] >= 5:     # hard cap: never loop forever
        return "done"
    return "revise"

builder.add_conditional_edges("review_draft", should_continue, {"revise": "write_draft", "done": END})
```

The hard cap on `revision_count` is not optional. Every cycle in a graph needs an explicit, reachable exit condition that doesn't depend solely on the model's judgment, because a model that never says "approved" turns an elegant revision loop into the unbounded-loop failure mode from the agent-failures post earlier in this series.

## When the Complexity Is Actually Justified

Every pattern here adds real cost: more state to reason about, more paths to test, more surface for the reliability patterns from two posts ago to need applying. Reach for them specifically when:

- **Subgraphs** — When a piece of logic is genuinely reusable across multiple parent workflows, or when its internal state shouldn't leak into the parent's, not merely to organize code that's only used once.
- **Fan-out/fan-in** — When branches are truly independent (no branch needs another branch's result to start) and the latency savings of parallel execution justify the added state-merging complexity.
- **Multi-way routing** — When the number of outcomes is fixed and known, not as a substitute for an agent that should be deciding its own next step dynamically (that's the tool-calling and ReAct patterns from earlier posts, not graph routing).
- **Cycles** — Only with an explicit, non-model-dependent exit condition. A cycle without a hard cap isn't a workflow feature, it's the agent-failure pattern from two posts ago waiting to happen.

> A complex graph isn't a sign you've overengineered something. A complex graph with no exit conditions, no fallback branches, and no subgraph boundaries is.

---
*If your team's LangGraph app is one giant flat node list with no subgraphs or fan-out, this is worth a look before it gets much bigger.*