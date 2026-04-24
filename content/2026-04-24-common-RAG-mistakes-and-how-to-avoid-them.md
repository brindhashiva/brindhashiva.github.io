Title: Common RAG Mistakes and How to Avoid Them
Date: 2026-04-24
Category: GenAI
Tags: GenAI, LLM, RAG, Retrieval
Slug: common-rag-mistakes-and-how-to-avoid-them
Status: published

Every RAG pipeline works in the first demo. That's not the same as working well, and the gap between the two shows up almost entirely in a handful of repeatable mistakes — ones that don't throw an error, they just quietly return worse answers than the pipeline should be capable of.

## Mistake One: Bad Chunking

**The problem** — Splitting documents into chunks that are either too small, losing surrounding context the model needs to make sense of the fragment, or too large, diluting the relevant part of a chunk with unrelated text around it. This is the single most common reason a RAG system underperforms despite every other component being set up correctly.

**The fix** — Test chunk size against your actual documents and actual questions, not a default number from a tutorial. Overlapping chunks slightly, so context doesn't get cut off at an arbitrary boundary, usually helps more than people expect.

## Mistake Two: Treating Retrieval as "Good Enough" Without Measuring It

Most teams evaluate the final generated answer and never separately check whether the retrieval step actually returned the right chunks in the first place. If retrieval is silently failing, no amount of prompt tuning on the generation side will fix it — you're optimizing the wrong half of the pipeline.

- Evaluate retrieval and generation separately, not just the end-to-end answer.
- Build a small test set of real questions with known correct source documents, and check whether retrieval actually surfaces them.
- Track retrieval metrics over time, not just once at launch.

## Mistake Three: Ignoring Metadata and Filtering

Pure semantic similarity search doesn't know that a document is outdated, from the wrong department, or superseded by a newer version — it just knows the text is similar. Skipping metadata filtering (by date, source, category, permissions) means a technically "relevant" but wrong or stale chunk can outrank the one you actually needed.

> Semantic similarity tells you what's related. It doesn't tell you what's current, authoritative, or allowed to be shown to this particular user — that's a separate problem retrieval alone doesn't solve.

## Mistake Four: Retrieving Too Few or Too Many Chunks

Retrieving too few chunks risks missing the answer entirely if it's split across multiple sections. Retrieving too many crowds the model's context with irrelevant text, which can measurably degrade answer quality even when the right chunk is technically in there somewhere. The right number is a tuning question, not a fixed default.

## Mistake Five: No Fallback for "Nothing Relevant Found"

When retrieval genuinely finds nothing relevant, a poorly built pipeline still hands the model *something* — and the model, doing what it does, generates a confident answer anyway. A well-built pipeline needs an explicit path for "the retrieved context doesn't actually answer this," rather than letting the model paper over a retrieval failure.

## The Pattern Underneath All Five

Every one of these mistakes comes from treating RAG as "add a vector database and you're done" instead of treating it as a search problem that happens to feed an LLM. Search systems have always needed tuning, evaluation, and edge-case handling — RAG doesn't get to skip that just because an LLM is generating the final sentence.

---

*If this saved you from debugging a "the model just isn't that good" problem that was actually a retrieval problem, share it with someone deep in a RAG pipeline that isn't performing.*