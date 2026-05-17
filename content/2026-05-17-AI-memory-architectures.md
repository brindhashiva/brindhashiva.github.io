Title: AI Memory Architectures
Date: 2026-05-17
Category: GenAI
Tags: AI Memory, LangGraph, Vector Memory, Knowledge Graphs, Memory Consolidation
Slug: AI-memory-architectures
Status: published

The agent-memory post earlier in this series covered LangGraph's two primitives, a checkpointer for short-term state and a store for long-term facts, which is the right starting point for most agents. This post is the layer above that: once you have a store, what do you actually put in it, how do you retrieve the right slice of it at the right time, and what happens when it grows large enough that "just search it" stops being a good enough retrieval strategy. Memory architecture is retrieval architecture (the series' opening topic) applied to a store that's constantly being written to by the same system that's reading from it.

## What Kind of Memory You're Actually Building

**Semantic memory** — Discrete facts about the user or the world: preferences, role, prior decisions. Retrieved by exact or near-exact key lookup, the pattern shown in the agent-memory post's `store.get((user_id, "facts"), key)` example.

**Episodic memory** — Records of specific past interactions or situations, used as examples of how the agent handled something similar before. Retrieved by similarity, not by key, because you're matching "a case like this one" rather than looking up a named fact.

**Procedural memory** — Instructions or behavior the agent has learned to follow, sometimes literally rewriting its own system prompt based on accumulated experience. The rarest of the three in production, and the one that most needs guardrails, since a self-modifying prompt with no review step can drift into behavior nobody signed off on.

Most systems only need semantic memory well-implemented. Reach for episodic memory once you've observed the agent repeatedly failing on a class of situation that a well-chosen past example would resolve; reach for procedural memory only with a human review step in the loop, given how much can go wrong when the thing being modified is the agent's own instructions.

## Retrieval Over Memory Is Still Retrieval

Once a memory store holds more than a handful of facts, retrieving from it has the same problem shape as retrieving from any knowledge base: exact-key lookup only helps when you know the key in advance, and free-text queries need the same embedding-based approach from the retrieval posts earlier in this series.

```python
# pip install langgraph
from langgraph.store.memory import InMemoryStore
from langgraph.store.base import IndexConfig

def embed(texts: list[str]) -> list[list[float]]:
    return embeddings_model.embed_documents(texts)

# A store indexed for similarity search, not just exact-key lookup.
store = InMemoryStore(index=IndexConfig(embed=embed, dims=1536, fields=["content"]))

store.put(("lance", "episodic"), "case-1",
          {"content": "User asked about refunds after a late delivery; "
                      "resolved by offering store credit, which they accepted."})
store.put(("lance", "episodic"), "case-2",
          {"content": "User asked to expedite shipping on an already-placed "
                      "order; resolved by contacting the carrier directly."})

# Similarity search over episodic memory, same mechanics as document retrieval.
similar_cases = store.search(("lance", "episodic"), query="the shipment is late again", limit=2)
```

<!-- VERIFY before publishing: confirm IndexConfig's current field names and the search() call's signature against the langgraph.store.base reference for your pinned version; the store's indexing API has been an active area of change. -->

This is exactly the hybrid-search and reranking machinery from the beginning of this series, applied to a corpus the agent itself is continuously writing, not a static document set someone ingested once.

## Consolidation: The Problem Static Retrieval Doesn't Have

A document corpus is written once and read many times. A memory store is written constantly by the same agent that reads it, which creates a problem retrieval-over-documents never faces: raw interaction logs accumulate faster than they're useful, and without a consolidation step, "long-term memory" becomes an ever-growing pile of near-duplicate, increasingly stale entries that degrade retrieval quality rather than improving it.

```python
def consolidate_episodic_memory(user_id: str, store, llm) -> None:
    """Periodically compress recent raw interactions into durable facts,
    rather than keeping every interaction as its own memory forever."""
    recent = store.search((user_id, "raw_interactions"), query="", limit=50)
    if len(recent) < 20:
        return   # not enough volume yet to be worth consolidating

    summary_prompt = (
        "Given these interaction logs, extract 3-5 durable facts or "
        "patterns about this user worth remembering long-term. Ignore "
        "one-off details that won't recur.\n\n" +
        "\n".join(r.value["content"] for r in recent)
    )
    facts = llm.invoke(summary_prompt).content

    for i, fact in enumerate(facts.split("\n")):
        if fact.strip():
            store.put((user_id, "facts"), f"consolidated-{i}", {"content": fact})

    for r in recent:
        store.delete((user_id, "raw_interactions"), r.key)   # raw logs served their purpose
```

This is the same trimming and summarization discipline from the agent-memory post, applied to the long-term store instead of a conversation thread: compress what's recurring into durable facts, and don't keep the raw material around once it's been distilled.

## Knowledge Graphs as Memory

The knowledge-graphs post earlier in this series covered graphs for representing relationships in a retrieval corpus. The same structure applies to memory when what needs remembering is relational rather than a flat list of facts, "this user's manager is X, who also manages Y's account", is naturally a graph traversal, not a keyword or vector lookup. The trade-off is identical to the one in that earlier post: a graph earns its complexity when relationships between remembered entities are themselves the thing being queried, and is overkill when semantic memory's simple key-value lookups already cover what the agent needs to recall.

## Forgetting Is a Feature, Not a Gap

An unbounded memory store has real costs beyond storage: stale facts actively mislead an agent ("the user prefers X" from eighteen months ago may no longer be true), and retrieval quality degrades as near-duplicate or outdated entries compete with current ones for relevance ranking.

- **Time-to-live on episodic memory.** A past interaction's relevance decays; expire or down-weight entries past a reasonable age rather than treating a two-year-old case the same as one from last week.
- **Explicit overwrite over accumulation for preferences.** "User prefers email over SMS" should replace a prior, contradictory entry, not sit alongside it as two now-ambiguous facts competing at retrieval time.
- **User-initiated deletion.** If people can tell your agent to forget something, that has to actually remove the fact, not just make it harder to surface, both because it's the right behavior and because in many jurisdictions it's a legal requirement.
- **Confidence decay for inferred facts.** A fact the agent inferred rather than one the user explicitly stated deserves lower retrieval weight and periodic re-verification, since an inference can be wrong in ways a direct statement isn't.

## Choosing an Architecture

- **A handful of durable facts per user** — Plain key-value semantic memory, as shown in the agent-memory post. No retrieval machinery needed; you know the keys.
- **A growing set of past interactions worth matching by similarity** — Add vector-indexed episodic memory, with a consolidation job running on a schedule, not indefinitely accumulating raw logs.
- **Relationships between remembered entities matter to the queries you'll run** — Layer in a graph, scoped to the same size-versus-complexity trade-off as the knowledge-graphs post.
- **The agent's own behavior should adapt from experience** — Procedural memory, but only with a human review step on what gets written, given how much is at stake in letting an agent rewrite its own instructions unsupervised.

> A memory system that only ever grows isn't remembering, it's hoarding. The consolidation and forgetting half of memory architecture matters as much as the retrieval half, and it's the half most systems skip.

---
*If your team's agent memory store has never deleted anything, this is worth reading before retrieval quality quietly degrades.*