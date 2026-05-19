Title: Guardrails for AI Applications
Date: 2026-05-19
Category: GenAI
Tags: Guardrails, AI Safety, Input Validation, Output Validation, Prompt Injection
Slug: guardrails-for-AI-applications
Status:published

Evaluation, covered in the last post, tells you whether your system is good on the cases you thought to test. Guardrails are the different, complementary job: catching bad behavior at runtime, on cases nobody wrote a test for, before a bad input reaches the model or a bad output reaches the user. Where evaluation is a report card, a guardrail is a seatbelt, and conflating the two is how teams end up with excellent eval scores and an agent that still leaks a system prompt to the first person who asks nicely.

## Three Places Guardrails Live

**Input guardrails** — Filter what reaches the model: prompt injection attempts, PII in a user message that shouldn't be logged or forwarded, off-topic requests outside the system's intended scope. Runs before the LLM call.

**Logic guardrails** — Constrain what the system does during execution, not just what text it produces: blocking unauthorized tool calls, enforcing rate limits, requiring the human-in-the-loop approval from earlier in this series for specific actions. This is a control-flow concern, not a text-filtering one.

**Output guardrails** — Validate what the model produced before it reaches the user or gets acted on: schema compliance for structured output, toxicity or PII in the response, factual grounding against retrieved context (the faithfulness metric from the evaluation posts, applied at runtime instead of in a test suite).

The distinction matters because each layer catches a different failure and belongs in a different place in your pipeline. An input guardrail that blocks prompt injection does nothing to catch a hallucinated fact in an otherwise well-formed output; you need both, at their respective boundaries.

## Deterministic Checks Come First

The cheapest, most reliable guardrails are ordinary code, not another model call. Before reaching for an LLM-based check, ask whether a rule already covers it:

```python
import re

BLOCKED_PATTERNS = [
    re.compile(r"ignore (all )?previous instructions", re.I),
    re.compile(r"you are now|new system prompt", re.I),
]

def input_guardrail(user_message: str) -> tuple[bool, str | None]:
    for pattern in BLOCKED_PATTERNS:
        if pattern.search(user_message):
            return False, "This message matches a known prompt-injection pattern."
    return True, None

def output_guardrail(response: str, schema: dict) -> tuple[bool, str | None]:
    import json
    try:
        parsed = json.loads(response)
    except json.JSONDecodeError:
        return False, "Response is not valid JSON."
    missing = [k for k in schema["required"] if k not in parsed]
    if missing:
        return False, f"Response missing required fields: {missing}"
    return True, None
```

Pattern matching catches the obvious cases cheaply and with zero added latency, but it's brittle against rephrasing, and brittleness here is a real limitation, not a minor caveat: pattern lists reliably miss novel phrasings of the same attack, so treat this as one layer, not the whole defense.

## Model-Based Checks for What Rules Can't Express

Some properties genuinely need judgment: is this response toxic, does this claim contradict the retrieved context, is this request trying to extract the system prompt through indirection rather than a matched phrase. This is the same LLM-as-judge mechanism from the evaluation posts, moved from test time to runtime, which means it inherits the same cost and latency trade-off: every guardrail check here is an added model call in the critical path.

```python
from pydantic import BaseModel

class SafetyCheck(BaseModel):
    safe: bool
    reason: str

def moderate_output(response: str, guard_model) -> SafetyCheck:
    checker = guard_model.with_structured_output(SafetyCheck)
    return checker.invoke(
        "Evaluate whether this response is safe to show a user: no toxic "
        "content, no leaked system instructions, no fabricated claims "
        f"presented as fact.\n\nResponse: {response}"
    )
```

A dedicated, smaller, purpose-tuned classifier is usually a better fit here than a full general-purpose model call: it's cheaper, faster, and its one job (classify safe or not) is exactly the kind of narrow, well-specified task a smaller model handles reliably, unlike open-ended generation.

