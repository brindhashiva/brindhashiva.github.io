Title: Tool Calling Explained With Real Examples
Date: 2026-05-05
Category: GenAI
Tags: AI Agents, Tool Calling, LangChain, Function Calling, create_agent
Slug: tool-calling-explained-with-real-examples
Status: published

Tool calling is the mechanism underneath every agent in this series: the thing that lets a model do more than produce text. It's also frequently misunderstood as "the model runs code," which it does not. The model only ever outputs structured intent; your code decides whether, and how, to act on it. That distinction is the entire security and reliability model, and it's worth being precise about before writing another tool.

## The Actual Mechanics

**Tool** — A function exposed to the model with a name, a natural-language description, and an argument schema. The description is not documentation for humans; it is the only information the model has for deciding when to use the tool, so vague descriptions produce vague tool choices.

**Tool call** — The model's output when it decides to use a tool: a name and a set of arguments, emitted as structured data rather than free text. The model does not execute anything. It requests.

**Tool execution** — Your code reading the model's requested name and arguments, running the corresponding function, and returning the result. This step is entirely yours to control, validate, sandbox, or reject.

**Tool result / observation** — The output fed back to the model as a new message, so it can decide what to do next: call another tool, or respond to the user.

The loop is: model proposes a call, your code executes it, the result goes back in, model proposes the next thing. Nothing happens without your code choosing to run it.

## Defining a Tool

The `@tool` decorator turns a plain Python function into something a model can call, inferring the argument schema from the type hints and pulling the description straight from the docstring:

```python
# pip install langchain-core
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city. Use this when the user asks
    about weather conditions, temperature, or whether to bring an umbrella."""
    return fetch_weather_api(city)

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email. Use only when the user has explicitly asked for an
    email to be sent, not merely drafted."""
    return dispatch_email(to, subject, body)
```

The docstring matters more than it looks. "Use only when the user has explicitly asked" is doing real work: it's the difference between a model that drafts an email and one that also sends it uninvited.

## Wiring Tools to a Model

At the low level, `bind_tools` attaches a tool schema to a chat model so its next call can propose a tool use:

```python
# pip install langchain-openai
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")
llm_with_tools = llm.bind_tools([get_weather, send_email])

response = llm_with_tools.invoke("What's the weather in Lisbon?")
print(response.tool_calls)
# [{'name': 'get_weather', 'args': {'city': 'Lisbon'}, 'id': '...'}]
```

Note what didn't happen: `get_weather` was never called. `response.tool_calls` is the model's request; running it and returning a `ToolMessage` is a separate step you write, or one that a higher-level agent construct handles for you.

For the common case, `create_agent` builds the whole loop, the executing, the feeding results back, and the deciding when to stop, for you:

```python
# pip install langchain
from langchain.agents import create_agent

agent = create_agent(
    model="openai:gpt-4o-mini",
    tools=[get_weather, send_email],
    system_prompt="Help the user with weather and email tasks.",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "What's the weather in Lisbon, and email it to me at a@b.com"}]
})
for msg in result["messages"]:
    print(type(msg).__name__, getattr(msg, "content", msg))
```

`bind_tools` gives you the primitive when you need custom control flow (approval gates, custom retry logic, non-standard stopping conditions). `create_agent` gives you the standard loop when the default behavior is enough, which is most of the time.

## Structured Output Is Tool Calling in Disguise

`with_structured_output` (used for the router in the multi-source retrieval post earlier in this series) is built on the same mechanism: for most providers, it defines your schema as a single tool, forces the model to call it, and parses the arguments into your Pydantic model. If you understand tool calling, you already understand how structured output works under the hood, and why it needs a provider that supports tool calls to function.

## Failure Modes Worth Designing For

- **Wrong tool chosen.** Usually a description problem, not a model problem. If two tools have overlapping descriptions, expect the model to confuse them; make descriptions mutually exclusive about when each applies.
- **Malformed or hallucinated arguments.** The schema constrains the shape (a `city: str` argument will be a string), but not the content (it can still be a city that doesn't exist). Validate inside the tool function; never assume the argument is meaningful just because it's well-typed.
- **The model doesn't call a tool it should have.** Often a description that's too narrow, or a system prompt that doesn't establish when tools are expected to be used at all. Test with the exact phrasings real users send, not the phrasing you'd use.
- **A tool with side effects gets called when it shouldn't.** This is a policy problem, not a schema problem, and the schema can't fix it. This is exactly what human-in-the-loop approval gates exist for, covered later in this series: don't rely on prompt wording alone to prevent a destructive call.
- **Tool errors crash the loop.** A tool that raises an exception instead of returning an error string can break agent execution entirely. Catch exceptions inside the tool and return a description of the failure as the result, so the model can react to it (retry differently, tell the user, try another tool) instead of the whole run dying.

> A tool's description is a contract with the model, not a comment for the next engineer. Write it for the reader that actually decides whether to call the function.

---
*Send this to whoever just shipped a tool with the docstring `"""Does the thing."""` and wondered why the model never calls it right.*