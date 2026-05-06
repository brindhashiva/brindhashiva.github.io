Title: Multi-Agent Systems and Collaboration
Date: 2026-05-06
Category: GenAI
Tags: AI Agents, Multi-Agent, LangGraph, Supervisor Pattern, Orchestration
Slug: multi-agent-systems-and-collaboration
Status: published

The instinct to split an agent into multiple agents usually arrives the moment one system prompt starts accumulating "if the user asks about X, do Y, but if it's actually about Z, do W instead." That's a real signal, but multi-agent systems trade one kind of complexity (a bloated prompt) for another (an orchestration and communication problem). Before reaching for it, it's worth being precise about what problem multiple agents actually solve, because it isn't "one agent isn't smart enough."

## What Multiple Agents Actually Buy You

**Specialization** — Each agent gets a focused system prompt and a small, coherent tool set, instead of one agent juggling every tool and every persona at once. A research agent and a math agent are each easier to prompt well than one agent trying to be both.

**Isolation** — One agent's context window doesn't fill up with another agent's intermediate scratch work. A research agent's search results don't need to pollute a writing agent's context.

**Parallelism** — Independent subtasks can run concurrently instead of serially through one agent's loop, when the architecture supports it (the network pattern below; supervisor is typically sequential per task).

None of this makes the combined system smarter than a single well-designed agent with the same tools would be. It makes the system easier to build, prompt, and debug incrementally, and easier to isolate failures in, at the cost of an orchestration layer that is itself a new thing to get wrong.

## Three Topologies

- **Supervisor** — A central orchestrator decides which specialist handles each step and holds the shared conversation. Workers report back to the supervisor, never to each other. Predictable control flow, one place to look when routing goes wrong, and the natural default for most teams.
- **Network** — Any agent can hand off to any other directly, with no central coordinator. Flexible, but the failure modes are genuinely harder to reason about: nothing stops two agents from handing off to each other indefinitely, and there's no single place to add a circuit breaker.
- **Hierarchical (supervisors of supervisors)** — A top-level supervisor coordinates several sub-supervisors, each managing their own team. This is overkill for most projects; reach for it only when a single supervisor's worker list has grown large enough that its own routing decisions are the bottleneck.

A rule of thumb worth stating plainly: if a single agent with all the tools is already hitting your target accuracy on your eval set, stay single. Move to a supervisor when you have demonstrably distinct task types and a single agent's accuracy has plateaued despite prompt iteration, not before you've tried that.

## Building a Supervisor

`langgraph-supervisor` builds this pattern on top of LangGraph's primitives: worker agents are ordinary `create_agent` agents, and handoff between them is implemented as tool calls the supervisor makes.

```python
# pip install langgraph-supervisor langchain langchain-openai
from langchain.agents import create_agent
from langgraph_supervisor import create_supervisor
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o")

research_agent = create_agent(
    model=model,
    tools=[web_search_tool],
    system_prompt="You are a research expert. Find facts; do not do any math.",
    name="research_agent",
)

math_agent = create_agent(
    model=model,
    tools=[calculator_tool],
    system_prompt="You are a math expert. Do calculations; do not research facts.",
    name="math_agent",
)

workflow = create_supervisor(
    [research_agent, math_agent],
    model=model,
    prompt=(
        "You are a team supervisor managing a research expert and a math "
        "expert. For current events, use research_agent. For math "
        "problems, use math_agent."
    ),
)

app = workflow.compile()

result = app.invoke({
    "messages": [{
        "role": "user",
        "content": "What's the combined headcount of the FAANG companies?"
    }]
})
print(result["messages"][-1].content)
```

`create_supervisor` returns an uncompiled `StateGraph`, so it takes a checkpointer and store exactly like any other graph, which means everything from the memory post in this series applies unchanged:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore

app = workflow.compile(checkpointer=InMemorySaver(), store=InMemoryStore())
```

Under the hood, the supervisor is equipped with a handoff tool per worker (named `transfer_to_<agent_name>` by default), and calling one of those tools is how control moves from the supervisor to a worker and back. You're not writing custom routing logic; you're letting the supervisor's own tool-calling decide it, the same mechanism from the tool-calling post earlier in this series, just used for delegation instead of external actions.

## Where This Breaks

- **Runaway handoffs.** Nothing inherently stops a network-topology system from bouncing between two agents. Cap total steps or hops, and log every handoff so a loop is visible immediately rather than discovered from a token bill.
- **Context duplication.** By default, workers can receive the full message history, including other workers' outputs, which balloons token usage fast. `create_supervisor`'s `output_mode` controls how much of a worker's history flows back into the shared conversation; keep it tight unless a worker's full trace genuinely matters downstream.
- **Cost multiplies with agents.** Every worker call is its own set of model calls, on top of the supervisor's own routing calls. A four-specialist team can easily cost several times a single agent's token spend for the same task. Multi-agent is expensive; know when it's worth it before committing to the architecture.
- **Evaluation needs a routing metric, not just an output metric.** Score whether the supervisor picked the right worker for a given task, separately from whether the final answer was correct. A right answer reached via the wrong worker is a routing bug you'll want to know about even though the output looked fine.
- **Debugging spans multiple agents' contexts.** When something goes wrong, you need to see the supervisor's routing decision and each worker's own trace, not just the final message. This is exactly what the observability techniques covered later in this series are for.

> Multi-agent systems don't make any individual agent smarter. They make a complicated prompt into a complicated org chart, which is only progress if the org chart is easier to debug than the prompt was.

---
*If your team's one mega-agent prompt just grew another "unless the user is asking about billing" clause, this is worth a read before the next one gets added.*