Title: Knowledge Graphs and RAG
Date: 2026-04-30
Category: GenAI
Tags: RAG, Knowledge Graphs, GraphRAG, Neo4j, LangChain, Text2Cypher
Slug: knowledge-graphs-and-rag
Status: published

Vector RAG answers "which passages sound like this question?" It struggles with "which suppliers of Company X also supply its competitors?" or "how many incidents involved this service last quarter?": questions about relationships, multi-hop chains, and aggregation, where the answer isn't in any single chunk. Knowledge graphs store those relationships explicitly, and an LLM can be pointed at them. The honest version of this topic is less exciting than the hype: graphs are powerful for a specific class of question and an expensive detour for the rest.

## The Vocabulary

**Knowledge graph** — Data stored as nodes (entities like people, companies, services) and typed edges (relationships like `IS_CEO_OF`, `DEPENDS_ON`). Facts are triples: subject, relationship, object. Because relationships are first-class, multi-hop questions become traversals instead of guesses.

**Graph extraction** — Using an LLM to read unstructured text and emit nodes and relationships. This is how most RAG-oriented graphs get built. It costs at least one LLM call per chunk and is the main source of noise in the graph.

**Text2Cypher** — Having an LLM translate a natural-language question into a Cypher query (Neo4j's query language), running that query, and answering from the rows. Precise for structured questions, brittle when the model writes invalid or wrong queries.

**GraphRAG** — An umbrella for retrieval that uses graph structure. Two common flavors: query-time graph traversal (find entities, expand their neighborhoods), and Microsoft's approach of building an entity graph, clustering it into communities, and pre-summarizing those communities to answer corpus-wide "what are the main themes?" questions that plain chunk retrieval handles poorly.

**Entity resolution** — Deciding that "Acme," "Acme Corp.," and "ACME Corporation" are one node. Skip it and your graph fragments into disconnected duplicates, and traversals silently miss connections.

## When Each Approach Fits

- **Vector (plus BM25 and reranking)** — Questions answerable from one or a few passages: docs Q&A, policy lookup, support. Cheapest to build and maintain. This should be your default.
- **Text2Cypher over a graph** — Precise relational or aggregation questions over well-defined entities ("how many," "which are connected to"). Best when you already have or can design a clean schema.
- **Graph-augmented retrieval (vector hits, then graph expansion)** — Questions that start fuzzy but need connected context: find the relevant entity by semantic search, then pull its neighbors for the prompt.
- **Community-summary GraphRAG** — Global questions across a whole corpus ("what themes recur across these 5,000 reports?"). Heavy indexing cost, so justify it against a real need.

## Code: Build a Graph and Query It

Building and querying a graph with `langchain-neo4j`. Extraction is constrained to an explicit schema, which is the single most effective way to keep the graph clean:

```python
# pip install langchain-neo4j langchain-openai neo4j
import os
from langchain_core.documents import Document
from langchain_neo4j import Neo4jGraph, GraphCypherQAChain, LLMGraphTransformer
from langchain.chat_models import init_chat_model

llm = init_chat_model(
    os.environ["CHAT_MODEL"], model_provider=os.environ["CHAT_PROVIDER"]
)

graph = Neo4jGraph(
    url=os.environ["NEO4J_URI"],
    username=os.environ["NEO4J_USERNAME"],
    password=os.environ["NEO4J_PASSWORD"],
)

# 1. Extract entities and relationships under an explicit schema
transformer = LLMGraphTransformer(
    llm=llm,
    allowed_nodes=["Person", "Company", "Location"],
    allowed_relationships=[
        ("Person", "IS_CEO_OF", "Company"),
        ("Company", "HAS_HEADQUARTERS_IN", "Location"),
    ],
)

docs = [Document(page_content=(
    "Tim Cook is the CEO of Apple. Apple has its headquarters in California."
))]
graph_docs = transformer.convert_to_graph_documents(docs)
graph.add_graph_documents(graph_docs, include_source=True)

# 2. Answer questions with Text2Cypher
chain = GraphCypherQAChain.from_llm(
    llm=llm,
    graph=graph,
    verbose=True,
    return_intermediate_steps=True,
    allow_dangerous_requests=True,   # required opt-in; see security note below
)

out = chain.invoke({"query": "Who is the CEO of the company headquartered in California?"})
print(out["result"])
print(out["intermediate_steps"])   # inspect the generated Cypher
```

<!-- VERIFY before publishing: (1) Some current docs import LLMGraphTransformer from langchain_neo4j, while older tutorials import it from langchain_experimental.graph_transformers. Try the import above first and fall back to the experimental path if it fails on your installed versions. (2) allowed_relationships as (source, type, target) tuples is the format in current docs; older versions accepted plain strings. -->

For the graph-augmented pattern, use semantic search to find entry points, then let the graph supply connected context that no single chunk contains:

```python
def expand_entity(name: str, limit: int = 25) -> list[dict]:
    """Pull an entity's immediate neighborhood as prompt context."""
    return graph.query(
        """
        MATCH (e {id: $name})-[r]-(n)
        RETURN e.id AS entity, type(r) AS relationship, n.id AS neighbor
        LIMIT $limit
        """,
        params={"name": name, "limit": limit},
    )

for row in expand_entity("Apple"):
    print(f'{row["entity"]} -[{row["relationship"]}]- {row["neighbor"]}')
```

## The Costs Nobody Puts in the Demo

- **Security.** `GraphCypherQAChain` executes LLM-generated Cypher against your database, which is why it demands `allow_dangerous_requests=True`. Connect with narrowly scoped, read-only credentials. Without them, a cleverly worded question can become a write or a delete.
- **Extraction cost and noise.** Every chunk costs an LLM call, and extraction quality varies by document type. Constrain the schema, sample the output, and expect to iterate.
- **Schema drift.** Your ontology will change. Decide up front how you'll migrate an existing graph when it does.
- **Freshness.** Updating a graph when a document changes means finding and retracting the facts it used to support, which is harder than replacing a few vectors.
- **Evaluation.** Score the graph path with the same golden-set discipline as the rest of your pipeline: check the generated Cypher, not just the final answer.

My rule of thumb: build vector + BM25 + reranking first, then measure which of your real questions still fail. If they fail because the answer spans relationships, a graph is justified. If they fail because of chunking or retrieval quality, a graph will make things worse and more expensive.

> A knowledge graph doesn't make retrieval smarter; it makes relationships queryable. If your questions aren't about relationships, you've built an expensive index.

---
*If a colleague is proposing GraphRAG for a docs chatbot, pass this along before the ontology workshop gets scheduled.*