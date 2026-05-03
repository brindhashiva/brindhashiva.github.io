Title: What Makes an AI Agent Different From a Chatbot
Date: 2026-05-03
Category: GenAI
Tags: AI Agents, Chatbots, LangGraph, Tool Calling, Architecture
Slug: what-makes-an-ai-agent-different-from-a-chatbot
Status: published

"Agent" has been stretched to cover everything from a chatbot with a system prompt to a system that runs unattended for hours across dozens of tool calls. That looseness costs real engineering decisions: whether you need a control loop, how you handle partial failure, what "done" even means. The useful distinction isn't vibes, it's architecture: does the system decide its own next step, or does it just respond?

## The Actual Distinction

**Chatbot** — A conversational interface around a single request-response step. The control flow is fixed: user message in, one model call (maybe with retrieved context), response out. The model can be very good at that one step without ever deciding what to do next.

**Agent** — A system where the model's output determines what happens next in the system, not just what's shown to the user. The model chooses whether to call a tool, which tool, with what arguments, and whether the task is finished. Control flow is data, not code.

**Control loop** — The mechanism that repeatedly asks the model "given everything so far, what next?" and executes whatever it decides, until some stopping condition. This loop is the actual technical difference; everything else follows from having one or not.

**Autonomy** — How many decisions the system makes without a human confirming each one. This is a spectrum, not a boolean, and it is a separate axis from "agent vs. chatbot": a agent can be low-autonomy (a human approves every tool call) or high-autonomy (it runs a multi-hour pipeline unattended).

## Why RAG Sits in Between

The RAG systems built across this series so far are instructive here, because they complicate a clean binary. A fixed retrieve-then-generate pipeline (from the production RAG post) is not an agent: the sequence of steps is hardcoded, even though an LLM generates the final answer. But the agentic RAG pattern from that same post, where the model decides whether to call the retriever tool at all, is a genuine agent, just a narrow one with a single tool and a tight stopping condition.

That's the useful test: not "does it use an LLM to produce output" but "does an LLM's output change which code path runs next." A model choosing words is generation. A model choosing actions is agency.

## Same Task, Both Architectures

A support bot that looks up order status. As a chatbot with a fixed pipeline:

```python
# pip install langchain langchain-openai
from langchain.chat_models import init_chat_model
from langchain_core.prompts import ChatPromptTemplate

llm = init_chat_model("gpt-4o-mini", model_provider="openai")
prompt = ChatPromptTemplate.from_template(
    "Order status: {status}\n\nAnswer the user's question using only this "
    "information.\n\nQuestion: {question}"
)

def handle(question: str, order_id: str) -> str:
    status = order_lookup(order_id)          # fixed: always looked up
    msg = prompt.invoke({"status": status, "question": question})
    return llm.invoke(msg).content            # fixed: one model call, then done
```

There is no branch here the model controls. It always looks up the order, always answers once, never decides to do anything else.

As an agent, using LangChain's current high-level `create_agent`:

```python
# pip install langchain langchain-openai
from langchain.agents import create_agent
from langchain_core.tools import tool

@tool
def order_lookup(order_id: str) -> str:
    """Look up the current status of an order by its ID."""
    return ORDER_STATUSES.get(order_id, "Order not found.")

@tool
def cancel_order(order_id: str) -> str:
    """Cancel an order if it hasn't shipped yet."""
    return process_cancellation(order_id)

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[order_lookup, cancel_order],
    system_prompt="Help with order questions. Only cancel an order if the "
                  "user explicitly asks to cancel.",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Cancel order 4471, it hasn't shipped"}]
})
print(result["messages"][-1].content)
```

Same domain, structurally different system. The agent version decides, per message, whether to look something up, cancel something, both, or neither, and the model's choice of tool and arguments is what drives execution, not a hardcoded sequence.

## What You Take On By Adding a Loop

- **Non-determinism in control flow, not just content.** A chatbot can give a slightly different answer to the same question. An agent can take a structurally different path: two runs of "cancel order 4471" might call `order_lookup` then `cancel_order`, or call `cancel_order` directly, or ask a clarifying question first.
- **A stopping condition you must design.** Without one, a control loop can call tools indefinitely. Most frameworks cap iterations or steps by default; know your framework's default and whether it fits your task.
- **Partial failure becomes a normal case, not an edge case.** A chatbot either answers or errors. An agent can succeed at step 2 of 4 and fail at step 3, leaving real side effects (an email sent, a record updated) that a retry would duplicate.
- **Evaluation gets harder.** You're no longer scoring one output against one reference; you're scoring a trajectory of decisions. The evaluation techniques from earlier in this series (golden sets, LLM judges) still apply, but now you also need to check the path taken, not just the final answer.
- **Latency and cost scale with steps, not calls.** A five-tool-call agent run costs five times the model calls of a chatbot turn, and each one adds latency serially unless you specifically parallelize.

## When You Don't Need One

Not every "smart" system needs a control loop:

- **The steps are always the same, regardless of input** — a fixed pipeline (RAG, a form-filler, a summarizer) is simpler, cheaper, and easier to test. Build that first.
- **The task has one clear action, no branching** — a single tool call bolted onto a prompt doesn't need a loop around it.
- **You can't tolerate the non-determinism** — compliance-sensitive workflows where the exact sequence of actions must be auditable and repeatable often want an explicit state machine, not a model deciding control flow at runtime.

Reach for an agent when the number of steps or their order genuinely depends on what the model discovers along the way, and a fixed pipeline would need an unmanageable number of branches to cover the cases. That's also the condition the rest of this series' agent posts, on memory, tool calling, multi-agent systems, planning, and human-in-the-loop control, assume you've already decided you need.

> A chatbot answers. An agent decides what to do next and then answers. That "and then" is the entire architecture.

---
*Forward this to whoever's calling a single-prompt wrapper an "agent" in the next planning doc.*