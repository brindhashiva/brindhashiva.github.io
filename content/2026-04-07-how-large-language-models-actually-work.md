Title: How Large Language Models Actually Work
Date: 2026-04-07
Category: GenAI
Tags: GenAI, LLM, AI Basics, Machine Learning
Slug: how-large-language-models-actually-work
Status: published

Ask most people how an LLM works and you'll get "it's trained on the internet." True, but it explains nothing about why that training produces something that can hold a conversation, write code, or summarize a contract. The actual mechanism is simpler than people expect, and stranger than "it's trained on the internet" suggests.

## The Core Mechanism

**Next-token prediction** — At its core, an LLM does one repeated thing: given everything so far, predict the most likely next piece of text (a "token"). Do that once, and you get a word. Do it thousands of times in a row, each prediction feeding into the next, and you get paragraphs, code, and coherent answers.

**Tokens, not words** — Models don't process whole words. Text gets broken into tokens — sometimes a full word, sometimes a fragment — and every prediction the model makes is really a probability distribution over which token comes next.

## How Training Actually Shapes This

- Pretraining: the model reads enormous amounts of text, learning statistical patterns of language — grammar, facts, reasoning patterns, style — by repeatedly predicting the next token and adjusting when it's wrong.
- Fine-tuning: after pretraining, the model is further trained on curated examples of following instructions, which is what turns a raw text-predictor into something that behaves like an assistant.
- Reinforcement learning from feedback: human (or AI) preferences are used to further steer the model toward responses people actually find helpful, honest, and safe.

## Why This Explains the Weird Parts

**Why they hallucinate** — Because the model is predicting plausible text, not looking anything up, it can generate a fluent, confident-sounding sentence that's simply wrong — the prediction was statistically reasonable, not factually verified.

**Why context matters so much** — Every prediction depends entirely on what's already in the conversation. Give the model more relevant context, and its next-token predictions get correspondingly better grounded.

> An LLM isn't retrieving an answer from memory. It's predicting the most statistically plausible continuation of the text you gave it — which is usually right, and occasionally confidently wrong in exactly the same tone.

## The Architecture Underneath, Briefly

Most modern LLMs are built on the transformer architecture, which lets the model weigh the relevance of every other word in the input when predicting the next one — instead of processing text strictly left to right, one word informing only the next. This is what allows the model to connect a pronoun to a name mentioned several sentences earlier, or weigh a caveat stated at the start of a long prompt.

## Why This Matters Practically

Understanding "next-token prediction, shaped by training" rather than "the AI knows things" changes how you use these tools. You stop expecting perfect recall and start giving them the context they need to predict well — which is the actual lever you have as a user.

---

*If this made LLMs feel less like magic and more like a mechanism you can reason about, share it with someone who still thinks the model is "looking things up."*