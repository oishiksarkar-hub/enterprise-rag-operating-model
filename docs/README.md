# Enterprise RAG: An Operating Model

**Your organisation has decades of knowledge scattered across hundreds of systems. The obvious move is to build one RAG over all of it. That will not work — and understanding exactly why is the beginning of an architecture.**

This is a book about what happens after you accept that retrieval has to be federated: the problems that decision creates, and how a platform team makes them survivable.

It is vendor-agnostic by design. Principles first, patterns second, and a worked example third — in clearly marked blocks you can skip if you are on a different cloud, or on none at all.

---

## The problem with one central RAG

A single enterprise-wide RAG fails for reasons that are structural, not technical. Better engineering does not fix them.

**Embedding spaces cannot be shared.** Two teams choosing different embedding models produce vectors that are mathematically incomparable. One index means one model for everyone, forever — and the decision is effectively irreversible once a large corpus is built on it.

**Access control cannot be centralised.** Entitlement rules live with the systems that produced the data, they change constantly, and they are understood only by the teams that own them. A central index must either replicate all of it — permanently, accurately, for every source — or leak.

**Cross-domain noise produces confident nonsense.** A corpus spanning every domain retrieves passages from unrelated ones, and a model asked to synthesise them will do so fluently. The wider the corpus, the worse the precision.

**Ingestion contends.** One pipeline, one queue, one team's urgent backfill blocking everyone else's refresh.

**Nobody owns the content.** A central platform team owns the infrastructure and cannot possibly own the meaning, currency or accuracy of a hundred domains' knowledge. So nobody does, and it decays.

---

## The answer, and what it costs

> **RAG ownership follows data ownership. The team that produces and maintains the data owns the RAG over it — and exposes it through a governed interface that returns text and citations, never raw vectors and never the underlying store.**

Each failure above dissolves:

| Failure | How federating resolves it |
| --- | --- |
| Embedding collision | Each team chooses its own model; spaces never meet, because they never share an index |
| Access control | Entitlement is enforced by the team that already understands it, at their own boundary |
| Cross-domain noise | A scoped corpus retrieves from its own domain, and precision rises |
| Ingestion contention | Independent pipelines, independent schedules, no shared queue |
| Unowned content | The team that authored it is accountable for it |

And then the bill arrives. **Every hard problem you solved once is now a problem every team faces.** Pipelines, infrastructure, identity, authorisation, tracing, evaluation, cost attribution, freshness, incident response — multiplied by the number of teams.

Solved naively, that is worse than the monolith.

---

## The resolution: decentralised execution, centralised principle

The way out is not to take ownership back. It is to make every expensive decision once.

**Centralised:** the interface contract, the identity and authorisation model, the tracing standard, the evaluation harness, the cost ledger, the security baseline, the pipeline itself, the registry.

**Decentralised:** what is in the corpus, what it means, what good looks like, when it refreshes, and who is accountable when it is wrong.

The team gets a versioned, parameterised, supported capability — closer to a library they adopt than a service they are subject to. They configure and run it inside their own boundary, on their own data.

**That is what the platform team is for, and it is the argument this book makes chapter by chapter.** There is no separate "platform" chapter. Every chapter ends with what the platform provides and what the team still owns, because that is how the division actually works.

---

## Assumptions

- **Vendor and tool agnostic.** Principles hold across providers. Where a concrete example helps, one cloud is used as a worked illustration in a marked block.
- **Cloud, private cloud, on-premises and air-gapped are all in scope.** [Chapter 9](chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md) covers what changes without a public cloud — which is less than most people assume.
- **MCP and A2A are assumed as the interface layer.** They are protocols, and they work identically over an internal network.
- **A platform team exists, or should.** In a constrained environment it is not optional; it is the only way the capability exists at all.
- **You are not starting from a blank page.** [Chapter 11](chapter-11-measuring-whether-it-works/chapter-11-measuring-whether-it-works.md) covers migrating from a central RAG that already exists.

---

## How to read the specifics

Four conventions, stated once so they need not be repeated on every page.

**Marked reference blocks are examples, not specifications.** Where a block names a product, a service or a capability, it is there to make an abstract argument concrete. Product names change, limits move, features appear and are withdrawn. **Verify every specific against the current vendor documentation before you build on it.** The principle around the block is what is intended to last; the block itself is a snapshot.

**Code and configuration come in two kinds.** Some is illustrative — a shape that shows what a parameter surface looks like, not a schema to implement literally. Some is a contract — a span name, an attribute, a claim, a field that has to match for the estate to work as a whole. Where it is a contract, the surrounding text says so explicitly. Where it does not, assume illustrative.

**Numbers are principles with an example attached.** Where a figure appears — a window, a threshold, a ratio, a period — the sentence around it gives the rule that produced the number. Take the rule and derive your own figure. An organisation's deprecation window, cache lifetime or refresh cadence depends on facts about that organisation, and a borrowed number is a guess wearing a uniform.

