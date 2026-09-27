# Tier 1 — Managed

> Supporting material for [Chapter 2 — Building a RAG: The Paved Road](chapter-2-building-a-rag-the-paved-road.md)
>
> One of three tier documents. [Tier 1 — Managed](pipeline-tier-1-managed.md) · [Tier 2 — Assembled](pipeline-tier-2-assembled.md) · [Tier 3 — Bespoke](pipeline-tier-3-bespoke.md). All three follow the same eight sections so they can be read side by side.

---

## 1. When this tier is the right choice

A managed search service does extraction, chunking, embedding, indexing, hybrid retrieval, ranking and permission filtering as a single configured product. You bring content and configuration; it brings the retrieval stack.

**Choose it when:**

- Your content is documents — reports, policies, manuals, wikis, tickets, contracts
- Your sources have connectors, or can be pushed to a location that does
- Your questions are answerable from passages, not from relationships between records
- Your permission model lives in the source systems and should keep living there
- You want to be answering questions this month rather than building infrastructure this quarter

**It is disqualified when** — and these are the only reasons that count:

- The question is a graph question, not a similarity question
- Meaning depends on structure the parser cannot see from the page
- You need transactional consistency between unstructured search and live operational data
- Relevance depends on domain logic beyond the tuning surface — supersession chains, regulatory status, authority hierarchies
- The service cannot be deployed where your data must live
- A genuine published limit — corpus size, document size, query rate — is below your actual numbers

Note what is not on that list: wanting control, avoiding lock-in, sensitivity of data, and preferring a particular embedding model. Those are addressed in [Managed Search in Depth](managed-search-in-depth.md), which also gives an evaluation protocol for testing the disqualifiers against your own corpus rather than assuming them.

**The default assumption is that this tier is sufficient.** The burden of proof sits with the team that wants to leave it, and the proof is an evaluation result, not a preference.

---

## 2. What you configure

The surface is small. That is the point. It is not, however, trivial.

**Sources.** Which systems, and which subsets within them. Include and exclude patterns. Whether a connector pulls on a schedule or you push on an event.

**Parsing.** Plain text extraction, optical character recognition for scanned material, or layout-aware parsing that detects headings, tables and lists. Which page regions are suppressed as navigation or boilerplate.

**Chunking.** Whether content is chunked at all, how large the chunks are, and whether each chunk carries the headings above it.

**Identity and permissions.** How reader principals map from the source system into the index, and which identity provider the query-time check resolves against.

**Refresh.** Cadence per source, and whether refresh is incremental or full.

**Relevance.** Boosting, filtering, synonyms, structured facets, and which fields are filterable, sortable or returned in citations.

**Placement.** Which region holds the data, and which encryption keys protect it.

**Hold all of this in version control as declarative configuration, not as console clicks.** This is the single practice that separates a Tier 1 deployment you can reproduce, promote between environments and recover from a disaster, from one that exists only as a series of choices somebody made on a Thursday.

---

## 3. What you still build yourself

The pipeline gets thin here. It does not disappear, and teams that assume it does are surprised in the second month.

- **Source enumeration and scope maintenance** — deciding what is in, and revisiting that as the source grows
- **Scheduling** — something must run, and something must notice when it has not
- **Metadata and permission emission** — attaching reader principals, classification, effective dates and provenance for the service to index alongside the content
- **Parser and chunking configuration as code** — see above
- **Ingestion monitoring** — documents skipped, documents failed, and whether today's number is normal
- **Deletion propagation** — the hardest part of any incremental sync, and not something a connector exempts you from

The platform team can supply most of this as a template. It still has an owner, and the owner is you.

---

## 4. Decisions you cannot undo

This is the section people skip and then need.

**Fixed at creation, changeable only by rebuilding and reindexing:**

- Whether chunking is enabled
- How chunks are sized
- Whether the data store enforces per-document access control

**Half-reversible.** Changing the parser applies to documents ingested from that point onwards. Material already in the index is not reprocessed unless you deliberately reprocess it. An estate that changed its parser six months ago and never reindexed is running two parsers at once and does not know it.

**Effectively permanent by inertia.** The identity mapping, once consumers depend on it, and the region, once data volumes make a migration a project rather than a task.

> **The uncomfortable part.** These choices are made on day one, when the team knows least about its own corpus. The platform team's job at this tier is not to run the pipeline. **It is to make sure the team understands which decisions they are about to make forever.** A thirty-minute conversation before a data store is created is worth more than any amount of support afterwards.

---

## 5. What typically goes wrong, and the early symptom

| Symptom | What it means | Response |
| --- | --- | --- |
| Answers are confidently wrong about a document type | Extraction failed on that format and nobody read the output | Read the extracted text for your ten worst documents, before blaming retrieval |
| Retrieval quality good in testing, poor in production | The test corpus was the clean corpus | Re-evaluate against a realistic sample including scans and odd formats |
| A user sees something they should not | Permission mapping is stale, or the store was created without per-document access control | Check which; the second one is a rebuild |
| Skip rate climbing quietly | A source changed format and the parser is failing silently | Alert on skip rate as a ratio, not as an absolute |
| Deleted documents still being cited | Deletion is not propagating from source to index | This is a design gap, not a bug; see Chapter 7 |
| Team asking to move to Tier 2 after three months | Usually extraction or chunking, occasionally a real disqualifier | Establish which before agreeing; most such requests are solvable inside Tier 1 |

**The pattern across all of these:** at Tier 1 the failures are almost never in retrieval. They are in what went into the index, and they are invisible unless someone is looking at the ingestion report.

---

## 6. What the platform provides, what the team owns

| Platform provides | Team owns |
| --- | --- |
| The account, the entitlement, the onboarding path | Deciding to use it |
| Configuration templates as code, per source type | Configuration values |
| Connector setup, including permission propagation | Which sources, which subsets |
| The conversation about irreversible decisions | Making those decisions |
| The evaluation protocol and a query-set template | Running it, on their own corpus |
| Ingestion reporting and skip-rate alerting | Acting on it |
| The governed interface in front of the service | Keeping the contract |
| Cost attribution labelling | Their own spend |

---

## 7. Reference implementation

> **Reference implementation.** Agent Search on Google Cloud — previously named Vertex AI Search, and several other things before that, which is itself an argument for writing against the capability rather than the brand name. It provides connectors with access-control propagation, layout-aware parsing, configurable chunking, hybrid retrieval with reranking, and per-document permission filtering resolved against the caller's identity at query time.
>
> The equivalents elsewhere: Amazon Kendra or Bedrock Knowledge Bases on AWS; Azure AI Search with integrated vectorisation on Azure; an Elastic or OpenSearch-based stack self-operated on-premises.
>
> Treat every specific in this block as an example to verify rather than a specification to build against. Names, limits and feature coverage in this category change faster than any document can track.

---

## 8. What you are signing up to operate

Modest, and non-zero.

**Ongoing:** the ingestion schedule and its failures; the scope of each source as it grows; the ingestion report; the evaluation query set, refreshed as the corpus changes; the configuration repository.

**Periodically:** reindexing when the parser changes; re-running the evaluation after any configuration change; reviewing whether the disqualifiers have become true as the use case has grown.

**Not yours:** scaling, availability, index maintenance, the embedding model's lifecycle, the retrieval algorithm, security patching of any of it.

**Staffing reality:** a fraction of one engineer, steady state. That is the number this tier exists to produce, and it is why it should be the default.

---

[← Managed Search in Depth](managed-search-in-depth.md) | [Tier 2 — Assembled →](pipeline-tier-2-assembled.md)
