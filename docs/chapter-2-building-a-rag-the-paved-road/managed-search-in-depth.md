# Managed Search in Depth

> Supporting material for [Chapter 2 — Building a RAG: The Paved Road](chapter-2-building-a-rag-the-paved-road.md)
>
> Chapter 2 argued that a managed search product should be the default for most teams. This document is the detail behind that claim: what these services actually do, where they genuinely fall short, how to evaluate one against your corpus, and how to avoid the decision becoming permanent.

---

## 1. What is actually being managed

It is worth being precise about the scope, because "managed search" understates it considerably. What is managed is the entire retrieval pipeline:

| Stage | What the service does | What it would cost you to build |
| --- | --- | --- |
| **Connection** | Authenticated, incremental sync from source systems | Per-source clients, token lifecycle, change detection, backfill |
| **Extraction** | Format-aware text and structure recovery | OCR, table parsing, layout analysis, per-format handling |
| **Chunking** | Segmentation with overlap and structure awareness | Strategy design, tuning, boundary handling |
| **Embedding** | Model hosting, batching, throughput management | Inference infrastructure, rate limiting, retries, cost control |
| **Indexing** | Vector plus keyword index, kept current | Vector store operation, index maintenance, compaction |
| **Retrieval** | Hybrid search, filtering, reranking | Ranking implementation, fusion, tuning |
| **Entitlement** | Per-document permission filtering at query time | An access-control engine synchronised with every source |

The last row is the one that most often decides the matter. Reimplementing source-system permissions, keeping them synchronised, and applying them at query latency is a substantial and permanently ongoing engineering commitment. Very few teams should take it on.

---

## 2. Where the real value sits

### Incremental sync

A first crawl is easy. Keeping an index correct as the corpus changes is not. It requires detecting edits, additions, moves, permission changes and — hardest — deletions, then propagating each without reprocessing everything.

Deletion detection is the usual casualty of hand-built pipelines. A document removed at source lingers in the index and keeps retrieving. Under a deletion request or a retention policy, that is a compliance failure, not a bug. See [Chapter 6](../chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md).

### Extraction quality

Test this with your worst documents. Specifically:

- **Scanned PDFs** — is OCR applied, and is the output usable?
- **Tables** — is structure preserved, or flattened into unreadable text? A flattened table is worse than an omitted one: it retrieves and misleads.
- **Multi-column layouts** — is reading order correct, or are columns interleaved?
- **Embedded images with text** — diagrams, screenshots, scanned signatures.
- **Headers, footers, navigation** — suppressed, or ingested as content?

The reason to test the worst cases is that extraction fails silently. Nothing raises an alarm; the answers just get quietly worse.

### Permission-aware retrieval

The mechanism differs by product, but the pattern is consistent: permissions are captured at ingestion alongside content and applied as a filter at query time against the calling identity.

Two things to verify:

1. **Propagation latency.** If someone's access is revoked, how long until the index reflects it? The acceptable window is set by what the content is, not by the technology: for most internal material a short lag is tolerable, for anything where the revocation is the point — a departure, a deal team, a disciplinary file — it is not. Establish the actual figure, and state it as a commitment rather than discovering it during an incident.
2. **Granularity.** Document-level is the norm. Section- or field-level is rarer. If your corpus has documents where only part is restricted, ask directly.

This capability is what makes the entitlement-before-retrieval rule in [Chapter 3](../chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md) achievable without every team building an authorisation engine.

---

## 3. Where managed search genuinely falls short

An honest list. These are the legitimate reasons to move to Tier 2.

**Relationship traversal.** "Which suppliers are indirectly exposed to this sanctioned entity?" is a graph question. Similarity search cannot answer it at any quality. See [Graph RAG](../chapter-1-why-one-central-rag-fails/graph-rag-explained.md).

**Structure-critical content.** Source code, deeply nested specifications, legal documents where clause hierarchy determines meaning.

Be precise about what is missing here, because the obvious version of this argument is out of date. Managed services increasingly offer layout-aware parsing that detects headings, tables and lists, chunks on those boundaries rather than on arbitrary character counts, and carries the surrounding headings into each chunk as context. That covers a great deal of what used to force teams off the paved road, and it should be tested before it is dismissed.

What layout-aware parsing still does not give you is *semantic* structure: that this clause is subordinate to that one, that this function calls that one, that this requirement supersedes an earlier one. Detecting a heading is not the same as understanding a hierarchy. **The test is whether your content's meaning depends on relationships the parser cannot see from the page.** If it does, this is a real reason. If it merely depends on tables and headings, it probably is not any more.

