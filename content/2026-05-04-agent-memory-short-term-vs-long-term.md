Title: Agent Memory: Short-Term vs Long-Term
Date: 2026-05-04
Category: GenAI
Tags: AI Agents, Memory, LangGraph, Checkpointer, Persistence
Slug: agent-memory-short-term-vs-long-term
Status: published

An agent without memory re-derives everything from scratch on every turn: who's asking, what's already been tried, what happened three tool calls ago. LangGraph splits memory into two primitives with genuinely different scopes, and the single most common architecture mistake is using only one when the task needs both. Get the scope wrong and you either forget the conversation mid-task or, worse, leak one user's facts into another user's session.

## The Two Primitives

**Checkpointer (short-term memory)** — Persists the graph's full state, including message history, keyed by a `thread_id`. This is what makes a multi-turn conversation coherent within one session. It answers "what have we discussed in this thread?"

**Store (long-term memory)** — A key-value store organized by namespace and key, independent of any thread, typically namespaced by user or org identity. It answers "what do I know about this user across every session they've ever had?" A returning user with a brand-new `thread_id` still gets their stored preferences, because the store isn't scoped to the thread at all.

**Thread** — A single conversation session, identified by `thread_id`. Checkpointer state lives and dies with the thread (conceptually; the data itself persists in the backing store, but a new thread starts with empty state).

**Namespace** — A tuple used to organize entries in the long-term store, similar to a folder path. A common pattern is `(user_id, "preferences")` or `(user_id, "facts")`, which lets you scope reads and writes to exactly one user without a separate table per user.

The mistake worth naming explicitly: passing only a checkpointer when the task needs cross-session memory. That gives you a coherent conversation within one thread and total amnesia the moment the thread changes, which is often invisible in testing (you keep reusing the same thread) and glaring in production (every new session is a stranger).

## Short-Term Memory: One Line to Add

```python
# pip install langgraph
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph

checkpointer = InMemorySaver()

builder = StateGraph(...)
graph = builder.compile(checkpointer=checkpointer)

graph.invoke(
    {"messages": [{"role": "user", "content": "hi, I'm Bob"}]},
    {"configurable": {"thread_id": "session-1"}},
)
# A later call with the same thread_id continues this exact conversation.
# A different thread_id starts with no memory of Bob at all.
```

`InMemorySaver` is fine for development; it disappears on restart. In production, swap in a database-backed checkpointer:

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)
```

## Long-Term Memory: Across Every Session

```python
# pip install langgraph
from langgraph.store.memory import InMemoryStore
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, MessagesState

store = InMemoryStore()
checkpointer = InMemorySaver()

def write_memory(state: MessagesState, config: dict, *, store) -> dict:
    user_id = config["configurable"]["user_id"]
    last_user_msg = state["messages"][-1].content
    # Namespaced by user, keyed by whatever fact this is.
    store.put((user_id, "facts"), "last_topic", {"value": last_user_msg})
    return {}

def call_model(state: MessagesState, config: dict, *, store) -> dict:
    user_id = config["configurable"]["user_id"]
    remembered = store.get((user_id, "facts"), "last_topic")
    context = remembered.value["value"] if remembered else "no prior context"
    # ... build the prompt using `context`, call the LLM, return the response
    return {"messages": [llm.invoke(state["messages"])]}

builder = StateGraph(MessagesState)
builder.add_node("call_model", call_model)
builder.add_node("write_memory", write_memory)
builder.add_edge(START, "call_model")
builder.add_edge("call_model", "write_memory")

graph = builder.compile(checkpointer=checkpointer, store=store)

config = {"configurable": {"thread_id": "session-1", "user_id": "lance"}}
graph.invoke({"messages": [{"role": "user", "content": "Hi, I'm Lance"}]}, config)

# A brand-new thread, same user: long-term facts persist, short-term
# conversation history from session-1 does not.
new_session = {"configurable": {"thread_id": "session-2", "user_id": "lance"}}
graph.invoke({"messages": [{"role": "user", "content": "What was I asking about?"}]}, new_session)
```

Notice the two config keys serve different scopes: `thread_id` drives what the checkpointer restores, `user_id` drives what the store looks up. Conflating them, for example using `thread_id` as the store's namespace root, silently turns long-term memory into per-session memory again, defeating the point of having a store at all.

## Three Kinds of Long-Term Memory

The store is a generic key-value system; what you put in it determines what kind of memory it becomes:

- **Semantic memory** — Facts about the user: preferences, role, prior decisions. The example above is this: a fact keyed and retrieved by lookup.
- **Episodic memory** — Examples of past interactions, used as few-shot context for how the agent should behave in similar situations. Retrieved by similarity rather than exact key, since you're matching "a case like this one," not "the fact with this name."
- **Procedural memory** — Instructions the agent has learned to follow, sometimes to the point of the system prompt itself being rewritten based on accumulated experience. This is the least common and hardest to get right: a self-modifying prompt needs strong guardrails, or it drifts.

Most production agents only need semantic memory. Episodic and procedural memory are worth reaching for once you've measured that plain fact storage isn't capturing what the agent needs to improve.

## Managing the Growing Context

Short-term memory has its own failure mode independent of scope: a long-running thread's message history eventually exceeds the model's context window.

- **Trim before the call, not in storage.** `trim_messages` produces a shorter copy of the message list for the prompt while leaving the checkpointed state untouched, so you can change your trimming strategy later without having destroyed history.
- **Delete when you mean it.** `RemoveMessage` actually edits the graph's persisted state, removing messages for good. Use it deliberately, typically right after generating a summary of what's being removed, not as a routine trimming step.
- **Summarize old turns.** Replace a block of older messages with a single summary message once the thread crosses a length threshold, keeping recent turns verbatim and compressing the rest.

## Choosing a Backend

- **Development** — `InMemorySaver` and `InMemoryStore`. Zero setup, resets on restart, wrong for anything you need to survive a deploy.
- **Local persistence, single instance** — `SqliteSaver`. Survives restarts, doesn't scale across multiple app instances.
- **Production** — `PostgresSaver` and a matching `PostgresStore`. Survives restarts, scales across instances, and is where you should already be running evaluation and observability (covered later in this series).

> Short-term memory makes an agent coherent within a conversation. Long-term memory makes it worth having a second conversation at all.

---
*If your team's agent "forgets everything" between sessions, this is probably why. Pass it along before someone reaches for a bigger context window instead.*