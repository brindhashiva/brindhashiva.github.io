Title: Model Context Protocol (MCP) Explained
Date: 2026-05-13
Category: GenAI
Tags: MCP, Model Context Protocol, Tool Calling, Interoperability, Anthropic
Slug: model-context-protocol-mcp-explained
Status: published

Before MCP, connecting an agent to ten external systems meant writing ten bespoke integrations, and none of that work carried over to the next agent framework you tried. MCP standardizes the other half of the equation: not what a model can reason about, but how any AI application discovers and calls tools, reads external data, and uses reusable prompt templates through one shared interface. Anthropic published it in late 2024; it's since become the common integration layer across most major agent frameworks and AI applications, and Anthropic donated the protocol to the Linux Foundation's Agentic AI Foundation in December 2025.

## The Core Model

**Host** — The AI application the user actually interacts with (Claude, an IDE, a custom agent). The host is what a person opens.

**Client** — A connector living inside the host, responsible for exactly one MCP server. A host with five connected servers runs five clients internally, each maintaining its own connection.

**Server** — An external process exposing capabilities to clients. Can be local (a filesystem server running as a subprocess on your machine) or remote (a hosted service reached over HTTPS).

**Tool** — An executable action a server exposes: run a query, create a ticket, write a file. This is the same concept as the tools covered in the tool-calling post earlier in this series, just exposed over MCP's standard interface instead of defined inline in your own code.

**Resource** — Read-only contextual data a server provides: file contents, a database schema, an API response. Distinct from a tool because a resource is meant to be read into context, not executed.

**Prompt** — A reusable, parameterized template a server offers, such as a system prompt or a set of few-shot examples, so that prompt engineering can be shared and versioned the same way tools are.

Every message on the wire is JSON-RPC 2.0: a request, a notification, or a response, transported over either stdio (the client spawns the server as a local subprocess and talks over stdin/stdout, the default for anything running on the same machine) or Streamable HTTP (for remote servers reachable over the network).

## Why This Solves an Actual Problem

Without a shared protocol, connecting M AI applications to N tools requires up to M×N custom integrations, one per pairing, each with its own auth pattern, its own way of describing what the tool does, and its own failure modes. A tool written for one agent framework's tool-calling format didn't work in another's. MCP collapses this to roughly M+N: write a server once, and any MCP-compliant client can use it; build a client once, and it can speak to any MCP server. This is the same shape of problem the Language Server Protocol solved for editors and programming languages, and MCP explicitly borrows that framing.

## A Minimal Tool Call, on the Wire

This is the actual JSON-RPC shape a client sends to call a tool, which is useful to see once even if you'll normally use an SDK rather than hand-writing this:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search",
    "arguments": { "query": "quarterly revenue" }
  }
}
```

The server responds with the tool's result in the same JSON-RPC envelope. Discovery works the same way: a client calls a `list` method for whichever primitive it wants (tools, resources, or prompts) to learn what a server offers before invoking anything, which is what makes a client able to work with a server it has never seen before, rather than needing a hardcoded manifest.

<!-- VERIFY before publishing: MCP underwent a major protocol revision on 2026-07-28 that removed the initialize/initialized handshake and protocol-level sessions in favor of a fully stateless design, where every request carries its own protocol version and capabilities inline (in a _meta field) rather than negotiating them once at connection time. This is a very recent, large change; the JSON-RPC method shape above (tools/call, arguments) should still hold, but confirm which spec revision your SDK and the servers you're connecting to actually implement before writing integration code, since a client speaking the new revision cannot talk to a server still on the older, stateful one, and the reverse also fails. Check https://modelcontextprotocol.io/specification for the current revision. -->

That stateless shift, if your SDK has adopted it, matters operationally even if it doesn't change how you call a tool: it's what lets an MCP server run behind an ordinary load balancer with no sticky routing or shared session store, because no request depends on state established by an earlier one.

## Using MCP From an Agent Framework

The pattern across frameworks (LangChain, and others covered in the frameworks-compared post in this series) is the same: adapt an MCP server's tools into the framework's native tool format, so the agent calls them exactly like any other tool from the tool-calling post earlier in this series.

```python
# pip install langchain-mcp-adapters langgraph
# Shape only; check the current package for exact API surface, see note below.
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent

mcp_client = MultiServerMCPClient({
    "filesystem": {
        "transport": "stdio",
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
    },
    "search": {
        "transport": "streamable_http",
        "url": "https://mcp.example.com/search",
    },
})

tools = await mcp_client.get_tools()   # MCP tools adapted to the framework's tool format

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=tools,
    system_prompt="Use the available tools to help with file and search tasks.",
)
```

<!-- VERIFY before publishing: langchain-mcp-adapters' exact API (class names, method names, config shape) moves quickly and I couldn't verify its current surface with confidence. Confirm against the package's own README before publishing this snippet. -->

Once adapted, an MCP tool is indistinguishable to the agent from a tool you defined directly with `@tool`: same tool-call and tool-result loop, same description-quality concerns from the tool-calling post. What MCP changes is where the tool's implementation lives and how many other applications can reuse it unmodified.

## Security Is an Authorization Problem First

A host that connects an MCP server inherits the permissions of everything that server exposes. Connecting a filesystem server scoped to `/workspace` is very different from connecting one scoped to the whole disk, and that scoping happens on the server side, not something the client can retroactively narrow. Treat every MCP server connection with the same scrutiny you'd give a new dependency with filesystem or network access, because that's precisely what it is:

- **Scope servers narrowly.** A filesystem server should expose only the directory a task actually needs, not the whole machine.
- **Review what a server's tools can actually do**, not just their names. A tool named `search` that's actually allowed to write files is a bigger risk than its name suggests.
- **Treat resources and prompts as untrusted input**, the same as any tool result. A server can return crafted content in a resource just as easily as a compromised web page can; nothing about MCP's transport makes server responses inherently trustworthy.
- **Require explicit permission prompts for new tool or resource access**, and don't treat a user clicking through a permission dialog as informed consent unless the dialog actually communicates what's being granted.

## When MCP Is (and Isn't) the Right Layer

- **You're building a tool meant to be reused across multiple agents or applications** — MCP is exactly the point: write once, connect from anywhere that speaks the protocol.
- **You're building a single tool for a single agent in your own codebase** — A plain `@tool`-decorated function (from the tool-calling post) is simpler and has less operational surface. Reach for MCP when reuse or external distribution is the actual goal, not by default.
- **You need an agent to control a third-party desktop or web application it doesn't have an API for** — That's a different problem, computer-use and browser-agent territory, covered in the next post in this series, not what MCP's tool/resource model solves.

> MCP didn't make tool calling smarter. It made tool calling portable, which turns out to be the harder problem once you have more than one agent and more than one team.

---
*Worth sending to whoever's team just built its fourth bespoke tool integration this quarter instead of one MCP server.*