# Chapter 2 — How Do I Actually Build One?

> **The bridge.** Chapter 1 handed every team a new responsibility: you own the RAG over your data. The first question that lands is the practical one — *fine, but how do I build it, and how much of this am I expected to invent?*
>
> **This chapter covers** choosing between a managed service, a platform pipeline and a bespoke build; what each choice commits you to; selecting an embedding model and surviving the day you change it; ingestion quality; and whether you need retrieval at all.

---

## 2.1 The Problem: Ten Teams, Ten Architectures

Tell ten product teams they own their RAG and give them no further guidance, and you will get ten different systems.

One will use a managed search product and be running in a fortnight. One will assemble a framework-based pipeline from a tutorial and spend four months on it. One will run a vector database on a virtual machine that nobody patches. One will discover halfway through that their documents are scanned PDFs and quietly abandon the project. One will build something genuinely excellent that only its author understands, and that author will change jobs.

Every one of these is a reasonable local decision. Collectively they are a disaster, for reasons that have nothing to do with the quality of any individual choice:

- **Security posture varies per team**, so the estate's posture is the worst team's posture.
- **Nothing is transferable.** Ten codebases, ten deployment models, ten sets of tribal knowledge. Any cross-cutting improvement must be implemented ten times.
- **Effort is spent on the wrong things.** The valuable work is curation and domain knowledge. Instead, engineering months go into chunking strategies that were solved problems before the team started.
- **The estate cannot be reasoned about.** Nobody can answer "are our RAG systems safe to connect to a customer-facing assistant?" because the answer differs ten ways.

This is the failure mode Chapter 1 warned about, arriving on schedule. The fix is not to take ownership away. It is to make the expensive decisions once.

---

## 2.2 The Principle: A Paved Road With Marked Exits

Platform engineering has a well-established answer, and it predates anything to do with AI.

> **Provide a default path that is so well-supported it is easier to take than to avoid. Make departing from it possible, deliberate, and still supported.**

The word that matters is **still supported**. A paved road that abandons you the moment your requirements get interesting is not a paved road; it is a fence, and engineers route around fences. The road has to continue past the exit.

This gives three tiers, and the shape holds regardless of technology, cloud, or whether there is a cloud at all.

```text
  TIER 1 — MANAGED SERVICE                       most teams
  ────────────────────────────────────────────
  A product does extraction, indexing, retrieval.
  You build a thin pipeline that feeds it and
  configure how it behaves. Days, not months.
  
                 │ insufficient for a specific, stated reason
                 ▼
  
  TIER 2 — PLATFORM PIPELINE                     a minority
  ────────────────────────────────────────────
  A parameterised pipeline, built and versioned by the
  platform team, configured and run by you, inside your
  own boundary. You get the knobs. You do not get the
  maintenance burden.
  
                 │ genuinely unprecedented requirement
                 ▼
  
  TIER 3 — BESPOKE                               rare, and reviewed
  ────────────────────────────────────────────
  You build it. Under platform guardrails, against the
  same interface contract, with an explicit owner and
  an explicit review date.
```

**Every tier has a pipeline.** This is worth stating plainly, because the opposite is widely assumed and it sets teams up to be surprised. What changes across the tiers is not whether a pipeline exists but how much of it you *author* versus *configure*, and how many of the decisions are permanent. A managed service does not remove the pipeline; it removes the retrieval stack underneath it.

Two rules keep this honest.

**Movement between tiers is justified, not chosen.** Leaving Tier 1 requires naming the specific capability Tier 1 lacks. "We want more control" is not a reason. "Our corpus is 200,000 scanned engineering drawings requiring layout-aware extraction that the managed connector does not support" is.

**The interface contract is identical at every tier.** Whatever tier a team is on, it exposes the same governed text-in, text-out surface from Chapter 1. Consumers cannot tell, and must not need to tell, which tier is behind it. This is what lets a team move tiers later without breaking anybody.

### The serverless-then-orchestrated analogy

Most engineers already have this instinct from compute. Deploy to the serverless platform by default — no cluster to manage, scales to zero, security handled. If you truly need orchestration-level control, you get it.

But here is the part usually stated badly. The exception is **not** "and then the team builds a cluster."

