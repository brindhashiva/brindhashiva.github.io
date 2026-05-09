Title: Human-in-the-Loop Agent Systems
Date: 2026-05-09
Category: GenAI
Tags: AI Agents, Human-in-the-Loop, LangGraph, Interrupts, Approval Workflows
Slug: human-in-the-loop-agent-systems
Status: published

An agent that can send emails, place orders, or run database queries is only as safe as its ability to stop and ask before doing something irreversible. Human-in-the-loop (HITL) isn't a UI feature bolted on afterward; it's a control-flow primitive, and in LangGraph it reduces to exactly two operations repeated as needed: pause, and resume with input. Everything else, approval gates, editable state, multi-turn clarification, is that same primitive applied at different points in a graph.

## The Primitive

**Interrupt** — A function call inside a graph node that pauses execution at that exact point, persists the graph's state via its checkpointer, and surfaces a value to whatever is running the graph. Unlike a static breakpoint set between nodes, `interrupt()` can sit anywhere inside a node's code and can be called conditionally, so a tool call gets reviewed only when it actually needs review.

**Resume** — Re-invoking the graph with `Command(resume=<value>)`, which becomes the return value of the `interrupt()` call inside the node, letting execution continue exactly where it left off.

**Checkpointer dependency** — Interrupts require a checkpointer to be compiled into the graph. Without one, there's no persisted state to pause and resume from, and this is the single most common reason HITL code fails silently: everything looks right until the first interrupt, which has nothing to pause into.

One mechanical detail worth internalizing early: on resume, the node restarts from its beginning, re-executing any code before the `interrupt()` call. Side effects placed before an interrupt inside the same node will run twice across a pause-and-resume cycle unless you make them idempotent or move them to a separate node.

## Approval Before a Risky Tool Call

The most common HITL pattern by a wide margin: the agent decides to do something real, execution pauses, a human approves or rejects, and only then does the action actually happen.

```python
# pip install langgraph
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.types import interrupt, Command

def review_tool_call(state: MessagesState) -> dict:
    last_message = state["messages"][-1]
    tool_call = last_message.tool_calls[0]

    decision = interrupt({
        "action": tool_call["name"],
        "args": tool_call["args"],
        "question": f"Approve {tool_call['name']}({tool_call['args']})?",
    })

    if decision.get("approved"):
        result = execute_tool(tool_call["name"], tool_call["args"])
    else:
        result = "Action rejected by reviewer."

    return {"messages": [{"role": "tool", "content": result,
                           "tool_call_id": tool_call["id"]}]}

builder = StateGraph(MessagesState)
builder.add_node("agent", call_model)          # proposes the tool call
builder.add_node("review", review_tool_call)   # pauses for approval
builder.add_edge(START, "agent")
builder.add_edge("agent", "review")
builder.add_edge("review", END)

graph = builder.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "req-1"}}
result = graph.invoke({"messages": [{"role": "user", "content": "Cancel order 4471"}]}, config)

print(result["__interrupt__"])
# [Interrupt(value={'action': 'cancel_order', 'args': {'order_id': '4471'}, ...})]

# A human reviews the payload out-of-band, then resumes:
final = graph.invoke(Command(resume={"approved": True}), config)
```

`interrupt()` accepts any JSON-serializable value, so the payload can carry exactly what a reviewer needs to decide (the tool name, the arguments, a plain-language question), not just a bare string. The `thread_id` in `config` is what ties the resumed call back to the exact paused state.

## Editable State: Letting a Human Correct, Not Just Approve

The same primitive extends naturally to a human editing state before the graph continues, rather than a binary approve or reject:

```python
def human_review_node(state: MessagesState) -> dict:
    edited = interrupt({"draft": state["messages"][-1].content})
    return {"messages": [{"role": "assistant", "content": edited}]}
```

Whatever value is passed to `Command(resume=...)` becomes the return value of `interrupt()`, so a reviewer editing a draft reply, correcting a misextracted field, or rewriting a generated plan all use this same shape: pause, surface the current value, resume with the corrected one.

## Where to Put the Gate

Not every tool call needs a human in the loop, and gating everything defeats the point of having an agent at all.

- **Read-only actions** (lookups, searches, calculations) — No gate needed. Nothing changes state, so there's nothing to approve.
- **Reversible actions with low blast radius** (drafting an email, adding an item to a cart) — Usually no gate, or a lightweight one shown after the fact rather than blocking execution.
- **Irreversible or high-consequence actions** (sending an email, canceling an order, executing a financial transaction, deleting data) — Always gate. This is the check-signing pattern: the agent fills out the check, nothing moves until a human co-signs it.
- **Actions where confidence is genuinely uncertain** (an ambiguous match, a low-relevance-score retrieval from the evaluation post earlier in this series) — Gate conditionally, based on a confidence signal, rather than gating every instance of that tool.

Encode this as a conditional inside the node calling `interrupt()`, not as a blanket policy per tool: the same `cancel_order` tool might need a gate for orders over a dollar threshold and not for tiny ones, and that's a business rule your review node should express directly.

## Failure Modes Specific to HITL

- **Interrupted state waiting forever.** A paused thread persists indefinitely until resumed. If your system has no process for surfacing pending approvals to a human, threads pile up silently. Build the review queue before you build the interrupt.
- **Non-idempotent code before the interrupt.** As noted above, anything before `interrupt()` in the node re-runs on resume. A side effect there (a log write with real consequences, an API call) fires twice.
- **Resuming with the wrong shape.** `Command(resume=...)` hands its value straight back as the interrupt's return value with no schema enforcement. A reviewer's malformed input can crash the node exactly where you can least afford a crash: mid-approval. Validate the resume payload the same way you'd validate a tool argument.
- **Treating `Command(update=...)` as a way to continue a conversation.** The LangGraph docs are explicit that `Command(resume=...)` is the only Command pattern meant as input to `invoke`; the other Command fields are for returning from node functions, not for driving the graph from outside.

> A human-in-the-loop gate is not a UI nicety. It's the difference between an agent that can act and an agent that's allowed to.

---
*If your agent can already send an email or move money with no pause anywhere in its graph, this is worth reading before it does either by mistake.*