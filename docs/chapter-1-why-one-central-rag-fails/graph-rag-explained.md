# Graph-Based RAG: What It Is, Why It Exists, When It Earns Its Complexity

> Supporting material for [Chapter 1](chapter-1-why-one-central-rag-fails.md). Chapter 1 notes that a team may legitimately need a retrieval approach that ordinary semantic search cannot provide. Graph RAG is the most common of those. This document explains what it is without assuming you have used one.

---

## 1. The limitation graph RAG exists to fix

Conventional RAG is, underneath, a similarity search over independent fragments.

Documents are split into chunks. Each chunk is embedded. A question is embedded the same way. The system returns the chunks whose vectors sit closest to the question's vector.

This works remarkably well for one specific shape of question: **the answer is stated, more or less in full, somewhere in the corpus.** "What is the refund window?" is answerable because a sentence somewhere says what the refund window is. Retrieval's job is to find that sentence.

The limitation is structural. **Chunks do not know about each other.** Each is retrieved on its own merits. Nothing in the index records that this chunk describes a service that calls the service described by that chunk, or that this person approved the contract mentioned in that one.

So conventional RAG struggles with questions where the answer is not *in* any chunk, but *between* them:

- *"If the pricing service goes down, which customer-facing features stop working?"* — no document says this. It has to be assembled by following dependencies outward, possibly several steps.
- *"Which of our active contracts contain a clause that this new regulation invalidates?"* — requires joining contracts to clauses to regulatory concepts.
- *"Who has approved payments to this supplier over the last three years, and who did they report to at the time?"* — a chain of relationships that changes over time.

Throwing more chunks at the model does not help. The relationships were never captured, so there is nothing to reason over. You can retrieve fifty chunks that each mention the pricing service and still not answer the first question, because "which features break" is a property of the *edges*, and the edges were discarded at ingestion time.

---

## 2. What graph RAG does differently

Graph RAG adds a second representation alongside the vectors: an explicit graph of **entities** and the **relationships** between them.

During ingestion, as well as chunking and embedding, the pipeline extracts structure:

```text
Conventional
    document ──> chunks ──> vectors ──> index

Graph-based
    document ──> chunks ──> vectors ──> index
             └─> entities and relationships ──> graph
                    │
                    └─ (Pricing Service) ─[depends on]→ (Rate Store)
                       (Checkout)        ─[calls]→      (Pricing Service)
                       (Checkout)        ─[owned by]→   (Payments team)
```

Retrieval then works in two stages. Find the relevant entities — often using ordinary vector search to get a foothold — then **traverse** the graph outward from them to collect connected context. What reaches the model is not a flat list of similar passages but a connected neighbourhood: these things, and how they relate.

For *"if pricing goes down, what breaks?"*, the traversal walks inbound dependency edges from the pricing service and returns the affected set. That set was never written down in any single document. The graph made it derivable.

### Where the graph comes from

Three broad options, and the difference matters more than people expect.

**Already structured.** Service catalogues, CMDBs, org charts, identity systems, bills of materials. The relationships are authoritative because a system of record maintains them. This is the best case by a wide margin — you are exposing existing structure, not inventing it.

**Extracted by a model.** Run an LLM over the text to pull out entities and relationships. Flexible, and applicable to unstructured corpora. Also fallible: it hallucinates edges, misses others, and resolves the same entity inconsistently across documents. Extraction quality is now a thing you must evaluate and maintain, indefinitely.

**Curated by hand.** Accurate, and it does not scale past a few thousand entities.

Most real deployments combine them: authoritative structure as the backbone, extraction to enrich, human curation for the parts that must be exactly right.

---

## 3. Honest comparison

| | Conventional RAG | Graph RAG |
| --- | --- | --- |
| **Retrieval unit** | Independent chunks | Connected subgraph plus supporting text |
| **Good at** | "What does the document say about X?" | "How does X relate to Y?", multi-hop questions |
| **Bad at** | Anything requiring traversal | Simple lookups — adds cost and latency for nothing |
| **Ingestion cost** | Chunk and embed | Chunk, embed, extract entities, resolve them, build edges |
| **Ongoing burden** | Keep the index fresh | Keep the index *and the graph* fresh and consistent |
| **Main failure mode** | Retrieves similar but irrelevant text | Wrong or missing edges silently produce confident wrong answers |
| **Explainability** | Here are the passages | Here is the path — usually more convincing to an auditor |
| **Team skills needed** | Retrieval tuning | Retrieval tuning *plus* graph modelling and entity resolution |

Two things in that table deserve emphasis.

**The ongoing burden is the real cost.** Standing up a graph is a project. Keeping it accurate as the underlying reality changes is a permanent obligation, and a stale graph is worse than no graph — it answers confidently from relationships that no longer hold.

**Entity resolution is the hard part, and it is usually underestimated.** "Pricing Service", "pricing-svc", "PricingAPI" and "The Pricing Platform" may be four names for one thing, or four different things. Get this wrong and the graph fragments into disconnected islands, or worse, silently merges things that should be distinct. Most graph RAG projects that disappoint, disappoint here.

---

## 4. When to choose which

### Conventional RAG is correct when

- Answers are stated in documents; the job is finding the right passage
- The corpus is mostly prose — policies, manuals, wikis, notes, correspondence
- There is no authoritative source of relationships, and inventing one is not justified
- You need it working soon, and you need it cheap to operate

**This covers the clear majority of enterprise use cases.** Start here. Always.

### Graph RAG earns its complexity when

- **Questions are genuinely multi-hop.** Users routinely ask things requiring two or three relationship traversals.
- **Relationships already exist somewhere authoritative.** You are surfacing them, not fabricating them.
- **The graph *is* the value.** Impact analysis, dependency mapping, lineage, entitlement chains, investigations, supply chains.
- **Traceable reasoning is required.** Regulated contexts where "here is the path we followed" is a deliverable.
- **Conventional RAG has been tried and demonstrably fails.** Not "we suspect it will" — it was built and it did.

### A practical middle position

Before committing to a graph, try **metadata-enriched conventional RAG**: attach structured attributes to chunks — owner, system, effective date, document type, classification — and filter on them at query time.

This handles a surprising number of "relationship-ish" questions at a fraction of the cost, because a filter on `system = pricing` answers many questions people assume need traversal. If enriched metadata gets you there, you have avoided a permanent operational commitment.

---

## 5. What this means inside a federated estate

Two consequences follow directly from Chapter 1's encapsulation rule.

**A graph is an internal implementation detail.** A team running graph RAG exposes the same text-in, text-out interface as everyone else. Callers do not know, and must not need to know, that a traversal happened. The graph never crosses the boundary, for the same reason vectors never do.

**Graph RAG is not the default, and cannot be.** It is a Tier 2 choice on the paved road in [Chapter 2](../chapter-2-building-a-rag-the-paved-road/chapter-2-building-a-rag-the-paved-road.md) — available to teams whose use case demands it, provided by the platform as a supported option with the graph store, extraction pipeline and traversal patterns already built, and never something a team assembles alone.

Which leads to the honest summary: **graph RAG solves a real problem that conventional RAG genuinely cannot, and it charges a permanent operational fee for doing so.** Pay it when the questions require traversal. Do not pay it because the architecture diagram looks more sophisticated.

---

[← Back to Chapter 1](chapter-1-why-one-central-rag-fails.md)
