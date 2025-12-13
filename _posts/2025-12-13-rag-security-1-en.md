---
layout: post
title: "The Center of RAG Security Is the Context Layer"
---

RAG is one word, but the implementations vary wildly. Some teams start with dense retrieval; others go hybrid. Then come reranking, filtering, summarization, and prompt templates. Some even move toward graph-style flows that decide *which evidence to fetch, when*.

So when the topic turns to security, a question comes first:

> “With so many variations, can we really argue about security using the shared concept of ‘RAG’?”

I think that question points at the core. The more diverse the implementation becomes, the easier it is for the conversation to fragment into a debate about details.

This is Part 1 of my attempt to organize the idea. Rather than claiming a definitive conclusion, I’m proposing a **lens** for where to start so we don’t lose the plot.

Instead of listing defenses, I want to first sketch where the **trust boundary and observability surface** start to wobble once a context layer is introduced.

---

## Common denominator: the context layer

If you try to define RAG by specific implementation details, there’s no end to it. But if you define RAG as a pattern, a common denominator appears.

> **Question → (build context) → the LLM sees it**

Dense vs hybrid vs rerank vs graph vs “prompt engineering” are different ways of implementing what happens inside the parentheses. In that sense, RAG is less a name for a retrieval method and more a way of adding a **context construction layer** to the system.

What matters to me is that this layer isn’t just “fetch a few documents.” It tends to produce multiple artifacts:

- raw text / chunks
- candidate lists
- (sometimes) intermediate representations such as embeddings and indexes
- retrieval / reranking / filtering outputs
- the final context (the LLM input)

And these artifacts usually come with operational surfaces: caches, logs, storage, metrics.

That’s why I find the following analogy useful:

> RAG is not simply “adding more data to answer better.” It’s closer to adding a **control plane** that decides *what counts as evidence* for the answer.

If the model is the data plane, the context layer becomes a higher-level mechanism that **indirectly steers** the model’s behavior.

---

## A shifted trust boundary

Even an LLM used alone has security issues. But once RAG is added, the center of gravity often shifts from “the model itself” to the **system’s trust boundary**.

A typical RAG-enabled system looks like this:

- untrusted inputs (documents, web pages, wikis, customer-provided files, partner data, tool outputs, …) enter the system
- these inputs are curated into **context**
- the model treats that context as “evidence” and answers or acts on it

The key point is simple:

> **Untrusted input enters the model’s decision path.**

Risks of this kind are often discussed under names like prompt injection[1][2].

I see “context” not as mere data, but as the model’s *situation definition*—a premise for action. Change the context and the same model produces different answers; in tool-using systems, it may call different tools and make different decisions. Context becoming “evidence” is already risky; once tools are involved, context can become a **trigger for actions**, which makes trust boundary design even more sensitive.

Meanwhile, the context layer tends to grow more complex over time. What starts as “one retrieval step” becomes a pipeline: filters, reranking, summarization, rules, routing—more stages.

As stages increase, the same things usually happen from a security perspective:

- more input paths
- more observation points (logs/caches/storage/metrics)
- more surface area for mistakes

That’s why “RAG security” feels incomplete if we only talk about the model. RAG is, by design, an additional layer *in front of* the model.

---

## Security objective: observability surface

Security conversations often collapse into “is plaintext exposed?” That matters. But in RAG systems, a question one level up is often more decisive:

> **Who can observe what?**

Here, “observation” is not limited to plaintext. RAG systems generate many meaningful intermediate signals:

- which documents became candidates
- which documents ended up in top‑k results
- which queries repeat (patterns/frequency)
- what topics users appear to seek (behavioral signals)
- and where all of this persists (logs/caches/metrics)

Observability is not only about plaintext—derived representations (e.g., embeddings) can also become sensitive signals depending on system conditions, so *what remains* matters.

The catch is that protecting *every* intermediate signal in the same way is rarely realistic. Whatever privacy-enhancing technology (PET) you pick, applying it uniformly across all paths tends to cost performance, money, and quality (accuracy/reproducibility).

So in practice, designs often converge to something like this:

Operationally, we still need observability. The goal is not “delete logs,” but to be stricter about *what we record* and *who can access it*—reducing the side effect of sensitive signals accumulating through observation.

> **Reduce the observability surface, starting with the highest‑leverage areas.**

“High leverage” areas tend to share two properties:

- they accumulate into **long‑lived assets** (stored and reused)
- if leaked, the blast radius is large (patterns/relationships/interests become structurally visible)

In the context layer, candidates and results matter—but more fundamentally, the **intermediate artifacts created for storage and search** (e.g., embeddings, indexes, caches, logs) often become long‑lived and widely reused as the system scales.

So the direction I keep coming back to is not “expand the trust boundary,” but closer to:

> **Shrink the untrusted region, and minimize the signals it can observe.**

---

## Three questions: assets · observability · propagation

No matter how RAG is implemented, I’ve found the following three questions keep the discussion from drifting.

1) **Assets**: What do we actually need to protect?
   - source data, context, retrieval logs, candidate lists, access control (ACL), (if any) keys

2) **Observability**: Who can see what?
   - not only plaintext, but meaningful intermediate signals (results/frequency/patterns/logs)

3) **Propagation**: How far can an input influence the system?
   - context → model output → (if present) tool calls → data access / real actions

These questions stay valid whether you use dense retrieval, hybrid search, reranking, graph flows, or something else. If anything, they matter more as the implementation grows more complex.

---

## Next question

If we treat RAG as “just search,” security discussions tend to orbit around the model. If we treat RAG as the addition of a **context layer**, it becomes clearer where trust boundaries shift and where observability needs to be redesigned.

This isn’t only a matter of technical aesthetics. In real environments—audits, regulations, partner operations—it’s also about being able to explain **what we can (and cannot) trust**.

In the next post, I want to tackle a question people inevitably ask:

> “But the LLM still sees plaintext, right?”

—and share how I currently think about it through the lens of separating security targets.

(For what it’s worth, frameworks like OWASP are useful less as “authority” and more as shared language for aligning risk. NIST AI RMF and Google SAIF are languages for the same purpose[3][4]. In Part 1, I’m deliberately not enumerating checklists—I’m focusing on why the context layer becomes central.)

---

## References

[1] OWASP Gen AI Security Project — *OWASP Top 10 for Large Language Model Applications*: https://genai.owasp.org/llm-top-10/

[2] UK National Cyber Security Centre (NCSC) — *Prompt injection is not SQL injection (it may be worse)*: https://www.ncsc.gov.uk/blog-post/prompt-injection-is-not-sql-injection

[3] NIST — *Artificial Intelligence Risk Management Framework (AI RMF 1.0)* (NIST AI 100-1, PDF): https://nvlpubs.nist.gov/nistpubs/ai/nist.ai.100-1.pdf

[4] Google — *Secure AI Framework (SAIF)*: https://saif.google/
