---
layout: default
title: "RAG Security (2): Relevance ≠ Authorisation — What Changes Once Access Control (ACL) Enters"
---

# RAG Security (2): Relevance ≠ Authorisation — What Changes Once Access Control (ACL) Enters

In Part 1, my point was simple: the center of RAG security isn’t the model itself. It’s the **context layer** in front of it.

And I don't want to view that layer as only "query-time composition." It also includes the artefacts created during ingestion/indexing: embeddings, indexes, caches, logs. Once you look at the lifecycle, the security conversation naturally shifts towards those things.

When I talk with people about this, one question almost always comes up:

> “But the LLM still sees plaintext, doesn't it?”

It's an honest and important question. But if we use it as the only lens, conversations about "RAG security" become tangled. Different security targets get conflated.

The cleanest example that exposes the confusion is **access control (ACL)**.[1]

---

## Targets get mixed

I find it useful to avoid treating RAG security as one axis. A better start is to separate targets:

- **Confidentiality**: does content leak? (data, queries, context, scores)
- **Integrity**: is context polluted or manipulated? (prompt injection, data poisoning)[2]
- **Authorisation**: is access allowed? (ACL, multi-tenancy, policy/roles)
- **Observable signals**: do meaningful traces remain even without plaintext? (logs, caches, metrics, patterns)
- **Propagation**: do model outputs turn into tool calls or real actions?

The “LLM sees plaintext” question typically probes confidentiality, especially at the generation boundary.

But in production, what teams often hit more frequently is authorisation. ACL is the most common shape that authorisation takes. It's a different target, with different failure patterns.

---

## Relevant documents ≠ allowed documents

Retrieval in RAG optimises for relevance. But production context cannot be built from relevance alone.

> Context must be an **authorisation-passed artefact** before it is "evidence."

Once you accept that, the system changes. It’s not enough to produce top‑k. “Are we allowed to show this?” becomes part of the pipeline.

- It’s not “top‑k are candidates,”
- but “top‑k, filtered down to what we are allowed to show, are candidates.”

Retrieval still matters. But now retrieval becomes coupled with an authorisation policy engine.

A simple analogy: a library search result is not the same as a book you can actually open. Search can be accurate and still be blocked by access rules.

---

## Three production frictions once ACL enters

ACL details can go on forever. To keep the argument focused, here are three structural frictions I keep running into.

### 1) Pre-filter vs post-filter: when do you apply authorisation?

To enforce ACL, you end up filtering. The question is *when*. That choice changes system behaviour and changes incident paths.

- **Pre-filter (filter before retrieval)**: search only within an allowed candidate set  
  Pro: easier to reduce leak paths  
  Con: smaller candidate pools can destabilise retrieval quality, performance, and index design

- **Post-filter (filter after retrieval)**: retrieve top‑k first, then apply ACL  
  Pro: easier to preserve retrieval quality  
  Con: top‑k may become empty or unstable after filtering  
  And intermediate signals (“candidates,” “top‑k,” “why it was filtered”) are more likely to end up in caches/logs

This is not a coding preference. It’s a security target choice, and it comes with tradeoffs you should name explicitly.

### 2) Caches/logs/metrics erode the boundary

Production RAG does not run without observability. But once ACL is involved, observability itself can become a new risk.

Even without raw plaintext, these signals can be strong clues:

- which documents became candidates
- what the top‑k results were
- what topics a user repeatedly searches for

So “reduce observable signals” does not mean “turn off logs.” Observability is operationally necessary.

The point is to design observability so it doesn't quietly erode the authorisation boundary: aggregation over raw records, per-tenant separation, strict access controls, strict retention windows. The recurring problem of sensitive data ending up in logs fits this pattern.[3]

### 3) Authorisation is not static (lifecycle problem)

ACL is often simplified as a "per-document table." But in real operations, authorisation changes:

- org/team changes
- document moves/deletes/disposal
- link expiration
- project shutdown
- offboarding / access revocation

At that moment, embeddings/indexes/caches/logs become **long‑lived assets** that create a policy synchronisation problem. It's not "build once and done." Updates, invalidation, deletion, and re-indexing follow.

This is why I increasingly see RAG security as a lifecycle problem in a running system—not only a model problem, and not only a retrieval problem.

---

## Conclusion: if you don’t separate targets, you talk past each other

"The LLM sees plaintext" is important. But it targets confidentiality.

Authorisation is a different target. ACL makes it concrete.

If we don't first align on *which target we are discussing*, choices around encryption, isolation, filtering, policy design, and observability tend to become conflated in one argument.

In the next post, I want to start from a practical question: once context is "authorisation-passed," where does it actually end up living? Caches, logs, indexes, agent artefacts. And how do those long‑lived assets and observable signals reshape the trust boundary over time?

---

## References

[1] OWASP Top 10 2021 — A01: Broken Access Control: https://owasp.org/Top10/A01_2021-Broken_Access_Control/

[2] UK NCSC — Prompt injection is not SQL injection (it may be worse): https://www.ncsc.gov.uk/blog-post/prompt-injection-is-not-sql-injection

[3] MITRE CWE-532 — Insertion of Sensitive Information into Log File: https://cwe.mitre.org/data/definitions/532.html

