Title: Understanding Context Windows
Date: 2026-04-10
Category: GenAI
Tags: GenAI, LLM, Context Windows, AI Basics
Slug: understanding-context-windows
Status: published

"This model has a 200K context window" sounds like a spec sheet detail until you hit its limit mid-conversation and watch the model forget something you told it twenty messages ago. Context windows quietly shape nearly everything about how these tools behave in practice — what they remember, what they cost, and why longer conversations sometimes get worse instead of better.

## What a Context Window Actually Is

**Context window** — The total amount of text, measured in tokens, that a model can "see" at once — including your prompt, any documents you've provided, and the entire conversation history so far. Anything outside that window simply isn't visible to the model when it generates its next response.

**It's a hard limit, not a soft one** — Once a conversation's total token count exceeds the context window, something has to give — older messages get dropped, summarized, or truncated, depending on how the application is built. The model itself doesn't choose what to forget; the system feeding it text decides.

## Why This Explains Common Frustrations

- "It forgot what I said earlier" — usually means the conversation grew past the context window and the earlier content was dropped or summarized away, not that the model has some deeper memory failure.
- "It's getting slower and more expensive" — a longer context window means more tokens processed on every single turn, since the model re-reads the whole visible history each time, not just your newest message.
- "It's ignoring instructions from the start of a long prompt" — models don't always weigh every part of a long context equally; instructions buried in the middle of a very long document can get less attention than ones near the start or end.

## Context Window vs. Memory

It's worth being precise here: a context window is not the same thing as persistent memory. A large context window lets a model reference a lot of text within one session, but that text disappears once the session ends unless the application explicitly saves and reloads it — the model itself isn't "remembering" anything between separate conversations.

> A bigger context window doesn't mean a better memory. It means a bigger desk to spread today's papers on — everything still gets swept off when the session ends, unless something explicitly saves it first.

## How Applications Work Around the Limit

Real applications rarely just let a conversation run until it hits the hard limit. Common strategies include summarizing older parts of a conversation to compress them, retrieving only the most relevant past context instead of all of it (the same idea behind RAG), or explicitly truncating the oldest messages first. Each approach trades some fidelity for staying inside the window.

## Why This Matters When You're Actually Using These Tools

Knowing that context is finite and re-processed every turn changes how you work with these models — front-loading the most important instructions, periodically re-stating key constraints in a long session, and not being surprised when a very long conversation starts losing the thread. It's not a bug; it's the window doing exactly what it's built to do.

---

*If this explained why your long conversations sometimes go sideways, share it with someone who just assumed the model had infinite memory.*