**Every chapter separates what the platform provides from what the team owns.** That table is not a summary; it is the operating model in its most compressed form, and it is where most of the argument actually lands.

---

## The map

Each chapter is one problem, with its own solution, and can be read on its own.

| | Chapter | The question it answers |
| --- | --- | --- |
| 1 | [Why One Central RAG Fails](chapter-1-why-one-central-rag-fails/chapter-1-why-one-central-rag-fails.md) | Why is the obvious architecture wrong, and what replaces it? |
| 2 | [Building a RAG: The Paved Road](chapter-2-building-a-rag-the-paved-road/chapter-2-building-a-rag-the-paved-road.md) | How do I build one without inventing a pipeline? |
| 3 | [Exposing a RAG Securely](chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md) | How do I let others use it without losing control? |
| 4 | [Proving Retrieval Quality](chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) | How do I know it is right, and where did it go wrong? |
| 5 | [Cost Attribution and Chargeback](chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md) | Who pays when one team's question runs on three teams' infrastructure? |
| 6 | [Security and Safe Operations](chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md) | What happens when retrieved content is an attack? |
| 7 | [Keeping Knowledge Fresh](chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md) | How do I stop it becoming confidently out of date? |
| 8 | [Data You Cannot Move](chapter-8-data-you-cannot-move/chapter-8-data-you-cannot-move.md) | What if the knowledge is not mine to copy? |
| 9 | [On-Premises and Air-Gapped](chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md) | What if there is no cloud at all? |
| 10 | [Discovering What Already Exists](chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md) | How does anyone find and reuse what has been built? |
| 11 | [Measuring Whether It Works](chapter-11-measuring-whether-it-works/chapter-11-measuring-whether-it-works.md) | Is this earning its place, and where do we start? |

### Going deeper

Each chapter stays at reading length. The detail sits alongside it:

- [Graph RAG, explained](chapter-1-why-one-central-rag-fails/graph-rag-explained.md)
- [Managed search in depth](chapter-2-building-a-rag-the-paved-road/managed-search-in-depth.md) · the three tier documents, written to the same structure so they can be compared section for section: [Tier 1 — Managed](chapter-2-building-a-rag-the-paved-road/pipeline-tier-1-managed.md) · [Tier 2 — Assembled](chapter-2-building-a-rag-the-paved-road/pipeline-tier-2-assembled.md) · [Tier 3 — Bespoke](chapter-2-building-a-rag-the-paved-road/pipeline-tier-3-bespoke.md)
- [Securing MCP and A2A interfaces](chapter-3-exposing-a-rag-securely/securing-mcp-and-a2a-interfaces.md)
- [Tracing across the agent mesh](chapter-4-proving-retrieval-quality/tracing-across-the-agent-mesh.md)
- [The cost optimisation catalogue](chapter-5-cost-attribution-and-chargeback/cost-optimisation-catalogue.md)
- [Defending against indirect prompt injection](chapter-6-security-and-safe-operations/defending-against-prompt-injection.md)
- [Choosing between ingest, tool and agent](chapter-8-data-you-cannot-move/choosing-ingest-tool-or-agent.md)

---

## Where to start

**Read it in order** if you are designing the estate. The argument is cumulative — each chapter exists because the previous one created its problem.

**Jump to your pain** if you already have something running:

| If you are… | Start at |
| --- | --- |
| Choosing between managed search and building a pipeline | [Chapter 2](chapter-2-building-a-rag-the-paved-road/chapter-2-building-a-rag-the-paved-road.md) |
| Being asked to let another team use your RAG | [Chapter 3](chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md) |
| Unable to explain why an answer was wrong | [Chapter 4](chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) |
| Holding an AI bill nobody can break down | [Chapter 5](chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md) |
| Facing a security review | [Chapter 6](chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md) |
| Discovering that answers have quietly gone stale | [Chapter 7](chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md) |
| Blocked because the data belongs to someone else | [Chapter 8](chapter-8-data-you-cannot-move/chapter-8-data-you-cannot-move.md) |
| Working without a public cloud | [Chapter 9](chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md) |
| Watching teams rebuild what already exists | [Chapter 10](chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md) |
| Asked whether any of this is working | [Chapter 11](chapter-11-measuring-whether-it-works/chapter-11-measuring-whether-it-works.md) |
| Starting from nothing, or from one central RAG | [Chapter 11 §11.6](chapter-11-measuring-whether-it-works/chapter-11-measuring-whether-it-works.md#116-starting-from-where-you-are) |

---

**The thesis in one line:** the organisations that win at enterprise AI will not be the ones with the best models — models are a commodity and improve for everyone at once. They will be the ones whose knowledge is owned, current, discoverable, governed and reusable, because that is the part no vendor supplies and the part that compounds.

[Start with Chapter 1 →](chapter-1-why-one-central-rag-fails/chapter-1-why-one-central-rag-fails.md)
