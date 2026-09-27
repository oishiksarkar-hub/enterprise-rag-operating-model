# Chapter 9 — On-Premises and Air-Gapped

> **The bridge.** Every chapter so far named a managed service, a serverless platform or a hosted model endpoint. For a great many organisations, none of those are available. This chapter is about what changes — and, more importantly, what does not.
>
> **This chapter covers** disconnected, on-premises, sovereign and air-gapped estates: which principles hold unchanged, which patterns need substitution, and what genuinely degrades.
>
> **Scope note.** This is a starting point, not a deep dive. Deploying models on your own infrastructure is a large subject with its own literature. What follows is the architectural position: which principles hold, which patterns need substitution, and what genuinely degrades.

---

## 9.1 The Problem: The Same Need, Without the Assumed Substrate

The organisations concerned are not unusual, and they are not small.

A defence or intelligence agency on a network with no external connectivity by design. A central bank whose supervisory data cannot leave its own facilities. A hospital network under data-protection rules that exclude external processing of patient records. A manufacturer whose plant control systems sit on an isolated operational network. A government operating a sovereign cloud precisely so that it need not depend on foreign-owned infrastructure.

Every one has the problem this book opens with: decades of accumulated knowledge, scattered across systems, that people need answers from. **The constraint is on the substrate, not on the need.**

And the constraint is rarely all-or-nothing. Four positions, with different consequences:

| Position | Connectivity | What it rules out |
| --- | --- | --- |
| **Sovereign cloud** | Public cloud stack, operated in-country under local control | Some managed services; usually the newest ones |
| **Private cloud** | Own data centre, cloud-like platform, some egress | Managed AI services entirely |
| **On-premises, connected** | Own infrastructure, controlled egress for updates | Hosted model endpoints |
| **Air-gapped** | No external connectivity at all | Anything requiring a network call outward |

Teams in these environments are routinely told, implicitly, that modern AI is not for them. That is wrong, and the cost of believing it is high — these are often the organisations whose institutional knowledge is oldest, largest and least accessible.

---

## 9.2 The Principle: The Architecture Was Never About the Cloud

This chapter is short for a reason worth stating plainly.

> **Every principle in this book is a property of the architecture, not of any provider. Ownership follows data. Encapsulate behind a governed interface. Authorise before retrieval. Trace every hop. Attribute cost to the causing identity. Treat retrieved content as untrusted. Declare freshness and measure it.**
>
> **Not one of those requires a public cloud. Each needs a different implementation, and the same implementation everywhere within a given estate.**

This is why the book has kept vendor examples in marked blocks. Remove them and the argument is unchanged.

What does change is the platform team's position — and it changes in their favour.

**On public cloud, the platform team assembles.** Managed services exist; the work is integration, governance and paving the road.

**On-premises, the platform team builds.** Model serving, the vector store, the identity plumbing, the observability stack. More work, and a stronger justification for the team existing at all. **In a constrained environment, per-team self-service is not merely inefficient — it is impossible.** No product team can stand up GPU-backed model serving on its own. The platform team is not a nice-to-have; it is the only way the capability exists.

---

## 9.3 What Substitutes for What

| Capability | Public cloud | On-premises / air-gapped |
| --- | --- | --- |
| Managed search (Tier 1) | Managed service | Self-hosted search stack with a vector plugin, operated by the platform as a shared service |
| Vector store | Managed | PostgreSQL with `pgvector`, or a self-hosted vector database |
| Embedding model | Hosted endpoint | Open-weight model served on local GPUs |
| Generation model | Hosted endpoint | Open-weight model served on local GPUs |
| Reranker | Managed | Small local cross-encoder — cheap, high value |
| Agent runtime | Managed platform | Self-hosted framework on the container platform |
| MCP / A2A transport | Same protocol | **Identical** |
| Identity | Cloud IdP | Existing enterprise IdP — usually already present |
| Tracing | Managed backend | Self-hosted collector and store |
| Cost attribution | Provider billing export | Internal chargeback over GPU-hours and utilisation |
| Object storage | Managed | On-premises S3-compatible storage |

**Tier 1 does not disappear; it relocates — and it changes character.** Chapter 2's argument — that most teams should consume a managed capability rather than build a pipeline — is *more* important here, not less. The difference is that the platform team is the provider. A shared, well-operated internal search and retrieval service is what stops eleven teams each attempting to run their own vector database.

But be clear about what is being substituted. **The two tiers of this stack do not port equally.**

The model tier ports well. Open-weight models can be served on your own GPUs, behind your own endpoint, with an interface close enough to a hosted one that most calling code does not care. Vendor offerings for disconnected environments generally lead with exactly this — model serving — because it is the part that packages cleanly.