> **The platform team runs the cluster. The team that needs it gets a namespace.**

They get the control they came for — workloads, scheduling, sidecars, custom runtimes — and they do not inherit upgrades, patching, node pools, networking or the on-call rotation that comes with all of it. The escape hatch leads to *a different supported place*, never to a team quietly becoming an infrastructure team.

That same rule governs the tiers here. Tier 2 does not mean "build your own pipeline." It means the platform's pipeline, configured by you, running in your boundary.

> **Reference implementation — public cloud.** The serverless/orchestrated split maps to Cloud Run and GKE on Google Cloud, Lambda or App Runner and EKS on AWS, Container Apps and AKS on Azure. On-premises it maps to a shared internal platform versus dedicated namespaces on a central cluster. The naming changes; the principle does not.

---

## 2.3 Tier 1: Managed Search

Every major cloud offers a service that takes the entire RAG pipeline as a managed product: connect sources, and it handles crawling, extraction, chunking, embedding, indexing, retrieval and permission filtering.

**This should be the default for most teams, and the bar for leaving it should be high.** The reason is not that these services are technically remarkable. It is that a team on Tier 1 spends its time on the thing only that team can do — deciding what belongs in the corpus and whether the answers are right — instead of on a pipeline that a hundred other organisations have already built identically.

What you get without building it:

**Connectors to where documents actually live.** Collaboration platforms, wikis, object storage, ticketing, CRM. Including — crucially — incremental change detection, so edits and deletions propagate without a full re-crawl.

**Extraction that handles real documents.** Scanned PDFs, which are images rather than text and need optical character recognition (OCR) before there is anything to retrieve at all. Tables that lose all meaning if flattened to a line of text. Multi-column layouts. Presentations. This is dull, high-value engineering, and it is the single most common reason a hand-built pipeline disappoints: the retrieval was fine, the extraction was silently mangling the source.

**Permission-aware retrieval.** This one deserves more attention than it usually gets, because it is the strongest single argument for the paved road and it is routinely misunderstood.

A common assumption is that once a document has been embedded it becomes "just maths" — a point in a vector space with no memory of who was allowed to read the original. For a raw vector store, that assumption is correct. A vector index has no concept of a reader. If you build one, you are also building an authorisation engine, and you will be maintaining it for as long as the system lives.

A managed search service does not work that way. It carries the source system's reader principals into the index as first-class metadata alongside the vector, and applies them as a filter at query time against the identity of the person actually asking. You are not rebuilding the source's access model. You are propagating it. When someone loses access to a document in the source system, they stop seeing it in answers, without anyone writing code to make that happen.

This does not replace the access control at the interface boundary, which Chapter 3 covers — that is a separate layer, and both are needed. But it is the difference between filtering by entitlement and reimplementing entitlement.

**Operational behaviour you did not have to think about.** Scaling, availability, index maintenance, encryption, region placement.

### What you still build

The pipeline does not disappear at Tier 1. It gets thin. What remains yours:

- **Source enumeration** — deciding what is in scope, and keeping that decision current as the source grows
- **Upload or connector scheduling** — something has to run, on a schedule or on an event, and something has to notice when it fails
- **Metadata and permission emission** — attaching the reader principals, classification and provenance that the service will index alongside the content
- **Parser and chunking configuration, expressed as code** — so the configuration is reviewable and reproducible rather than a set of choices somebody once made in a console
- **Ingestion monitoring** — documents skipped, documents failed, and whether that number is normal
- **Deletion propagation** — the hardest part of any incremental sync, and not something a connector exempts you from thinking about

That is a real pipeline. It is simply a much smaller one than the alternatives, and the platform team can supply most of it as a template.

### What you decide, and what you cannot undo

Managed does not mean opaque. The meaningful knobs remain yours:

- Which sources are connected, and which subsets within them
- How content is parsed — plain text extraction, optical character recognition for scanned material, or layout-aware parsing that preserves tables, headings and lists
- How content is chunked, and whether a chunk carries the headings above it
- Which parts of a web page are excluded as navigation or boilerplate
- How identity and permissions map from source to index
- Refresh cadence per source
- Relevance tuning — boosting, filtering, synonyms, structured facets
- Which fields are filterable, and which appear in citations
- Where data resides, and which encryption keys protect it

