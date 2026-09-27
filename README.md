# Enterprise RAG: An Operating Model

### Enterprise RAG doesn't scale. Neither does breaking it up — without a platform team.

Eleven chapters on what it actually takes to run retrieval-augmented generation across a large organisation: why the obvious architecture fails, what replaces it, and the bill that replacement comes with.

Written for enterprise architects, platform engineering leaders, and the executives who have to fund this.

---

## The argument, in ninety seconds

**The prototype always works.** Point a script at a folder, embed the chunks, wire a model in front. A week's work, an impressive demo, and an executive asking the obvious next question: *now do that for the whole company.*

**Scaling that prototype is the wrong answer**, and it fails structurally rather than technically. Two teams choosing different embedding models produce vectors that are mathematically incomparable — one index means one model for everyone, forever. Entitlement rules live with the systems that produced the data and are understood only by the teams that own them; a central index must mirror all of them, permanently and accurately, or it leaks. A corpus spanning every domain retrieves from unrelated ones and synthesises confident nonsense. And nobody owns the content, so it decays.

**So ownership follows the data.** The team that produces and maintains knowledge owns the RAG over it, and exposes it behind a governed interface that returns text and citations — never raw vectors, never the underlying store.

**That decision is correct, and it creates a new problem set** — multiplied by every team in the organisation. Pipelines, identity, authorisation, tracing, evaluation, cost attribution, freshness, incident response. Solved naively, that is worse than the monolith you replaced.

**Which is what the platform team is for.** Not to own the knowledge — it cannot, and should not try. To make every expensive decision once, and make the right thing the easy thing, eleven times over.

> **The principle is centralised. The execution is not.** One embedding registry, many corpora. One interface contract, many interfaces. One tracing standard, many services. One paved road, many destinations.

---

## Read it

**→ [Start here](docs/README.md)** — the book's introduction, the full map, and how to read it.

Or go straight to a chapter:

| | Chapter | The question it answers |
| --- | --- | --- |
| 1 | [Why One Central RAG Fails](docs/chapter-1-why-one-central-rag-fails/chapter-1-why-one-central-rag-fails.md) | Why is the obvious architecture wrong, and what replaces it? |
| 2 | [Building a RAG: The Paved Road](docs/chapter-2-building-a-rag-the-paved-road/chapter-2-building-a-rag-the-paved-road.md) | How do I build one without inventing a pipeline? |
| 3 | [Exposing a RAG Securely](docs/chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md) | How do I let others use it without losing control? |
| 4 | [Proving Retrieval Quality](docs/chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) | How do I know it is right, and where did it go wrong? |
| 5 | [Cost Attribution and Chargeback](docs/chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md) | Who pays when one team's question runs on three teams' infrastructure? |
| 6 | [Security and Safe Operations](docs/chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md) | What happens when retrieved content is an attack? |
| 7 | [Keeping Knowledge Fresh](docs/chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md) | How do I stop it becoming confidently out of date? |
| 8 | [Data You Cannot Move](docs/chapter-8-data-you-cannot-move/chapter-8-data-you-cannot-move.md) | What if the knowledge is not mine to copy? |
| 9 | [On-Premises and Air-Gapped](docs/chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md) | What if there is no cloud at all? |
| 10 | [Discovering What Already Exists](docs/chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md) | How does anyone find and reuse what has been built? |
| 11 | [Measuring Whether It Works](docs/chapter-11-measuring-whether-it-works/chapter-11-measuring-whether-it-works.md) | Is this earning its place, and where do we start? |

Each chapter stays at reading length. Eleven supporting documents carry the detail — graph retrieval, the three pipeline tiers, interface security, distributed tracing, cost optimisation, prompt injection defence, and the ingest-versus-tool decision. They are linked from the [introduction](docs/README.md).

---

## What this is, and what it is not

**It is vendor-agnostic by design.** Principles first, patterns second, a worked example third — in clearly marked blocks you can skip if you are on a different cloud, or on none at all. Where a block names a product, it is there to make an abstract argument concrete, and it should be verified against current documentation before you build on it.

**It covers cloud, private cloud, on-premises and air-gapped.** [Chapter 9](docs/chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md) sets out what genuinely changes without a public cloud, which is both less and more than most people assume.

**It assumes you are not starting from a blank page.** [Chapter 11](docs/chapter-11-measuring-whether-it-works/chapter-11-measuring-whether-it-works.md) covers migrating away from a central RAG that already exists, without switching it off and without a rewrite.

**It is not a tutorial, and it contains no runnable code.** The configuration shapes in the text are illustrative. Reference implementations are planned under [samples/](samples/README.md) but are not written yet, and no date is promised.

**It is opinionated about the things that are hard to reverse** — embedding models, chunking, identity mapping, metadata vocabulary — and deliberately unopinionated about the things that are not.

---

## Who it is for

| If you are… | Start at |
| --- | --- |
| Choosing between managed search and building a pipeline | [Chapter 2](docs/chapter-2-building-a-rag-the-paved-road/chapter-2-building-a-rag-the-paved-road.md) |
| Being asked to let another team use your RAG | [Chapter 3](docs/chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md) |
| Unable to explain why an answer was wrong | [Chapter 4](docs/chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) |
| Holding an AI bill nobody can break down | [Chapter 5](docs/chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md) |
| Facing a security review | [Chapter 6](docs/chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md) |
| Discovering that answers have quietly gone stale | [Chapter 7](docs/chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md) |
| Blocked because the data belongs to someone else | [Chapter 8](docs/chapter-8-data-you-cannot-move/chapter-8-data-you-cannot-move.md) |
| Working without a public cloud | [Chapter 9](docs/chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md) |
| Watching teams rebuild what already exists | [Chapter 10](docs/chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md) |
| Asked whether any of this is working | [Chapter 11](docs/chapter-11-measuring-whether-it-works/chapter-11-measuring-whether-it-works.md) |

---

**The thesis in one line:** the organisations that win at enterprise AI will not be the ones with the best models — models are a commodity and improve for everyone at once. They will be the ones whose knowledge is owned, current, discoverable, governed and reusable, because that is the part no vendor supplies and the part that compounds.

**[Start reading →](docs/chapter-1-why-one-central-rag-fails/chapter-1-why-one-central-rag-fails.md)**
