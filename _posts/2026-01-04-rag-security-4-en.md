---
layout: default
title: "RAG Security (4): How FHE Changes Incident Paths - and What Still Remains"
---

# RAG Security (4): How FHE Changes Incident Paths - and What Still Remains

This is the last post in a four-part series where I've been trying to get my own thinking straight about RAG security.

- In Part 1, I argued that we should look at RAG security not as a "retrieval feature", but as a lifecycle of the context layer.
- In Part 2, I wrote about how the system changes the moment relevance and authorisation (ACL) get entangled.
- In Part 3, I claimed that long-lived assets and observability signals produced by that lifecycle are what shape real incident paths.

Part 3 ended with a simple question:

> "Can we make the value of long-lived assets less observable?"

## Start with one assumption

To keep this from dissolving into abstractions, let's assume a concrete situation.

- A production incident happens late at night, and you find signs that someone exfiltrated an "index snapshot".
- During incident response, you start asking: "What can they do right now?" and "How much have we actually lost?"
- If you follow the thread from Part 3, you can see where conversations tend to get tangled:
  1) the intuition: "If it's not the original plaintext, it's probably fine"
  2) the habit of collapsing everything into one sentence: "LLMs see plaintext anyway during normal operations, so it's fine"

In this post, I want to answer those questions not by dropping a technology name, but by describing how the incident path changes.

- Which failures become more likely?
- How does the cost of failure (exposure scope, provability, recoverability) change?

Once we state the threat-model assumptions explicitly, we can describe - without overclaiming - what a given PET (e.g., FHE) covers and what it still leaves behind.

For example, if we use FHE (Fully Homomorphic Encryption) to protect long-lived assets, an incident path like "data exfiltration = immediately useful" may shift into a path that requires "exfiltration + key/boundary collapse". Residual threats still remain (below, I keep them fixed as four). A shared language like NIST AI RMF or Google SAIF helps keep those residuals describable without turning the discussion into vibes.[^1][^2]

---

## Policy-based trust vs cryptographic boundary

Here I compare two trust models along two axes. I'm not claiming that one always dominates the other. Real systems are usually hybrids, and the best point depends on organisation and environment.

- Model A: Policy-based trust
  You rely on access control (ACL/RBAC/ABAC), operational processes, and audits. The boundary is primarily about "who is allowed to see what".

- Model B: Threat-model-scoped cryptographic boundary
  You structurally reduce plaintext dependence for some context-layer artefacts, so that the immediacy of a specific incident path goes down. FHE is a representative example in this axis.[^3][^4][^5]

Model B does not promise "perfect secrecy". It is a comparison frame that changes the path and the cost when failure happens.

And this is where the conclusion from Part 3 shows up again. Even if you reduce plaintext zones, observability signals (logs/caches/metrics/access patterns) can remain and spread. The more you talk about Model B, the more the design of the observability boundary becomes critical.

---

## Incident path: theft of stored retrieval artefacts

Among the long-lived assets listed in Part 3, the ones that feel most leveraged to me are the retrieval artefacts that are stored and reused.

- embeddings / vector indices (or text indices)
- candidate/result caches and replay samples
- traces of retrieval/filtering/reranking (including debug logs)

These assets do not exist only transiently per request. They accumulate and get reused. Once they leak, the blast radius grows. From the attacker's perspective, they become a target you can analyse offline, repeatedly, and deeply.

In Model A (policy-based), this path is fairly intuitive. If someone takes your storage/snapshots/index files, they can run offline experiments with what they stole. This is why the intuition "it's not plaintext, so it's safe" can break under mild assumptions. Derived representations (embeddings/scores/index structures) can still carry information that is close to value.

If we assume Model B (cryptographic boundary), the nature of this incident path changes. Under a concrete threat model, the claim is no longer "steal the index and you can search right away", but "steal the index and you still need the key/boundary to collapse before it becomes meaningfully usable". In other words, assuming the boundary assumptions hold, the immediacy of the attack goes down, and incident response changes shape.

But this is not the end of the story. Many things still remain.

- access patterns (which queries are issued how often)
- the shape of results (how distances/scores/rank information leak)
- observability signals produced by the system (caches/metrics/rate-limit logs, etc.)
- and, above all, plaintext outside the context-layer boundary (e.g., final context / model input)

