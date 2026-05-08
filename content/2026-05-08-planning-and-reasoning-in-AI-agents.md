Title: Planning and Reasoning in AI Agents
Date: 2026-05-08
Category: GenAI
Tags: AI Agents, ReAct, Planning, Reasoning, LangGraph
Slug: planning-and-reasoning-in-AI-agents
Status: published

Give an agent five tools and a complex task, and it needs some strategy for deciding what to do first, second, and when to stop. "Just let the model figure it out" is itself a strategy, and often a fine one, but it's worth distinguishing from the alternatives, because the choice determines how much visibility you have into the agent's reasoning before it acts, and how expensive that reasoning is.

## The Core Approaches

**ReAct (Reason + Act)** — Interleave reasoning and action one step at a time: think, act, observe the result, think again. The classic implementation is a text loop of Thought, Action, Action Input, Observation, though `create_agent`-style tool calling gets you the same interleaving without hand-parsing that format. Its strength is adaptability: each step's reasoning can incorporate what was just observed. Its weakness is myopia: the model never commits to a multi-step plan, so it can wander if a task genuinely needs foresight.

**Plan-and-execute** — Generate a full plan up front (a list of steps), then execute each step, optionally replanning if a step's result invalidates the rest of the plan. Trades ReAct's adaptability for visibility: you can inspect, log, and even approve the plan before a single tool runs, which matters when actions have consequences.

**Chain-of-thought (CoT)** — Reasoning expressed as intermediate text before a final answer, with no tool calls or actions involved. This is not agentic planning; it's a reasoning technique inside a single generation. It's worth naming separately because "the model reasons step by step" gets conflated with "the model plans a sequence of actions," and they're different things solving different problems.

**Reflection** — Having the model (or a separate call) critique its own prior output or plan before proceeding, and revise if the critique finds a problem. This catches a class of errors neither ReAct nor plan-and-execute catches on their own: confidently wrong intermediate steps that would otherwise propagate uncaught.

## ReAct in Practice

With `create_agent`, ReAct-style interleaving of reasoning and action is the built-in behavior. Every call, the model sees the conversation and tool results so far and decides the next tool call or final answer:

```python
# pip install langchain
from langchain.agents import create_agent
from langchain_core.tools import tool

@tool
def search(query: str) -> str:
    """Search for current information on the web."""
    return web_search(query)

@tool
def calculate(expression: str) -> str:
    """Evaluate a math expression."""
    return str(eval(expression, {"__builtins__": {}}, {}))

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[search, calculate],
    system_prompt="Reason step by step. Use tools when you need current "
                  "facts or need to compute something.",
)

result = agent.invoke({
    "messages": [{"role": "user", "content":
        "What's the combined market cap of the two largest EV makers, "
        "rounded to the nearest billion?"}]
})
```

Each iteration of the underlying loop is a Reason-then-Act step: the model reasons about what it still needs, acts by calling a tool, observes the result, and reasons again. Nothing here commits to a plan beyond the immediate next step, which is precisely what makes it flexible for tasks where the right next step depends on what the last tool call actually returned.

## Plan-and-Execute in LangGraph

When you want the plan itself to be inspectable, or approvable, before execution starts, build it as an explicit graph stage rather than relying on ReAct's implicit step-by-step reasoning:

```python
# pip install langgraph langchain
from typing import TypedDict
from pydantic import BaseModel, Field
from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END

llm = init_chat_model("gpt-4o", model_provider="openai")

class Plan(BaseModel):
    steps: list[str] = Field(description="Ordered steps to complete the task.")

class PlanState(TypedDict):
    task: str
    plan: list[str]
    completed: list[str]
    result: str

def make_plan(state: PlanState) -> dict:
    planner = llm.with_structured_output(Plan)
    plan = planner.invoke(f"Break this task into ordered steps:\n{state['task']}")
    return {"plan": plan.steps, "completed": []}

def execute_step(state: PlanState) -> dict:
    next_step = state["plan"][len(state["completed"])]
    outcome = run_step_agent.invoke({"messages": [{"role": "user", "content": next_step}]})
    return {"completed": state["completed"] + [outcome["messages"][-1].content]}

def should_continue(state: PlanState) -> str:
    return "execute" if len(state["completed"]) < len(state["plan"]) else "finish"

def finish(state: PlanState) -> dict:
    return {"result": "\n".join(state["completed"])}

builder = StateGraph(PlanState)
builder.add_node("plan", make_plan)
builder.add_node("execute", execute_step)
builder.add_node("finish", finish)
builder.add_edge(START, "plan")
builder.add_conditional_edges("plan", lambda s: "execute", ["execute"])
builder.add_conditional_edges("execute", should_continue, {"execute": "execute", "finish": "finish"})
builder.add_edge("finish", END)

graph = builder.compile()
```

The plan is a first-class value in `state["plan"]`, visible for logging, and it's the natural place to insert a human-in-the-loop approval step (covered in the next post in this series) between `make_plan` and the first `execute_step`, since the whole point of planning up front is having something concrete to review before anything runs.

## Choosing Between Them

- **The task is exploratory and each step depends heavily on the last result** (research, debugging, open-ended investigation) — ReAct. Committing to a plan before you've seen any intermediate results wastes the adaptability that makes these tasks tractable at all.
- **Actions have real consequences and someone should review the sequence before it runs** (financial transactions, infrastructure changes, anything irreversible) — Plan-and-execute, paired with an approval gate on the plan itself.
- **The task is long enough that losing track of the overall goal is a real risk** — Plan-and-execute's explicit plan acts as a running checklist that a pure ReAct loop doesn't naturally maintain.
- **Correctness of intermediate reasoning matters more than speed** — Add reflection on top of either approach: a critique step after planning, or after each ReAct action, at the cost of extra model calls per step.

None of these are mutually exclusive. A common production shape is plan-and-execute at the top level (for visibility and approval), with each individual step handled by a ReAct-style sub-agent (for adaptability within that step). Pick the level of foresight the task's consequences actually justify, not the most sophisticated-sounding option.

> Planning doesn't make an agent smarter. It makes its reasoning visible before the reasoning becomes a side effect you can't undo.

---
*Send this to whoever's letting a ReAct loop make irreversible calls with no plan anyone reviewed first.*