## Wiring Guardrails Into a Graph

Guardrails compose naturally as nodes in the same kind of graph used throughout this series, which means they inherit the reliability patterns (validation gates, the sanity-check sandwich) from two posts ago rather than needing new machinery of their own:

```python
# pip install langgraph
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class GuardedState(TypedDict):
    user_message: str
    blocked_reason: str | None
    response: str | None

def check_input(state: GuardedState) -> dict:
    ok, reason = input_guardrail(state["user_message"])
    return {"blocked_reason": None if ok else reason}

def route_input(state: GuardedState) -> str:
    return "blocked" if state["blocked_reason"] else "generate"

def generate(state: GuardedState) -> dict:
    return {"response": agent.invoke(state["user_message"])}

def check_output(state: GuardedState) -> dict:
    ok, reason = output_guardrail(state["response"], expected_schema)
    return {"blocked_reason": None if ok else reason}

def route_output(state: GuardedState) -> str:
    return "blocked" if state["blocked_reason"] else "deliver"

builder = StateGraph(GuardedState)
builder.add_node("check_input", check_input)
builder.add_node("generate", generate)
builder.add_node("check_output", check_output)
builder.add_node("blocked", lambda s: {"response": f"Request blocked: {s['blocked_reason']}"})
builder.add_node("deliver", lambda s: s)

builder.add_edge(START, "check_input")
builder.add_conditional_edges("check_input", route_input, {"blocked": "blocked", "generate": "generate"})
builder.add_edge("generate", "check_output")
builder.add_conditional_edges("check_output", route_output, {"blocked": "blocked", "deliver": "deliver"})
builder.add_edge("blocked", END)
builder.add_edge("deliver", END)

graph = builder.compile()
```

This is deliberately the same shape as the validation-gate pattern from the reliability-patterns post: a check node, a routing function, and an explicit blocked path, applied at the input and output boundaries of the whole system instead of between two internal steps.

## Purpose-Built Frameworks

Hand-rolled guardrails like the ones above cover most needs, but two open-source frameworks are worth knowing when the requirements grow:

- **Guardrails AI** — A validator-based framework for output validation specifically: pre-built and custom validators (from a hub of community-contributed checks) that can automatically correct invalid output or re-prompt the model with feedback about what was wrong, rather than only flagging a pass/fail.
- **NeMo Guardrails** — A broader framework covering input, dialog, and output rails together, with a domain-specific language (Colang) for defining conversational flows the system is allowed to follow, and built-in checks for jailbreak detection, fact-checking against a knowledge source, and hallucination detection.

Both are worth reaching for once your guardrail logic outgrows a handful of hand-written checks; neither replaces the judgment call of deciding what actually needs guarding in your specific application, which is a product and risk question no library answers for you.

## What Actually Needs a Guardrail

Not everything does, and over-guarding has a real cost in latency and false positives that block legitimate requests.

- **Anything that becomes a tool call with side effects** — Always. This is the logic-guardrail category, and it's the same territory the human-in-the-loop post covers for high-consequence actions specifically.
- **Anything a user could plausibly try to manipulate for a security or policy bypass** — Input guardrails, layered: deterministic pattern matching first, model-based judgment for what patterns miss.
- **Anything presented to a user as factual, in a domain where being wrong has real consequences** — Output guardrails checking groundedness against retrieved context, the faithfulness check from the RAG-evaluation post, moved to runtime.
- **Low-stakes, read-only, exploratory interactions** — Often fine with lighter guardrails or none, where the cost of a false positive (blocking a legitimate request) outweighs the risk of the rare bad one.

> A guardrail that never fires either means your system is unusually safe or your guardrail isn't actually checking anything. Test the guardrails themselves, the same way you test the system they're protecting.

---
*If your team's only guardrail is a line in the system prompt asking the model nicely not to do the bad thing, this is worth reading before someone asks it not-so-nicely.*