Still, lowering the "immediacy" of a stored-retrieval-artefact theft can change both severity and playbooks for many organisations. This is one reason I don't think RAG security can be reduced to "a model/prompt problem".

---

## What changes - and what doesn't

Model B doesn't erase incident paths. Sometimes it makes "what doesn't change" clearer.

### Operator viewing

In Model A, you manage "should not be seen" through policy and control: separation of duties, approval workflows, and post-hoc audits. But incidents repeat when you combine emergency response, a small number of super-privileged accounts, and misconfiguration.

Model B can structurally reduce this possibility in some segments. But this doesn't make viewing "impossible". It often just pushes the problem to wherever plaintext exists. The conversation shifts to: where do we place the boundary where plaintext appears?

### Incident response and observability signals

The context layer is hard to operate without observability. You end up with caches, logs, retries, sampling, and reproduction tests. A common failure in Model A is that sensitive value ends up in observability signals during incidents - and those "temporary" artefacts last longer than expected.

Even with Model B, the observability-boundary problem remains. Reducing observability signals can be unrealistic. Instead, you treat observability signals as assets and return to designing their access control, isolation, and retention.

### Backups and replicas

Long-lived assets get backed up and replicated. Once you add test environments, analytics pipelines, and long-term storage, the "replica" can become a looser boundary than the original system.

Model B can also change the meaning of exfiltration here, because a snapshot alone may no longer be sufficient to immediately extract value. But this pushes the centre of gravity toward key management and key usage rights.

---

## Residual threats: four items

If an incident path changes, the system does not become "fully safe". I find it useful to keep a fixed list of residual threats, because otherwise the discussion tends to drift.

### 1) The final plaintext boundary (model input/output)

Even if you hide the value of long-lived assets, plaintext still appears where users, applications, and models meet. This boundary can be "pushed" around, but it does not disappear. So you still need to document where plaintext appears, who can access it, and how long it persists.

### 2) Runtime / memory

The moment you use keys, and the moment you materialise plaintext, runtime becomes an attack surface. Insider paths, debugging tools, memory dumps, and agent long-term memory tend to converge here. This is why adding a cryptographic boundary usually forces you to split not only "key management", but also operational privileges (operator/debug/observability permissions) more carefully.

### 3) Key ownership / key custody

If you use FHE, you may shift a path toward "an index snapshot alone is not enough to extract value". But that statement rests on the assumption that keys do not collapse with the same boundary.

So a residual question remains: who holds the keys, where are they used (which process/host/privilege), and how are keys treated in exceptional situations like outages, debugging, and backups?

In practice, the key itself is often not the direct problem - it is the moment keys get replicated. Keys/data copied into test environments, keys leaking into dumps/logs by accident, and abuse of broadly privileged operator accounts are typical paths. If this isn't documented, the claim "we reduced immediacy for snapshot theft" can quickly get walked back.

### 4) Access patterns / observability signals

Even if you hide value, what remains tends to look like metadata. And observability signals are operationally necessary (otherwise you can't operate). The remaining task is to design the observability boundary: who can see it (permissions), whether tenants/environments are isolated (isolation), how long signals persist (retention), and whether raw logs can be replaced with aggregates (aggregation).

---

## Closing the series

If I had to compress what I've been trying to say across this series into one line, it's this:

Don't end RAG security discussions at "does the model see plaintext". Move the discussion to the context-layer lifecycle, long-lived assets, observability signals, and incident paths.

Concepts like FHE help shift the conversation from "technology names" to "failure shapes" - because they can change the cost profile for some incident paths. But residual threats do not disappear. So I'd like to end with the same final question.

> So what do we guarantee - and what do we explicitly document as residual threats?

---

## References

[^1]: NIST - Artificial Intelligence Risk Management Framework (AI RMF 1.0) (PDF): https://nvlpubs.nist.gov/nistpubs/ai/nist.ai.100-1.pdf
[^2]: Google - Secure AI Framework (SAIF): https://saif.google/
[^3]: Craig Gentry - A Fully Homomorphic Encryption Scheme (IACR ePrint 2009/616): https://eprint.iacr.org/2009/616
[^4]: Homomorphic Encryption Standardization Consortium - Homomorphic Encryption Standard: https://homomorphicencryption.org/standard
[^5]: Jung Hee Cheon et al. - Homomorphic Encryption for Arithmetic of Approximate Numbers (CKKS) (IACR ePrint 2016/421): https://eprint.iacr.org/2016/421