Title: Agent Observability and Debugging
Date: 2026-05-10
Category: GenAI
Tags: AI Agents, Observability, LangSmith, Tracing, Debugging
Slug: agent-observability-and-debugging
Status: published

An agent that reasons across multiple tool calls, sometimes delegating to other agents, produces failures that a single log line can't explain. "The answer was wrong" tells you nothing about which of five steps caused it. Observability for agents means something more specific than for a normal service: you need the full decision trace, not just inputs and outputs, because the decisions are the thing most likely to be wrong.

## What "Trace" Means Here

**Span** — One unit of work within a trace: a single LLM call, a single tool execution, a single retrieval. Spans nest, so a supervisor's span (from the multi-agent post earlier in this series) contains each worker agent's spans, which contain their own tool-call spans.

**Trace** — The full tree of spans for one end-to-end run, from the initial input to the final output, showing exactly which node called what, with what arguments, and what came back.

**Run tree** — The parent-child structure of spans within a trace, which is what actually lets you answer "why did the agent do that?": you can see the LLM call that decided to call a tool, the tool's actual output, and the next LLM call that reasoned over it.

**Instrumentation** — The mechanism generating spans and traces. For LangChain and LangGraph applications, this is largely automatic, which is the main practical advantage of building on top of them rather than a bespoke agent loop.

## Turning Tracing On

For LangChain or LangGraph applications, this needs no code changes, only environment variables:

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY="your-api-key"
export LANGSMITH_PROJECT="my-agent"   # optional; defaults to "default"
```

Every graph invocation, every LLM call inside it, every tool call, gets traced automatically once these are set, with no changes to the agent code itself. This applies to everything built across this series: the RAG graphs, the supervisor, the ReAct and plan-and-execute agents, the HITL interrupts, all of it shows up as spans without extra instrumentation.

## Manual Instrumentation for Non-LangChain Code

Not everything in an agent system runs through LangChain. Custom pre- or post-processing, calls to other services, or a bespoke tool implementation can be traced explicitly with the `@traceable` decorator:

```python
# pip install langsmith
from langsmith import traceable
from langsmith.wrappers import wrap_openai
from openai import OpenAI

client = wrap_openai(OpenAI())   # auto-traces every call this client makes

@traceable(run_type="tool")
def validate_order(order_id: str) -> dict:
    """Custom validation logic, traced as its own span."""
    return run_validation(order_id)

@traceable
def process_request(user_input: str) -> str:
    validation = validate_order(extract_order_id(user_input))
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": f"{user_input}\n\n{validation}"}],
    )
    return response.choices[0].message.content
```

`wrap_openai` instruments the client itself, so every call it makes becomes a properly typed LLM span without touching call sites. `@traceable` on your own functions nests them into the same trace tree, tagged by `run_type` (`tool`, `chain`, `retriever`, and so on) so the run tree reads correctly even for code LangChain never sees.

## What to Actually Look For

A trace is only useful if you know what you're checking. For an agentic system, walk it in this order:

- **Did the agent choose the right tool at each step?** This is the tool-calling failure mode from earlier in this series (wrong tool, or no tool when one was needed) made visible: the span shows the exact tool name and arguments the model proposed.
- **Did the tool actually return what the agent expected?** A tool can succeed and still return something misleading (a stale lookup, an empty result treated as valid). This distinguishes "the model reasoned badly" from "the model reasoned correctly over bad input."
- **Where did a multi-agent handoff go wrong?** In a supervisor system, check the supervisor's own span for its stated reasoning about which worker to call, separately from whether that worker's output was correct. A right answer from the wrong worker is a routing bug worth fixing even though the trace looks fine at a glance.
- **Where did an interrupt fire, and what was resumed?** For HITL systems, the trace should show the paused state and the value it was resumed with, which is the fastest way to confirm a reviewer's approval was actually applied to the right pending action.
- **How much did this trace cost, in tokens and in wall-clock time?** Every span typically carries token counts and latency. For multi-agent or multi-step plans, this is usually where the real surprise is: a five-step plan whose steps looked reasonable individually can add up to costs nobody sized in advance.

## Building Regression Detection, Not Just Inspection

Looking at one trace after a complaint is debugging. Catching problems before a user reports them requires treating traces as a dataset:

- **Sample production traffic into an eval set.** This connects directly to the evaluation post earlier in this series: traces from real usage are the best source of new golden-set examples, especially the failing ones.
- **Score trajectories, not just final answers.** An eval pipeline for an agent needs to check the sequence of tool calls against an expected pattern, not only whether the final text matches a reference, since two very different paths can arrive at the same correct-looking answer.
- **Watch cost and latency percentiles over time, not just averages.** A multi-agent or planning system's cost is a function of how many steps it takes, and that number can drift upward silently as prompts or tool sets evolve, well before it shows up in an average.
- **Alert on interrupts left unresolved,** for HITL systems specifically: a growing backlog of paused threads is an observability signal in its own right, not just an operational nuisance.

> You cannot debug a decision you cannot see. For an agent, the trace isn't a nice-to-have log, it's the only artifact that actually shows you the reasoning that produced the bug.

---
*If the last time your team debugged an agent was staring at raw JSON in terminal output, this is worth forwarding before the next incident.*