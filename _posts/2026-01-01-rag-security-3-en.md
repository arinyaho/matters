---
layout: default
title: "RAG Security (3): Long-lived Assets and Observable Signals Shape Incident Paths"
---

# RAG Security (3): Long-lived Assets and Observable Signals Shape Incident Paths

In Part 2, I argued that once "relevance" and "authorisation (ACL)" get mixed, the system changes. Context is not only "evidence for an answer." It must also be an artefact that has passed access control.

I ended Part 2 with one question:

> "Where does ACL-filtered context end up living?"

This post is Part 3. I want a lens for where incident paths actually emerge.

A RAG system does not only build context. It also creates "things that remain." Once those remain as long-lived assets, they can become the starting point of incident paths.

In this post, "boundary" is not one thing.

- **Authorisation boundary**: who can use which context. ACL defines it.
- **Observability boundary**: how far observable signals travel. Logs, caches, metrics, traces define it.

Even if you respect the authorisation boundary, a wide observability boundary creates leak paths.

---

## The Context Layer: an "Artefact Factory"

If you view RAG as only "question -> context -> LLM," security discussions tend to stick to the plaintext boundary at the LLM input. But in production, the context layer usually contains many operations and artefacts:

- what comes in: untrusted inputs such as documents, web pages, wikis, tool outputs
- what happens: chunking, metadata binding, embeddings, indexing, retrieval, reranking, filtering, summarisation, templates
- what remains: embeddings/indexes/caches/logs/metrics/debug traces

In other words, the context layer builds context per request and also stores multiple artefacts for operations and performance. I will call those stored artefacts **long-lived assets**.

Long-lived assets tend to share two properties:

1) they are stored and reused (not request-scoped)
2) if leaked, their blast radius is large

That leads to a natural question:

> "What does the system leave behind, and who can see it?"

---

## Long-lived Assets: by-products that accumulate

Long-lived assets often look like this:

- source text / chunks (knowledge base)
- embedding store, vector index, text index
- candidate/result (top-k) caches, rerank caches
- retrieval traces (which documents were candidates and why they were dropped)
- logs/metrics (latency, errors, quality indicators, reproduction samples)
- datasets and replay queues for evaluation and reproduction

These assets are not created with malicious intent. They exist because operations need performance, quality, and debugging.

The problem starts once ACL (access control) enters, as discussed in Part 2. Even if "documents that must not be shown" are removed from the final context, candidates/results/traces can still remain in logs and caches. At that point, the observability boundary expands and can bypass the authorisation boundary. This is why MITRE classifies sensitive data in logs as a CWE category.[1]

---

## Agents: more long-lived assets

Once you add agents or workflows, assets increase further.

An agent rarely only "retrieves." It summarises, plans, calls tools, and stores intermediate outputs. That creates new artefacts:

- conversation summaries / notes (long-term memory)
- tool outputs (tables, JSON, log dumps, code snippets)
- intermediate deliverables (drafts, checklists, reproduction steps)
- operational "state" and work history

These artefacts help UX and productivity. But from a security perspective, they are also "another storage layer." A summary/note can compress sensitive information into a smaller and more portable form. You can treat it as a kind of "compression leakage": even without the source text, a summary alone can leak the conclusion.

---

## Value vs metadata

The simplest labelling I want to use in Part 3 is this split:

- **value**: the content/value itself (source text, context text, embeddings, scores)
- **metadata**: patterns/traces (access patterns, frequency, timing, result sizes, document IDs, log events)

Many defences are good at hiding value, but metadata tends to remain. Even in encrypted search, access-pattern leakage has long been discussed as an attack surface.[2]

RAG systems also often confuse whether a "derived representation" is value or metadata. For example, an embedding is not the original text, but there are studies showing it can reveal information at near-text level under certain conditions.[3] So I try to avoid treating embeddings as harmless optimisation by-products.

This is not the same as claiming "embeddings are identical to plaintext." Risk depends on your threat model, your model/data conditions, and what an attacker can observe. My point is simply:

> If you do not separate "what remains" into value vs metadata, security conversations tend to drift into either hype or false certainty.

---

## Observability Boundary: where signals reach

It is unrealistic to "just reduce logs/caches/metrics" to reduce what remains. Operationally, observable signals are necessary for the system to run. That calls for a different stance:

> Observable signals are also assets. They need access control, isolation, and retention design.

In ACL-enabled RAG systems, a poorly designed observability boundary allows observable signals to bypass the authorisation boundary.

- the UI hides content via ACL
- but candidate IDs or titles remain in debug traces/logs/caches
- those signals are then shared with a wider set of people/tools/tenants

This is not about the generation boundary. It is about observable signals travelling beyond the observability boundary. That is what changes the incident path.

---

## Four incident paths

Importance varies by organisation and threat model. I do not claim this is exhaustive; I simply find it a useful diagnostic partition in production.

### 1) Reproduction data: tickets/work chat/email

To reproduce a quality issue, teams often paste "top-k results + similarity + filtering info" into tickets. Those artefacts then spread into ticketing systems, work chat, and email. At that point, observable signals move beyond the observability boundary.

### 2) ACL changes: cache/index persistence

Authorisation changes for many reasons (team changes, offboarding, link expiry...). But caches/indexes/logs can persist longer. "Not accessible now" becomes "still exists somewhere." This asymmetry can turn into a long-lived-asset risk.

### 3) Snapshots/backups: replication spread

Long-lived assets get backed up and replicated. Add test environments, analytics pipelines, long retention for cost or compliance, and you may end up with replicas that have weaker boundaries than the primary system.

### 4) Agent memory: a "new storage layer"

Summaries and memory are convenient. But they can be reused in the wrong context due to a small mistake. They can be dumped for testing/evaluation. They can be collected for analysis. And because summaries compress value, they can leak more than expected.

---

## Next question: make value less visible

This post may sound pessimistic. But I find this lens makes security conversations less emotional.

- what to hide (value)
- what will remain (metadata)
- what becomes long-lived assets
- which incident paths those assets create

If you write this down explicitly, the discussion can move away from "perfect security" towards being honest about "what remains."

In the next post, I want to move to this question:

> "Can we make the value of long-lived assets less visible?"

This is also why crypto-based approaches like FHE can feel attractive. But whatever design you choose, it is hard to make the remaining traces truly zero.

---

## References

[1] MITRE CWE-532 — Insertion of Sensitive Information into Log File: https://cwe.mitre.org/data/definitions/532.html

[2] PoPETs 2017 — Leakage abuse attacks against searchable encryption: https://petsymposium.org/popets/2017/popets-2017-0034.php

[3] EMNLP 2023 — Text embedding inversion: https://aclanthology.org/2023.emnlp-main.765/