> **The decisions that are permanent.** Some of these are one-way doors, chosen at creation time and not changeable afterwards. Whether chunking is enabled at all, how chunks are sized, and whether the data store enforces per-document access control are typically fixed when the store is created — changing your mind means building a new one and reindexing everything.
>
> Changing the parser is only half-reversible: new settings apply to documents ingested from that point, and material already indexed is not reprocessed unless you reprocess it deliberately.
>
> **This is why Tier 1 needs more judgement than its reputation suggests, not less.** The configuration is small, but several of the choices are permanent, and they are made on day one when the team knows least about its own corpus. The platform team's job here is not to run the pipeline. It is to make sure the team understands which decisions they are about to make forever.

### What to check before committing

- **Do connectors exist for your actual sources,** including permission propagation, not merely content?
- **Does extraction handle your document types?** Test with your worst documents, not your cleanest.
- **Is per-document permission filtering real,** and at what latency?
- **What are the ceilings** on corpus size, document size, query rate?
- **Can you export what you put in,** if you later need to move?

> **Reference implementation.** Agent Search on Google Cloud — the service formerly named Vertex AI Search, and before that several other things, which is itself a reason to write against the capability rather than the brand. Amazon Kendra or Bedrock Knowledge Bases on AWS; Azure AI Search with integrated vectorisation on Azure; Elastic or OpenSearch-based stacks on-premises. A deeper treatment of capabilities and limits is in [Managed Search in Depth](managed-search-in-depth.md), and the configuration surface in [Tier 1 — Managed](pipeline-tier-1-managed.md).

---

## 2.4 Tier 2: The Platform Pipeline

Some requirements genuinely exceed a managed product. The usual ones:

- **Relationship-heavy questions** needing traversal, not similarity — see [Graph RAG](../chapter-1-why-one-central-rag-fails/graph-rag-explained.md)
- **Structure-sensitive corpora** — source code, deeply nested specifications, documents where parent-child context must survive chunking
- **Mixed retrieval** — semantic search combined with live structured queries in one answer
- **Sovereignty or isolation constraints** that rule out the managed option
- **Retrieval logic specific to the domain** — custom ranking incorporating recency, authority, or regulatory status

For these, the answer is emphatically **not** "go and build it."

> **The platform team builds one pipeline, parameterised deeply enough to serve these cases. Teams configure it and run it in their own boundary. The platform owns the code and its lifecycle.**

Think of it as a library rather than a service. The team's deployment, the team's data, the team's boundary — but not the team's code to maintain.

This works only if the pipeline is genuinely good. A thin, under-parameterised platform pipeline is worse than useless: teams will hit its limits, conclude the platform does not understand their problem, and leave for Tier 3. **The cases arriving at Tier 2 are few but hard, and they are precisely the cases the organisation cares most about.**

What "deeply parameterised" has to mean in practice — chunking strategies, embedding model selection, retrieval modes, reranking, metadata schemas, refresh behaviour — plus versioning and upgrade policy, is specified in [Tier 2 — Assembled](pipeline-tier-2-assembled.md).

> **Reference implementation.** A managed RAG framework sits at this tier — one that exposes ingestion, transformation, embedding, indexing and retrieval as named stages you configure rather than write. Which products qualify, and the isolation-versus-residency trap that decides whether any of them is usable for a given corpus, is set out in [Tier 2 — Assembled](pipeline-tier-2-assembled.md).

### The division of responsibility

| Platform owns | Team owns |
| --- | --- |
| Pipeline code, versioning, upgrade path | Configuration values |
| Infrastructure modules and landing zone | Which sources are connected |
| Default configurations that work | Domain-specific tuning |
| Security baseline and interface contract | Corpus content and its quality |
| Support, documentation, escalation | Running it, monitoring it, and the answers it gives |

---

## 2.5 Choosing the Embedding Model — and Living With the Choice

Chapter 1 established that teams are free to choose an embedding model, because vector spaces never meet. Freedom to choose is not freedom to drift.

