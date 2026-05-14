Title: AI Agent Frameworks Compared
Date: 2026-05-14
Category: GenAI
Tags: AI Agents, LangGraph, OpenAI Agents SDK, CrewAI, AutoGen, Framework Comparison
Slug: AI-agent-frameworks-compared
Status: published

Every framework in this space claims to make building agents easy, and most of them do, for the specific shape of agent they were designed around. The differences that actually matter aren't feature checklists; they're architectural bets about how much control you keep over execution versus how much the framework decides for you, and how tightly you're willing to couple to one model provider. This post compares the frameworks by those bets, not by star counts.

## The Two Architectural Camps

**Graph-based orchestration** — You define an explicit state machine: nodes, edges, and the conditions that route between them. LangGraph is the clearest example. Control flow is data you author, even when a node's own behavior is model-driven. This is the more verbose option and the one that gives you the most explicit control over exactly what can happen and when, which matters a great deal for the reliability patterns from earlier posts in this series (retries, validation gates, checkpointing all slot naturally into named nodes).

**Model-driven loop** — You define agents, tools, and (for multi-agent setups) handoff rules; the framework runs the request-tool-response loop itself. The OpenAI Agents SDK, CrewAI, and Strands Agents are built this way. Faster to get something running, with less boilerplate, at the cost of the loop's internals being less visible and less customizable than a graph you built node by node yourself.

Neither camp is strictly better; they trade the same complexity for different things. A model-driven loop is the right choice when your agent's shape genuinely matches the framework's built-in pattern (a set of tools, an optional handoff to specialists). A graph is the right choice once you need custom control flow the framework's loop doesn't natively express, which is most of the reliability engineering from the last two posts.

## The Frameworks, by What They Optimize For

- **LangGraph** — Explicit graph-based state machines, first-class human-in-the-loop via `interrupt()`, durable checkpointing, and the deepest observability story through LangSmith. Model-agnostic. The steepest learning curve of the group (graph concepts, state schemas), and the highest ceiling for custom control flow. This is the framework used throughout the RAG and agent posts in this series for exactly that reason: every pattern from planning to HITL to reliability engineering needed an explicit graph to express cleanly.
- **OpenAI Agents SDK** — A clean, opinionated model-driven loop with tools, handoffs between agents, guardrails, and structured outputs built in. Low learning curve, strong built-in tracing. Tightly coupled to OpenAI models, which is either irrelevant or disqualifying depending on your provider strategy.
- **CrewAI** — Role-based multi-agent teams: you define agents by role, backstory, and goal, then assemble them into a "crew" with tasks. The lowest learning curve of any framework here and the fastest path from idea to a working multi-agent demo. State persistence and fine-grained control are comparatively limited; it's built for rapid prototyping of role-based collaboration, not for the reliability patterns from the last post.
- **AutoGen / AG2** — Conversational multi-agent patterns, where agents collaborate through structured group chat. AG2 is the actively developed community successor after Microsoft moved the original AutoGen into a more limited maintenance mode. Good fit for tasks that are naturally conversational between specialist agents; more limited streaming and state-persistence than LangGraph.
- **Google ADK, Microsoft Agent Framework, Claude Agent SDK, Strands Agents** — Each is the natural choice if you're already committed to that vendor's ecosystem (Vertex AI, Azure, Anthropic, or AWS respectively), with the vendor-specific integration and infrastructure advantages that implies, and the corresponding lock-in trade-off.

## Comparing the Two Camps on One Task

A supervisor delegating to a research specialist, in LangGraph's explicit style (from the multi-agent post earlier in this series):

```python
# pip install langgraph-supervisor langchain langchain-openai
from langchain.agents import create_agent
from langgraph_supervisor import create_supervisor
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o")
research_agent = create_agent(model=model, tools=[web_search_tool], name="research_agent")

workflow = create_supervisor([research_agent], model=model,
    prompt="Delegate research questions to research_agent.")
app = workflow.compile()
```

The same shape of task in the OpenAI Agents SDK's handoff style. Handoffs are a first-class primitive: an agent can hand the conversation fully to another agent, rather than a supervisor retaining control and calling workers as tools.

```python
# pip install openai-agents
# Shape only; verify current class and parameter names against the SDK's
# own docs before publishing, the API surface here moves quickly.
from agents import Agent, Runner

research_agent = Agent(name="Research Agent", instructions="Find current facts.",
                        tools=[web_search_tool])
triage_agent = Agent(name="Triage", instructions="Delegate research questions.",
                      handoffs=[research_agent])

result = Runner.run_sync(triage_agent, "What's the latest on X?")
```

<!-- VERIFY before publishing: the OpenAI Agents SDK's exact class and method names (Agent, Runner, handoffs=) move quickly; confirm against the SDK's current documentation before shipping this snippet, since I could not verify the precise current surface with confidence. -->

Both accomplish delegation. The supervisor pattern keeps a central coordinator in the loop for every step (useful when you want one place to add the reliability patterns from two posts ago); the handoff pattern transfers full control (useful when a specialist should own the rest of the conversation outright, with no need to report back).

## A Decision Framework, Not a Ranking

- **You need explicit control over every step, custom reliability patterns, or complex human-in-the-loop workflows** — LangGraph. The verbosity buys you the control the earlier posts in this series depend on.
- **You're building fast, staying within one provider's ecosystem, and want the framework's opinions to just work** — OpenAI Agents SDK, Claude Agent SDK, Google ADK, or Microsoft Agent Framework, matched to whichever provider you're already committed to.
- **You're prototyping a role-based multi-agent idea and want to validate it in an afternoon** — CrewAI. Expect to reconsider the framework if the prototype needs production-grade state management later.
- **Your agents communicate in a genuinely conversational, back-and-forth pattern rather than a fixed pipeline or handoff chain** — AutoGen/AG2's group-chat model fits that shape more naturally than a graph or a linear handoff chain would.
- **You need agents built in different frameworks to interoperate**, tools especially — reach for MCP (the previous post in this series) as the interoperability layer regardless of which framework each agent itself is built in; the frameworks compared here are about how one agent runs, not about cross-framework communication.

## What Doesn't Change Across Frameworks

Every pattern from earlier in this series, tool design, memory scoping, the reliability patterns, human-in-the-loop gates, observability, is a concept, not a LangGraph-specific feature. The framework changes the API surface for implementing them, sometimes dramatically, but the underlying design questions (what should be a separate step, what needs a human's approval, what state is short-term versus long-term) don't go away just because you picked a framework with less boilerplate. A model-driven loop that hides the control flow from you still has control flow; you've just traded visibility into it for less code to write.

> The right agent framework is the one whose default behavior matches the amount of control your task actually needs, not the one with the most GitHub stars this quarter.

---
*Send this to whoever's about to pick a framework by vibes instead of by whether they need graph-level control or a fast prototype.*