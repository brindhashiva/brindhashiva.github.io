Title: Design Patterns for Building Reliable AI Agents
Date: 2026-05-12
Category: GenAI
Tags: AI Agents, Design Patterns, Reliability, LangGraph, Architecture
Slug: design-patterns-for-building-reliable-ai-agents
Status: published

The failure modes from the last post (compounding error, tool failure, context poisoning, scope creep, goal drift) aren't solved by trying harder on any single one. They're solved by a handful of structural patterns that show up, in some combination, in every agent system that survives contact with production. These aren't novel: most are decades-old distributed-systems ideas, applied to a new kind of unreliable component.

## Pattern: Bounded Retry With Escalation

A tool call or an agent step fails. Retrying blindly risks looping forever on a call that will never succeed; giving up immediately wastes calls that would have succeeded on a second attempt. The pattern is a capped retry with a fallback path, not an infinite loop and not a bare try-once.

```python
# pip install langgraph
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class TaskState(TypedDict):
    query: str
    attempts: int
    result: str | None

MAX_ATTEMPTS = 3

def attempt_tool_call(state: TaskState) -> dict:
    try:
        return {"result": risky_tool.invoke(state["query"]), "attempts": state["attempts"] + 1}
    except ToolError:
        return {"result": None, "attempts": state["attempts"] + 1}

def route(state: TaskState) -> str:
    if state["result"] is not None:
        return "done"
    if state["attempts"] >= MAX_ATTEMPTS:
        return "escalate"          # hand off to a human or a fallback tool
    return "retry"

builder = StateGraph(TaskState)
builder.add_node("attempt", attempt_tool_call)
builder.add_node("escalate", lambda s: {"result": "Needs human review: tool failed repeatedly."})
builder.add_edge(START, "attempt")
builder.add_conditional_edges("attempt", route, {"retry": "attempt", "escalate": "escalate", "done": END})
builder.add_edge("escalate", END)

graph = builder.compile()
```

The escalation branch is the part teams skip and then regret. A retry cap with no fallback just changes "infinite loop" into "silent failure after three tries," which is better but still leaves nobody informed.

## Pattern: Validation Gates Between Steps

Directly addresses context poisoning from the previous post: don't let an unchecked intermediate result become the premise for the next step. Insert a cheap validation node between generation and use, not just at the very end of the pipeline.

```python
def extract_order_id(state: TaskState) -> dict:
    extracted = extraction_llm.invoke(state["query"])
    return {"candidate_order_id": extracted}

def validate_order_id(state: TaskState) -> dict:
    candidate = state["candidate_order_id"]
    # Cheap, deterministic check, not another LLM call: does this look
    # like a real order ID at all, before we spend a lookup on it?
    is_valid = bool(candidate) and candidate.isdigit() and len(candidate) == 6
    return {"order_id_valid": is_valid}

def route_validation(state: TaskState) -> str:
    return "lookup" if state["order_id_valid"] else "ask_clarification"
```

The cheapest validation is deterministic, not another model call. Reach for an LLM-based check only when the property you're validating genuinely can't be expressed as a rule, since every added LLM call is another place for the compounding-error math from the last post to bite.

## Pattern: The Circuit Breaker

A tool that's currently down shouldn't be retried by every incoming request; that just multiplies load on a failing service and burns budget on calls that will fail identically. Track failure rate per tool and stop calling it once it crosses a threshold, until it recovers.

```python
from collections import deque
import time

class CircuitBreaker:
    def __init__(self, failure_threshold: float = 0.5, window: int = 20, cooldown_s: int = 60):
        self.results: deque[bool] = deque(maxlen=window)
        self.failure_threshold = failure_threshold
        self.cooldown_s = cooldown_s
        self.tripped_at: float | None = None

    def is_open(self) -> bool:
        if self.tripped_at and time.time() - self.tripped_at < self.cooldown_s:
            return True
        if self.tripped_at:
            self.tripped_at = None   # cooldown elapsed, allow a trial call
        return False

    def record(self, success: bool) -> None:
        self.results.append(success)
        if len(self.results) == self.results.maxlen:
            failure_rate = 1 - (sum(self.results) / len(self.results))
            if failure_rate >= self.failure_threshold:
                self.tripped_at = time.time()

breaker = CircuitBreaker()

def call_tool_safely(query: str) -> str:
    if breaker.is_open():
        return "Service temporarily unavailable; try again shortly."
    try:
        result = flaky_tool.invoke(query)
        breaker.record(success=True)
        return result
    except ToolError:
        breaker.record(success=False)
        raise
```

This belongs at the tool-call boundary, wrapping any external dependency an agent relies on, not inside the agent's reasoning loop.

## Pattern: Checkpointing for Recoverability

Long-running or multi-step agents need to resume from where they failed, not restart from scratch, or a failure late in a plan wastes every step that succeeded before it. This is the same checkpointer mechanism from the memory post earlier in this series, applied for recoverability rather than conversational continuity:

```python
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)
    # If the process crashes mid-run, re-invoking with the same thread_id
    # resumes from the last successfully checkpointed step, not step one.
```

Combined with the bounded-retry pattern, this means a transient failure costs one step's worth of rework, not the whole task's.

## Pattern: The Sanity-Check Sandwich

Wrap a risky action between two cheap checks: a precondition check before it runs, and a postcondition check after. This is validation gates applied specifically around side-effecting tool calls, where the cost of a wrong action is much higher than the cost of a wrong intermediate fact.

```python
def cancel_order_safely(order_id: str) -> str:
    order = order_lookup(order_id)
    if order["status"] == "shipped":
        return f"Cannot cancel: order {order_id} has already shipped."   # precondition
    result = cancel_order(order_id)
    confirmation = order_lookup(order_id)
    if confirmation["status"] != "cancelled":
        raise ToolError(f"Cancellation did not take effect for {order_id}")  # postcondition
    return result
```

This is cheap insurance against exactly the class of error that human-in-the-loop approval (from earlier in this series) is the heavier-weight version of. Use the sandwich for actions where a rule can catch the problem; reserve human approval for actions where no rule can.

## Composing the Patterns

None of these five patterns stand alone in a real system; they compose:

- A **circuit breaker** protects a flaky tool from being hammered.
- A **bounded retry** decides how many times to try that tool before escalating.
- A **validation gate** decides whether the tool's result is trustworthy enough to use.
- A **sanity-check sandwich** protects any side-effecting action the tool triggers.
- **Checkpointing** means a failure anywhere in this stack costs one step, not the whole run.

The combined cost is real, more code, more latency per step, more things to test, which is exactly why the earlier posts in this series (planning, workflow design) push toward shorter agent chains and deterministic pipelines wherever the task allows it. Reliability patterns are worth their overhead on the steps that need them; they're not a reason to add steps you don't otherwise need.

> Every pattern here is a bet that failure is normal, not exceptional. Agents built on the opposite assumption work fine in the demo and nowhere else.

---
*Send this to whoever's agent has a single try/except around the entire graph and calls that error handling.*