The retrieval tier does not port. The managed search product from Chapter 2 — the one that does extraction, chunking, indexing, hybrid retrieval, reranking and per-document permission filtering as a single configured service — has no disconnected equivalent, from any provider. What you get instead is a database with vector support and a set of components you assemble yourself.

The practical consequence: **in a disconnected estate, every team is effectively at Tier 2 or Tier 3, and the platform team's job is to make that feel like Tier 1.** The shared internal retrieval service *is* the paved road, and if the platform team does not build it, eleven teams will each build a worse one.

> **Worth checking before you assume otherwise.** Disconnected and air-gapped product offerings are among the fastest-changing and least-documented parts of any provider's catalogue. Availability, model versions and service coverage differ by provider, by deployment mode and by region, and they differ from the connected offering in ways that are rarely summarised anywhere. Treat any list, including this one, as a prompt to verify rather than a statement of fact.

**The interface layer is unchanged.** MCP and A2A are protocols. They work identically over an internal network. Everything in Chapter 3 applies without modification, which means the security model, the boilerplate and the conformance suite are portable across both worlds.

---

## 9.4 What Is Genuinely Harder

Honesty matters more than reassurance here.

**Model capability.** The strongest open-weight models are very good and generally trail the strongest proprietary ones. For most enterprise RAG — synthesising retrieved passages into a grounded answer — the practical gap is usually smaller than the benchmark gap, because the model is working from supplied text rather than from what it knows. **Our working assumption, and it is a judgement rather than a measured finding, is that improving retrieval buys more than upgrading the generator.** A strong model cannot answer from a passage it was never given. Treat it as a default allocation of effort, not a law: spend on chunking, hybrid search and reranking first, then revisit model size with your own evaluation set — and if your evaluation says otherwise for your corpus, believe your evaluation.

**Embedding models age in place.** The embedding models available for disconnected deployment typically lag the hosted ones — older architectures, fewer languages, smaller maximum input length. That last one has a direct architectural consequence: if the local model accepts a materially shorter input than the hosted equivalent, your chunking strategy is constrained by the model rather than by the document, and passages that would have embedded as a unit elsewhere must be split here. Check the maximum input length and the supported languages of the *specific* model available to you before designing chunking around them, because this is the constraint most likely to be discovered late.

**Capacity planning.** No scale-to-zero, no elastic burst. You buy GPUs, and they are either idle or saturated. Peak-shaping, batch scheduling and queueing matter far more than in an elastic environment, and capacity is a procurement cycle rather than an API call.

**Operational load.** Model serving, GPU driver stacks, quantisation, batching, index operations. This is a real, permanent staffing commitment. It is also exactly why it must be centralised — the load is bearable once and unbearable eleven times.

**Model updates in an air gap.** New model weights arrive through a physical transfer and an approval process. Plan on quarterly, not continuous. Version pinning ([Chapter 2](../chapter-2-building-a-rag-the-paved-road/chapter-2-building-a-rag-the-paved-road.md)) is easy here; keeping current is not.

**No external sources at all.** Chapter 8's Options B and C only reach sources inside the boundary. External knowledge — regulatory updates, vendor documentation, public standards — arrives as a periodic controlled import, and freshness is bounded by that cadence. Say so in the freshness tier rather than pretending otherwise.

**Long context is effectively unavailable.** This is stronger than "windows are smaller", and it is worth planning around rather than discovering. Hosted frontier models advertise context windows measured in hundreds of thousands or millions of tokens. Models served in a disconnected environment commonly offer an order of magnitude less — tens of thousands, not hundreds of thousands — because the window is bounded by the memory on the hardware you actually bought, not by the provider's fleet.

The consequence is not subtle. Chapter 2 presents retrieval and long context as a genuine decision with a defensible answer on either side. **Here, one side of that decision is gone.** "Just put the whole document in the prompt" is not an available architecture for anything of real size. Retrieval is not the preferred option; it is the only option, and the quality of your chunking and reranking is therefore load-bearing in a way it is not elsewhere.

---

## 9.5 What Is Genuinely Easier

Less often said, and true.

**Residency and sovereignty are solved by construction.** Chapter 6's most awkward constraint — ensuring the *model* is in-region, not only the index — is trivially satisfied. Everything is in one place, by definition.

**Data never leaves.** The entire category of concern about content being sent to an external processor does not arise. For the most sensitive corpora this is not a convenience; it is the only acceptable posture, and it is why these organisations chose this substrate.