**Mixed semantic and structured retrieval.** Combining unstructured search with live queries against operational data in a single answer, with transactional consistency, is beyond what a search product offers.

**Domain-specific ranking.** If relevance in your domain depends on regulatory status, document authority, effective dates or supersession chains, the tuning surface may not reach far enough.

**Sovereignty and isolation.** Air-gapped, on-premises-only or specific sovereign requirements may simply exclude the managed option. See [Chapter 9](../chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md).

**Cost at extreme scale.** Managed pricing is excellent value at moderate scale. At very high volume, self-operated infrastructure can become cheaper — but model it honestly, including the engineering salaries, before concluding so.

**Unsupported sources.** A proprietary or internal system with no connector. Note the middle option here: push content to a supported location such as object storage rather than abandoning Tier 1 entirely.

---

## 4. Reasons that are not reasons

Stated often, and not sufficient on their own:

- *"We want more control."* Control over what, specifically, and to achieve what outcome?
- *"We want to avoid lock-in."* A reasonable concern, answered by ensuring you can export your source content — not by rebuilding the pipeline. Your corpus is the asset; the index is derived and rebuildable.
- *"Our data is too sensitive."* This needs splitting, because it usually bundles two very different concerns.

  **On protection, the objection is weak.** Managed services support customer-managed encryption keys, private networking and access controls that a team-managed virtual machine will not match in practice. A self-built system maintained by three people alongside their day jobs is very often the less secure option, and the comparison is rarely made honestly.

  **On residency and sovereignty, the objection can be entirely correct, and it is the one to check.** Strong encryption controls do not imply strong residency guarantees, and the two are frequently confused because they appear in the same section of the same documentation page. A service may hold your data in your chosen region and still process a request elsewhere. Managed RAG frameworks in particular tend to offer key management and perimeter controls while explicitly *not* offering data residency commitments — which is a perfectly reasonable product decision and a disqualifying one for some regulated workloads.

  So: ask for the residency commitment specifically, in writing, for the specific service and the specific region. Do not infer it from the encryption story.
- *"We want to use a specific embedding model."* Check before assuming. Managed search services now commonly allow you to bring your own embeddings or select among supported models, which removes this objection outright for most teams. Where it survives is narrower and more interesting: a genuinely domain-adapted model you have trained or fine-tuned yourself, which is a real reason and a rare one.
- *"It won't scale."* Check the published limits against your actual numbers. It usually will.

---

## 5. An evaluation protocol

Do not evaluate on a demo corpus. Evaluate on yours.

**Step 1 — Assemble a realistic sample.** A few hundred documents, deliberately including the ugly ones: scans, big tables, the odd formats, documents with restrictive permissions.

**Step 2 — Write the queries first.** Thirty to fifty real questions with known-correct source documents. Write them before you see any results, or you will unconsciously grade the system on what it happens to do well. This set becomes your evaluation baseline in [Chapter 4](../chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) — the work is not throwaway.

**Step 3 — Ingest and inspect the extraction.** Before looking at any retrieval quality, read the extracted text for your ten worst documents. Most disappointing evaluations are extraction failures misdiagnosed as retrieval failures.

**Step 4 — Measure retrieval.** For each query, is the correct document in the top five? Report the proportion. Below roughly 70% on a clean sample, investigate extraction and chunking before blaming the product.

**Step 5 — Test entitlement.** Query as a user who should not see certain documents. Confirm they are absent. Revoke access at source; measure how long until retrieval reflects it.

**Step 6 — Measure latency and model cost** at a realistic query rate, then extrapolate to your projected volume.

Two weeks is usually enough for all six steps. That is a small price for a decision that shapes the next several years.

---

## 6. Keeping the decision reversible

Tier 1 should not be a one-way door. Three practices keep it open:

**Own your source content.** The index is derived. As long as the documents remain in systems you control and can export from, you can rebuild elsewhere.

**Keep configuration in version control.** Connector definitions, permission mappings, tuning parameters — as declarative infrastructure, not console clicks. This is what makes environment promotion and disaster recovery routine.

**Hold the interface contract.** Because consumers reach the RAG only through the governed surface defined in [Chapter 3](../chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md), the implementation behind it can change without any consumer noticing. This is the single most valuable property in the whole architecture, and it is the reason the contract is non-negotiable.

---

[← Back to Chapter 2](chapter-2-building-a-rag-the-paved-road.md) | [Tier 1 — Managed →](pipeline-tier-1-managed.md)