**The problem.** A team picks a model, indexes a large corpus, and the decision sets. Two years later the model is deprecated, or a materially better one exists, or the provider changes terms. Re-embedding is not a config change — it is a full reprocessing of the entire corpus, and done naively it means downtime or a period of serving from a half-rebuilt index.

**The principle.** Model choice is reversible, and reversibility is engineered in advance rather than discovered in an emergency.

**What the platform provides:**

**An approved model list**, with dimensions, cost, latency, language coverage, and — importantly — deployment eligibility. A model available in a public region may be unusable in a sovereign or disconnected one. The list exists to accelerate the decision, not to restrict it to one option.

**Version pinning.** A team's index records exactly which model and version produced it. Silent provider-side upgrades are exactly how an index degrades with no corresponding change on your side.

**A re-embedding runbook, with dual-index cutover as the default pattern:**

```text
  1. Build the new index alongside the live one, new model
  2. Run both against the evaluation set (Chapter 4)
  3. Compare — a new model is not automatically better for your corpus
  4. Shift traffic when the new index demonstrably wins
  5. Retain the old index until confidence holds, then retire
```

Never rebuild in place. The dual-index pattern is what turns a migration from an outage into a routine operation.

**Re-embedding as a budgeted, scheduled event.** Reprocessing a corpus costs real money and real time. Teams should expect to do it every year or two and plan for it, rather than treating it as an unexpected crisis. The cost belongs in the cost model in [Chapter 5](../chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md).

---

## 2.6 Ingestion Quality: Where Answers Quietly Go Wrong

More RAG deployments are undone by ingestion than by retrieval, and the failures are hard to see because they do not raise errors.

**The failure modes:**

- A table becomes a single line of run-together text. Every number in it is now meaningless, and it retrieves for queries about those numbers.
- A scanned document yields no text. It is simply absent from answers, and nothing reports this.
- A chunk boundary splits a rule from its exception. Retrieval returns the rule. The answer is confidently wrong in exactly the situation the exception covers.
- A template's boilerplate header dominates the chunk's embedding, so every document of that type looks identical to the index.
- A navigation sidebar is captured as content and retrieves for everything.

None of these throw an exception. They produce a system that is subtly, unpredictably wrong — the hardest kind of problem to debug from the answer end.

**The principle: validate at ingestion, because the alternative is discovering it from a bad answer in front of a customer.**

**What the platform provides:**

**Format-aware extraction as a shared service** — OCR, table structure preservation, layout handling, header and footer suppression. Built once, for everyone.

**Quality gates that reject rather than ingest.** Chunks that are empty, near-duplicate, mostly boilerplate, or statistically anomalous against the corpus baseline are flagged, not indexed. **Refusing to ingest a bad chunk is cheaper than explaining a bad answer.**

**An ingestion report per run** — documents processed, skipped and why, chunks produced, distribution, warnings. Visible to the team that owns the corpus, because that team is the only one who can tell whether "142 documents skipped" is fine or alarming.

**Provenance metadata on every chunk** — source, document identifier, location within the document, extraction method and version, timestamp. This is what makes citation, deletion propagation ([Chapter 6](../chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md)) and retirement ([Chapter 7](../chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md)) possible at all.

---

## 2.7 Do You Even Need Retrieval?

Context windows have grown enormous. A reasonable question follows: why retrieve at all, rather than put the documents in the prompt?

The common answer — "use RAG when you have millions of documents" — is wrong, or at least incomplete enough to mislead. **Corpus size is not the deciding factor.** A team with four hundred documents often still wants retrieval, and a team with a hundred thousand sometimes does not.

The real axes are these.

### Retrieval is the right choice when

**The knowledge must be reusable.** This is the one most often missed. A RAG is a *governed asset* — discoverable, access-controlled, attributable, citable, with an owner. Documents stuffed into one application's prompt are none of those things. They are invisible to the rest of the organisation. Three other teams will independently do the same thing with the same documents, and nobody will know.

**Citations are required.** Retrieval knows which passages it used. Long-context generation can cite, but the link between claim and source is weaker and harder to verify. In a regulated setting this distinction is not academic.

**Access control is per-document.** Retrieval filters by entitlement before assembling context. A stuffed prompt is all-or-nothing — if any part is restricted, the whole thing is.

