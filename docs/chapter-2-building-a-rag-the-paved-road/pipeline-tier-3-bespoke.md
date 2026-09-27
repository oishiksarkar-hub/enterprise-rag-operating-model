# Tier 3 — Bespoke

> Supporting material for [Chapter 2 — Building a RAG: The Paved Road](chapter-2-building-a-rag-the-paved-road.md)
>
> One of three tier documents. [Tier 1 — Managed](pipeline-tier-1-managed.md) · [Tier 2 — Assembled](pipeline-tier-2-assembled.md) · [Tier 3 — Bespoke](pipeline-tier-3-bespoke.md). All three follow the same eight sections so they can be read side by side.

---

## 1. When this tier is the right choice

You build the retrieval system yourself: your own index, your own pipeline, your own operational responsibility. The platform provides guardrails, the interface contract and the telemetry schema, and nothing else.

**Choose it when the requirement is not expressible as configuration of anything that exists.** In practice that is a short list:

- **A different retrieval paradigm.** Graph traversal as the primary access path, not as an addition. Multi-modal retrieval over content no product handles. Retrieval over a data structure that is not documents.
- **A genuinely novel ranking objective** — one that depends on computation, not on boosting a field.
- **A hard external constraint** that excludes every managed and platform option, with the exclusion demonstrated rather than assumed.
- **Research.** The work is to find out whether something is possible. This is legitimate, and it should be labelled as research rather than as a production tier.

**It is disqualified when** — and these account for most requests that reach it:

- Tier 2 lacks a parameter. That is a gap to close, not a reason to leave.
- The team prefers a particular framework, library or vector database.
- The team believes it will be cheaper. Model it including salaries, on-call and the cost of the eventual migration, and it usually is not.
- The team has not evaluated Tier 1 or Tier 2 against its own corpus. An untested assumption is not a disqualifier.

> **This tier should be rare, and its population is a diagnostic.** A growing Tier 3 does not mean teams have unusual requirements. It means the paved road is not credible, and the fix is upstream in Tier 2 rather than downstream in governance.

Every Tier 3 system requires a written justification naming the specific capability that is missing, a named owner, and an explicit review date at which the justification is re-tested against what the lower tiers can do by then.

---

## 2. What you configure

Nothing, in the sense the other two tiers mean it. There is no configuration surface because there is no product underneath — there is code you wrote.

What you *accept* rather than configure is the guardrail set, which is not negotiable:

**The interface contract.** The same governed surface, the same identity model, the same versioning discipline as every other tier. Consumers must not be able to tell which tier they are calling. This is what preserves the option of migrating back later.

**The telemetry schema.** The span names, the required attributes, the provenance fields, the cost labels. You emit them yourself, and they must match — a Tier 3 system that traces differently is invisible in every estate-wide view.

**Authorisation before retrieval.** Entitlement resolved against the calling principal, at tool granularity, per request. You are building this yourself, which is precisely the cost being incurred.

**Approved components.** Embedding models from the approved set, pinned. Infrastructure from the platform's modules. Deployment inside your own boundary.

**Conformance testing.** The platform's contract suite runs against your implementation, in your pipeline, and a failure blocks release.

---

## 3. What you still build yourself

Everything. It is worth writing the list down before committing, because it is longer than it feels at the whiteboard.

- Source connectors, and permission extraction from each source
- Extraction across every document format you hold, including the ugly ones
- Chunking, and the strategy decisions behind it
- Embedding orchestration, batching, retry and backfill
- The index itself: schema, build, optimisation, compaction, replication
- Hybrid retrieval, if you want it — dense and sparse, and the fusion between them
- Reranking
- Query-time entitlement filtering
- Incremental refresh, and deletion propagation
- Quality gates and the ingestion report
- Provenance capture on every chunk
- Trace emission across every stage
- Cost labelling on every resource and every model call
- The governed interface in front of all of it
- The evaluation harness
- Disaster recovery, and a rebuild procedure you have actually tested

The platform can hand you modules for some of these. The integration, the correctness and the operation are yours.

---

## 4. Decisions you cannot undo

