Title: Tokens Explained Simply
Date: 2026-04-08
Category: GenAI
Tags: GenAI, LLM, Tokenization, AI Basics
Slug: tokens-explained-simply
Status: published

"Tokens" is one of those words that gets thrown around constantly around LLMs — token limits, token costs, token windows — without anyone stopping to explain what a token actually is. It's a small concept that quietly explains a lot of behavior people find confusing, from pricing to why a model sometimes struggles with counting letters in a word.

## What a Token Actually Is

**Token** — A token is a chunk of text a model treats as a single unit — sometimes a whole word, sometimes part of one, sometimes just a punctuation mark. "Chatbot" might be one token or split into "Chat" and "bot" depending on the model's tokenizer; a rare or unusual word is more likely to get split into smaller pieces.

**Tokenizer** — The specific algorithm that converts raw text into tokens (and back again). Different models use different tokenizers, which is why the same sentence can cost a different number of tokens depending on which model you're using.

## Why Text Isn't Split Into Words

- Efficiency: splitting into subword chunks lets a model handle rare words and typos gracefully, without needing a separate entry for every possible word.
- Consistency: a fixed, manageable vocabulary of tokens is easier for a model to learn well than an open-ended vocabulary of every possible word.
- Cross-language handling: subword tokenization lets one model reasonably handle multiple languages without a separate word list for each.

## Why Tokens Explain Counting and Spelling Struggles

Because a model doesn't see individual letters — it sees tokens, which often bundle several letters into one unit — asking it to count the letters in a word or reverse a string is genuinely harder for it than it sounds, since it's not operating on the letters directly the way a person reading the word is.

> A model doesn't see "strawberry" the way you do. It sees a small number of token IDs — and counting letters inside one of those tokens isn't a natural operation for it at all.

## Why Tokens Are What You're Actually Paying For

API pricing for LLMs is almost always denominated in tokens, not words or characters, billed separately for what you send in (input tokens) and what the model generates back (output tokens). This is also why a "token limit" or "context window" is really a hard cap on how many tokens — not words — a conversation can hold before older content has to be dropped or summarized.

## The Practical Takeaway

Tokens aren't a technical detail you can safely ignore once you're actually building with these models — they explain your bill, your context limits, and some of the model's stranger failure modes all at once. Roughly, a token is about three-quarters of an English word on average, which is a useful mental shortcut when estimating cost or length.

---

*If this made "tokens" click as more than jargon, share it with someone who's been staring at a token-limit error without knowing what it actually means.*