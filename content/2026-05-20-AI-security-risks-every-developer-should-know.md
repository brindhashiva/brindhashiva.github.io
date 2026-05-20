Title: AI Security Risks Every Developer Should Know
Date: 2026-05-20
Category: GenAI
Tags: GenAI, LLM, AI Security, Application Security
Slug: ai-security-risks-every-developer-should-know
Status: published

Traditional application security has decades of hardened instincts around it — sanitize input, validate output, never trust the client. LLM-powered applications break enough of those assumptions that a genuinely careful developer can still ship something exploitable, simply because the threat model is new enough that the old instincts don't fully cover it.

## Prompt Injection: The Headline Risk

**Direct prompt injection** — A user directly instructs the model to ignore its original instructions — "ignore all previous instructions and instead..." — attempting to override the system prompt or intended behavior through the input itself. This is the most talked-about LLM vulnerability, and defending against it fully is still an open problem, not a solved one.

**Indirect prompt injection** — Malicious instructions hidden inside content the model reads as data — a document, a webpage, an email — rather than typed directly by the user. This is the more dangerous variant precisely because the person operating the application never typed the malicious instruction at all; the model encountered it while doing its job.

## Where This Actually Bites

- A support bot with access to internal tools reads a user-submitted ticket containing hidden instructions, and takes an action the ticket author was never authorized to trigger.
- An agent summarizing a webpage encounters injected text instructing it to leak the conversation's earlier context back to the page's author.
- A document-processing pipeline ingests a file with embedded instructions that redirect the model's next tool call somewhere it shouldn't go.

## Data Leakage Risks

**Training data leakage** — Under certain conditions, a model can reproduce fragments of its training data verbatim, which becomes a genuine problem if sensitive or proprietary data was ever part of that training set.

**Context leakage** — In a multi-tenant application, a bug in how conversation context or retrieved documents are scoped can expose one user's data to another user's session — a failure mode with no equivalent in typical stateless web APIs, because LLM applications often carry much more state per request.

> The most dangerous prompt injection isn't the one a user types at you. It's the one hiding inside a document your own system asked the model to read.

## Excessive Agency Is Its Own Risk Category

Giving a model direct access to tools — sending emails, executing code, modifying records — multiplies the blast radius of every other vulnerability on this list. A prompt injection that would otherwise just produce a weird text response becomes an unauthorized email sent or a database record altered, once the model has the tools to actually act on bad instructions.

## Practical Defenses Worth Knowing

- Treat all retrieved content and user input as untrusted, the same instinct long applied to any other external data in software.
- Scope tool permissions as narrowly as possible — a model that can only read, not write, limits how much damage an injected instruction can actually do.
- Add a human approval step before any high-stakes or irreversible action a model's tool call could trigger.
- Log and monitor model inputs and outputs, not just application-level logs, since the attack surface lives in that content.

## Why This Deserves the Same Rigor as Any Other Security Domain

LLM applications are still applications, and everything that made input validation and least-privilege access non-negotiable in traditional software applies here too — it just needs to be re-derived for a system where the "input" can include an entire document, and the "logic" is a model whose behavior isn't fully predictable in advance.

---

*If this made LLM security feel like a real discipline rather than an afterthought, share it with someone building an agent with more tool access than they've actually threat-modeled.*