At this tier almost everything is reversible in principle and expensive in practice, which is a worse position than a clear one-way door.

**Reindex-class, as at Tier 2:** embedding model, dimensions, chunking strategy, preserved fields.

**Architecture-class, and this is the difference:** the index technology itself. A vector database chosen in month one shapes your operational model, your scaling story, your backup strategy, your hiring and your cost curve. Changing it later is not a migration; it is a rebuild.

**Organisation-class, and the most underestimated:** the decision creates a permanent team. Someone owns this system for as long as it exists. The team that built it will not be the team that operates it in three years, and the handover of a bespoke retrieval system is considerably harder than the handover of a configuration file.

**The irreversible one nobody writes down:** the opportunity cost. Every capability the platform ships to Tier 1 and Tier 2 over the next two years — better parsing, better ranking, a better permission model — arrives for those tiers and not for yours. You have opted out of the improvements as well as the constraints.

---

## 5. What typically goes wrong, and the early symptom

| Symptom | What it means | Response |
| --- | --- | --- |
| The justification cites a preference, not a capability | The exception should not have been granted | Send it back; ask which parameter is missing from Tier 2 |
| Six months in, still no evaluation set | The team is optimising by intuition | Stop feature work until there is a baseline |
| Telemetry does not match the schema | The system is invisible in estate-wide views | Conformance testing should have caught this at release |
| The original builders have moved on | The permanent staffing cost has arrived | This was predictable; plan for it at the review date |
| Retrieval quality below Tier 1 on the same corpus | The bespoke build is losing to the product it replaced | Re-test the original justification honestly |
| The review date passes unnoticed | The exception has become the default | The review date needs an owner, not a calendar entry |
| Migration back is "too hard" | The interface contract was not held | The contract is what makes return possible; enforce it at release |

**The most common failure is none of these.** It is that the system works, is never bad enough to replace, is never good enough to be a model for anyone else, and quietly consumes a headcount forever.

---

## 6. What the platform provides, what the team owns

| Platform provides | Team owns |
| --- | --- |
| The exception process and the written justification template | Making the case, specifically |
| The interface contract and its conformance suite | Passing it, at every release |
| The telemetry schema and provenance specification | Emitting it correctly |
| Infrastructure modules and approved model access | Everything built on them |
| The review process and its date | Being ready for it |
| Advice, on request | The system, its quality, its cost and its operation |

The asymmetry in that table is the point. It is not a punishment — it is an accurate description of what choosing this tier means.

---

## 7. Reference implementation

> **Reference implementation.** Where a bespoke build is justified, the usual components are a managed vector search service used as a primitive rather than as a product — Vertex AI Vector Search on Google Cloud, for example — or a database with vector support such as AlloyDB with its ScaNN index, PostgreSQL with `pgvector`, or a dedicated vector database self-operated on the container platform.
>
> Note what this list is: storage and nearest-neighbour search. Extraction, chunking, permission filtering, ranking, refresh and provenance are not included in any of them. That gap is the whole of section 3 above, and it is the honest measure of what this tier costs.
>
> Treat every specific here as an example to verify rather than a specification to build against.

---

## 8. What you are signing up to operate

All of it, permanently.

**Ongoing:** the pipeline and its failures; the index and its maintenance; the retrieval path and its latency; the entitlement filter and its correctness; the evaluation harness; the telemetry; the cost; the on-call rotation.

**Periodically:** reindexing on every model or chunking change; upgrading every dependency you chose; re-testing the justification at the review date; rehearsing the rebuild procedure.

**Permanently:** a named owner. When that person leaves, a named successor. This is the commitment that outlasts every technical decision in this document, and it is the one that should be hardest to obtain approval for.

**Staffing reality:** one to two engineers, indefinitely, for the retrieval system alone — before anyone works on the domain problem the system was built to serve. That number is the argument for the paved road, and it is why this tier is the last resort rather than the interesting one.

---

[← Tier 2 — Assembled](pipeline-tier-2-assembled.md) | [Back to Chapter 2 →](chapter-2-building-a-rag-the-paved-road.md)