**Deletion must be enforceable.** Removing a document from an index is an operation you can perform and evidence. Removing it from every prompt template and cache is not.

**Cost and latency must be predictable.** Retrieval sends a bounded amount of context per request. Long context sends everything, every time, and you pay for all of it on every call. Context caching changes the arithmetic but does not reverse it — a cached corpus is cheaper to resend than an uncached one, and still more expensive than not sending it. Model it against your actual query volume rather than against the per-token headline, because the two diverge sharply once traffic is real.

**Content changes independently of code.** Updating a corpus is a data operation. Updating an embedded prompt is a deployment.

### Long context is the right choice when

**The task needs the whole document at once.** Reviewing a contract for internal inconsistency, or an entire codebase for a cross-cutting change. Retrieval fragments exactly what the task requires be seen whole.

**The content is ephemeral.** A file the user just uploaded for this one conversation. Indexing it would be overhead with no reuse.

**The corpus is small, stable, single-purpose and single-tenant.** One team, one application, no reuse, no per-document entitlement.

**You are prototyping.** Long context first is a legitimate way to establish whether the task works at all before building infrastructure.

### And frequently: both

A mature pattern is retrieval to *select* which documents are relevant, then supplying those documents in full to a long-context model. Precision of retrieval, comprehension of long context. The two are not competitors.

> The short version: **retrieval is not an optimisation for large corpora. It is how knowledge becomes a governed, shared, auditable asset rather than a private prompt.** Choose it when those properties matter — which, in an enterprise, is most of the time.

---

## 2.8 What the Platform Team Provides

| Platform provides | Team owns |
| --- | --- |
| Tier 1 onboarding: account, connectors, configuration templates | Which sources, which subsets, what belongs |
| Tier 2 pipeline: versioned, parameterised, supported | Configuration values and domain tuning |
| Tier 3 guardrails, review process, interface conformance | Justifying the exception; building within the rails |
| Infrastructure modules; namespaces on platform-run clusters | Deploying into their own boundary |
| Approved embedding models, pinning, re-embedding runbook | Choosing a model; scheduling migrations |
| Extraction, quality gates, ingestion reporting | Acting on the report; corpus quality |
| The RAG-vs-long-context decision framework | Applying it to their use case |
| **The knowledge to choose well** — which tier suits which corpus, what each commits you to, what typically goes wrong and its earliest symptom | Making the choice, and living with it |

That last row is easy to skip past and is arguably the most valuable thing on the list. A platform team that ships a pipeline and no judgement produces teams that adopt the pipeline and misuse it. The decision support *is* part of the product, and the three tier documents below exist to carry it.

---

## Where this leaves you

A team can now get a RAG built without inventing a pipeline, and the organisation gets consistency without taking ownership away. Tier 1 for most, Tier 2 for the hard minority, Tier 3 rarely and deliberately — all behind one interface contract.

But note what that contract has been so far: an assertion. Chapter 1 said teams expose a governed text-in, text-out surface and nothing else. Chapter 2 has kept saying it. **Neither has specified what it is.**

A working RAG that anyone can call is not an asset — it is an incident waiting to be written up. The question of *how* it is exposed, to whom, under whose identity, and with what checked before a single document is retrieved, is where a federated estate is either secured or lost.

---

**Supporting material for this chapter**

The three tier documents share an identical structure, so they can be read side by side and compared section for section.

- [Tier 1 — Managed](pipeline-tier-1-managed.md) — the thin pipeline, what you configure, and the decisions you cannot undo
- [Tier 2 — Assembled](pipeline-tier-2-assembled.md) — the platform pipeline's parameters, configuration model and lifecycle
- [Tier 3 — Bespoke](pipeline-tier-3-bespoke.md) — what you take on when you own the index yourself
- [Managed Search in Depth](managed-search-in-depth.md) — what Tier 1 really gives you, its limits, and how to evaluate it

---

[← Chapter 1 — One RAG for the Whole Enterprise, and Why It Breaks](../chapter-1-why-one-central-rag-fails/chapter-1-why-one-central-rag-fails.md) | [Chapter 3 — How Do I Expose It Without Losing Control? →](../chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md)