**Per-query model cost is zero.** Cost is capital and operational, not transactional. Chapter 5's attribution model still applies, over GPU-hours rather than tokens — and it becomes a **utilisation** question, which is easier to reason about than a per-token one. The counterpart is that idle capacity is pure waste, so the pressure runs in the opposite direction.

**No provider dependency.** No deprecation notices, no terms changes, no regional availability surprises, no rate limits imposed from outside.

**The network is smaller and more governable.** Egress control, allow-listing and segmentation are all more tractable.

---

## 9.6 Federating Across a Boundary

Many organisations are not uniformly one thing. A classified enclave alongside a corporate network. A plant network alongside head office. A sovereign region alongside a global estate.

> **The mesh federates up to the boundary, and stops there. Cross-boundary knowledge moves as a deliberate, controlled transfer — never as a live query.**

```text
   CONNECTED SIDE                    ┃      RESTRICTED SIDE
                                     ┃
   agents ◄──► agents                ┃      agents ◄──► agents
     │                               ┃        │
     ▼                               ┃        ▼
   RAGs, MCP tools                   ┃      RAGs, MCP tools
                                     ┃
   full mesh, Chapters 1–8 apply     ┃      full mesh, Chapters 1–8 apply
                                     ┃
              ┌──────────────────────┸──────────────────────┐
              │  CONTROLLED TRANSFER                        │
              │  scheduled · reviewed · one-directional     │
              │  · content, never queries                   │
              │  · classification-checked at the boundary   │
              │  · audited both sides                       │
              └─────────────────────────────────────────────┘
```

**Both sides are complete meshes.** Neither depends on the other to function. This is the property that makes the arrangement survivable.

**Transfer is content, not connectivity.** Public standards, vendor documentation and regulatory updates are imported into the restricted side on a schedule. Nothing on the restricted side makes a live call outward — that would be a connectivity path, which is the thing the boundary exists to prevent.

**Direction is asymmetric and explicit.** Inbound is usually permitted under review. Outbound is usually prohibited, or permitted only for specific reviewed artefacts.

**Freshness is bounded by transfer cadence**, and declared as such. A restricted-side corpus of external standards is weekly-fresh at best, and its freshness tier should say weekly, not daily.

**The registry spans both sides in metadata only.** A team on the restricted side can see *that* a capability exists on the connected side, its owner and how to request a transfer — without being able to call it. See [Chapter 10](../chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md).

---

## 9.7 What the Platform Team Provides

The list is longer here than anywhere else in the book, and that is the point.

| Platform provides | Team owns |
| --- | --- |
| Model serving — embedding, generation, reranking — as a shared internal service | Choosing from the approved models |
| GPU capacity planning, scheduling and fair-share allocation | Forecasting their demand |
| The shared retrieval service — the on-premises Tier 1 | Their corpus and configuration |
| Self-hosted vector infrastructure | Nothing; consumed as a service |
| The same MCP/A2A boilerplate, unchanged from the cloud build | Their tools and agents |
| Identity integration with the enterprise IdP | Registering their workloads |
| Self-hosted tracing, evaluation and cost attribution | Using them exactly as in Chapters 4–5 |
| The model import and approval pipeline | Requesting models |
| Cross-boundary transfer process and tooling | Requesting transfers; declaring classification |
| Internal chargeback over GPU-hours and utilisation | Their consumption |

**The test of success is identical to Chapter 3's.** A team should be able to build and expose a RAG on this substrate with roughly the effort it would take on a public cloud. If the on-premises path is dramatically harder, the platform has not finished the work — and teams will either give up or build something unsupported in a corner.

---

## Where this leaves you

Nine chapters. A team can build a RAG, expose it safely, prove it is right, pay for it, defend it, keep it true, reach data it does not own, and do all of it without a public cloud.

Which leaves a problem created entirely by that success.

Chapter 1 said: break the monolith into many team-owned RAGs. Chapters 2 through 9 made each of those buildable and governable. **Nothing so far has helped anyone find them.**

A team in Logistics needs supplier risk knowledge. Does a RAG for it exist? Who owns it? Is it current? What is in it and what is not? May they use it, and how do they ask? They cannot answer any of it — so they build their own, over the same documents, badly, and now there are two.

That is the monolith's fragmentation problem, reappearing in a new form. And it is worse than an inconvenience: the chargeback model in Chapter 5, the deletion propagation in Chapter 6, the authority resolution in Chapter 4, and the ownership lookup in every incident all assumed a record of what exists.

None of it works without one.

---

[← Chapter 8 — Data You Cannot Move](../chapter-8-data-you-cannot-move/chapter-8-data-you-cannot-move.md) | [Chapter 10 — Discovering What Already Exists →](../chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md)
