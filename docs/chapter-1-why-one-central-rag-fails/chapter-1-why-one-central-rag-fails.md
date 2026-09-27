# Chapter 1 — One RAG for the Whole Enterprise, and Why It Breaks

> **The setup.** This chapter states the ambition, shows why the obvious architecture fails, gives you the answer — and then shows you the bill that answer comes with. Every chapter after this one pays off a line from the list at the end.

---

## 1.1 The Ambition

Every large organisation is sitting on decades of written knowledge. Policies, runbooks, contracts, design records, incident post-mortems, product specifications, customer correspondence, regulatory filings. Most of it is text. Almost none of it is in one place.

And the backlog is the smaller half of the problem. Every product team is generating more of it every day — new services, new decisions, new documentation, new operational records — and every data store a team creates is a potential source for retrieval. The corpus is not a fixed estate to be indexed once. It grows at the rate the organisation works, in places nobody has enumerated yet, under owners who do not know they have become knowledge owners. **Any architecture chosen here is being chosen for a corpus several times larger than the one you can see**, and that is the constraint that decides it.

Retrieval-Augmented Generation is the obvious unlock. Let people ask questions in their own words and get answers grounded in the organisation's own documents, with citations, instead of hunting through six systems and asking three colleagues.

The prototype is easy and it always works. Point an ingestion script at a folder, embed the chunks, put them in a vector store, wire an LLM in front. A week's work, a genuinely impressive demo, and an executive asking the obvious next question:

> *"Great. Now do that for the whole company."*

That question is where the trouble starts — because the natural way to answer it is to scale the prototype up. One ingestion pipeline pointed at everything. One embedding model. One vector database. One index. One endpoint.

It is the wrong answer, and it fails in ways that are structural rather than fixable.

---

## 1.2 Why One Central RAG Fails

These are not scaling problems. Buying a bigger vector database does not solve any of them.

### The access-control blind spot

Your source systems have rich, hard-won permission models. SharePoint sites have membership. Confluence spaces have restrictions. Repositories have visibility rules. Drives have per-file sharing. Those models took years to get right, and they are the actual legal boundary between an employee and information they are not entitled to see.

The moment a document becomes a chunk and that chunk becomes a vector, none of that survives *by default*. A vector is a list of floating-point numbers; it carries no group membership and no notion of who is allowed to read it. Permissions persist only if something deliberately carries them — reader principals extracted at ingestion, attached to every chunk, kept current as the source changes, and enforced as a filter before the search runs. Managed search services increasingly do this for you, and it is one of the strongest arguments for using one; a self-assembled pipeline does it only if someone builds it, and most do not.

That is the real shape of the problem. It is not that permissions are impossible to preserve — it is that preserving them is a design property somebody must own, and a single central index spanning every source system multiplies the number of permission models that must be mirrored, reconciled and kept fresh simultaneously. Each source added makes the mirror harder to keep true.

Teams patch around this with post-filtering: retrieve first, then check whether the user was allowed to see what you just retrieved. This is a leak waiting to happen, and the failure mode is bleak. The system retrieves a restricted document, the filter is imperfect or the metadata is stale, and an answer citing next quarter's redundancy plan lands in front of somebody who is named in it.

**The right order is the opposite one: decide entitlement first, retrieve second.** A central index makes that structurally hard, because it has stripped away the very information the decision requires.

### The embedding collision wall

This is the one that ends the argument, and it is worth being precise about.

An embedding model maps text into a vector space. Similarity search works by measuring distance in that space. The entire mechanism depends on every vector in the index having been produced by the *same* model, in the *same* space.

Now look at what actually happens in a large organisation. One team is already live on a Google embedding model at 768 dimensions. Another built on OpenAI at 3072. A third is on an open-weights model they run themselves because their data is not allowed to leave the building.

```text
Team A vector   ->  [ 0.12, -0.45,  0.89, ...  768 values ]
                              ✕  no meaningful distance exists
Team B vector   ->  [ -0.01, 0.72, -0.34, ... 3072 values ]
```

There is no cosine similarity between these. Not a poor one — *none*. The operation is undefined. Different dimensionality, different geometry, different training. You cannot put both in one index and search across them.

So the central design implies a corporate mandate: one embedding model, everywhere, for everyone.

That mandate is unenforceable, and even if you enforced it you would regret it. A materially better model ships every few months. Upgrading means re-embedding the entire corporate corpus in one coordinated operation — every team, every document, simultaneously — while the old index stays live. The cost is enormous, the coordination is impossible, and the day you finish, a better model exists.

**A shared corporate vector database is not merely hard to run. It is mathematically constrained in a way that guarantees it will age badly.**

### Cross-domain noise, and the hallucinations that follow

Semantic search has no concept of organisational context. Ask *"what is the approval process?"* against an index containing everything, and you will retrieve the procurement approval process, the code review approval process, the marketing copy approval process and the expenses approval process — all genuinely similar, all confidently irrelevant.

The model now has four incompatible procedures in its context window and no principled way to choose. It will produce something fluent. It will often produce something blended — a procedure that does not exist anywhere in your organisation, assembled from fragments of four that do.

This is the most damaging failure mode, because it does not look like a failure. There is no error, no empty result. Just an authoritative, well-cited, subtly wrong answer. **Retrieval precision degrades as corpus breadth grows, and hallucination risk rises with it.**

### Ingestion contention

One pipeline feeding everything has to serve wildly different rhythms at once: commits landing continuously, transactional data changing by the second, wikis edited all day, policy documents revised twice a year.

Tuned for the fast sources, it burns money re-processing documents that never change. Tuned for the slow ones, the fast-moving content is perpetually stale. There is no correct setting, because there is no single correct setting for four different problems sharing one queue. And it is one pipeline, so it is one failure domain: a malformed batch from one source stalls ingestion for everybody.

### Nobody owns the content

This is the quiet one, and in practice it is the one that kills adoption.

When the index belongs to a central platform, no one is accountable for what is *in* it. If retrieval returns a policy superseded two years ago, whose job is it to fix? The central team does not know it is obsolete — they do not work in that domain. The domain team does not know it is in there, and has no way to remove it.

**Correctness depends on curation, and curation only happens where expertise lives.** A central index systematically separates the two.

---

## 1.3 The Answer: Ownership Follows the Data

Stop building one enterprise RAG. Build many small ones, each owned by the team that owns the underlying data.

### The ownership rule

> **The team that produces and owns the data owns the RAG over it.**

That rule is deliberately narrow, because it has to settle arguments. It is not "one RAG per department" or "one per business unit" — those are negotiable, and anything negotiable will be re-negotiated at every reorganisation.

Take a concrete case. Customer profiles are created and updated by the **Customer Master** team. Customer Master sits inside a broader **Customer** domain alongside several other teams. The RAG over customer master data belongs to **Customer Master** — the team — not to the Customer domain.

Why the team and not the domain? Because the team is where the knowledge is. They know which fields are authoritative, which documents are superseded, what "active customer" means this quarter. A domain is a grouping on an org chart. A team is a set of people who can answer a question about their data correctly.

This rule also settles the questions that follow: who fixes a bad answer, who approves access, who pays, and what happens to a RAG when its team is reorganised. All of them resolve to "whoever owns the data."

```text
                 ┌───────────────────────────┐
                 │   Enterprise entry point  │
                 └─────────────┬─────────────┘
                               │
     ┌─────────────────────────┼─────────────────────────┐
     │                         │                         │
     ▼                         ▼                         ▼
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│ Customer     │        │ Payments     │        │ People Ops   │
│ Master team  │        │ Core team    │        │ team         │
├──────────────┤        ├──────────────┤        ├──────────────┤
│ owns data    │        │ owns data    │        │ owns data    │
│ owns corpus  │        │ owns corpus  │        │ owns corpus  │
│ owns index   │        │ owns index   │        │ owns index   │
│ owns quality │        │ owns quality │        │ owns quality │
├──────────────┤        ├──────────────┤        ├──────────────┤
│ governed     │        │ governed     │        │ governed     │
│ interface    │        │ interface    │        │ interface    │
└──────────────┘        └──────────────┘        └──────────────┘
```

Each unit decides what enters its corpus, what gets retired, how it is chunked, which model embeds it, and what quality bar it holds itself to. Each exposes a governed interface. None of them exposes its raw index.

### Encapsulation: exchange text, never vectors

This is the load-bearing principle of the whole book, so it gets stated flatly:

> **Nothing outside a team's boundary ever sees its vectors. Callers send text. They receive text.**

When another system needs something from Customer Master's knowledge, it does not connect to a database, and it certainly does not send an embedding to compare against. It sends a question in plain language, through a governed interface. Customer Master's own service embeds that question with its *own* model, searches its *own* index, applies its *own* entitlement rules, and returns a text answer with citations.

That is it. That is the whole contract. And it is why the rest of the architecture becomes possible.

### How this actually resolves the five failures

Worth being explicit, because the value is in the mechanism, not the shape of the diagram.

**Access control** — entitlement is now decided by the team that owns the data, inside their boundary, *before* retrieval is attempted, using the real permission model rather than a copy of it. The check happens where the truth lives. (Enforcement mechanics are Chapter 3.)

**Embedding collision** — dissolved rather than solved. Vector spaces never need to be compatible because they never meet. Customer Master can move to a new embedding model on a Tuesday and nobody else is affected or even informed. There is no corporate embedding standard to enforce, because there is no shared index to enforce it in.

**Cross-domain noise** — a question about approval processes now goes to the team that owns approvals. The corpus it searches contains one approval process. Precision rises because breadth fell, and the blended-answer failure mode largely disappears.

**Ingestion contention** — each corpus is refreshed on the rhythm its source actually has. Fast sources stay fresh, slow sources stay cheap, and one team's malformed batch is one team's problem.

**Ownership of content** — the people who know a document is obsolete are the people who can remove it. Curation and expertise are in the same place.

### When a single RAG is genuinely the right answer

This book argues for federation, so it should be honest about when federation is wrong.

If you have one product team, a few thousand documents from two or three systems, and a single permission model, **build one RAG.** Federating is overhead you will pay for and get nothing back from.

The forcing conditions are:

- **Multiple incompatible permission models.** One is fine. Five that disagree is not.
- **Multiple teams who genuinely own distinct bodies of knowledge.** If one team can curate all of it, one team should.
- **Enough breadth for cross-domain noise to bite.** Retrieval precision collapsing across domains is the practical signal.
- **Teams already moving independently.** If two teams have already chosen different embedding models, the decision was made for you.

Below that threshold, a single well-governed RAG with strong curation beats a federation of three. Federation is a response to organisational scale, not a sign of technical sophistication.

---

## 1.4 The Bill

So decentralise. The diagram is clean, each team owns its own knowledge, and the failures above are genuinely resolved.

Now count what you just agreed to.

Every one of those RAGs needs a pipeline, infrastructure, an ingestion path, an embedding strategy, a secure interface, an identity model, authorisation, tracing, evaluation, a refresh cadence, a retirement policy, a cost model and an owner. **You did not eliminate the work. You multiplied it by the number of teams.**

And this is where most federated architectures quietly fail. Not at the diagram — the diagram is fine. They fail eighteen months later, when twelve teams have each independently solved twelve versions of the same twelve problems, at twelve different quality levels, with twelve different security postures, and nobody can say with confidence which of them is safe to connect to a customer-facing assistant.

**That outcome is worse than the monolith you escaped.** The monolith was at least consistently bad. Twelve divergent implementations are inconsistently bad, and inconsistency is what makes an estate ungovernable.

So the decentralisation argument is only half an argument. The other half is this:

> **Teams should own their RAG. They should not have to invent how to run one.**

Everything hard, repeated and cross-cutting — the pipeline, the interface standards, the identity plumbing, the tracing backbone, the evaluation harness, the cost ledger, the registry — is built once, by a platform team, and handed to every team as something configured rather than something built.

**Decentralised execution. Centralised principle, governance, standards and boilerplate.**

The team keeps control: what enters the corpus, what it means, what good looks like, when it refreshes. It runs inside its own boundary, on its own data. What it does not do is write the pipeline, design the auth model, or invent a tracing schema. Those arrive as a versioned, parameterised, supported capability — closer to a library the team adopts than a service they are subject to.

Chapter 2 onward is that capability, one problem at a time.

---

## 1.5 The Problem Set

Here is the bill in full. Each line is a problem created or amplified by decentralising, and each is picked up properly later in the book. They are grouped rather than ranked — the order you meet them depends on where you are starting from, which is Chapter 11's subject.

**Building one**
- **Which kind of pipeline?** Fully managed search, a parameterised platform pipeline, or something bespoke — and who decides. → Chapter 2
- **Infrastructure without an infrastructure team.** Every team now needs runtime, storage and networking. None should be building it. → Chapter 2
- **Embedding choice and lifecycle.** Free to choose, but not free to drift. Model upgrades need a runbook, not an outage. → Chapter 2
- **Ingestion quality.** Bad parsing produces bad chunks produces bad answers, silently. → Chapter 2
- **RAG or just a long context window?** Modern models take enormous prompts. Knowing when retrieval earns its place matters. → Chapter 2

**Exposing it**
- **Which interface — MCP tool or agent-to-agent?** They are not interchangeable, and the choice decides where domain judgement lives. → Chapter 3
- **Identity that cannot be forged,** asserted by the issuer and never by the caller. → Chapter 3
- **Authorisation before retrieval, at tool granularity.** Reaching a server is not the same as being entitled to every tool on it. → Chapter 3
- **Callers who are not people.** Batch jobs, workflows and other agents are first-class identities, including delegation chains. → Chapter 3
- **Contract versioning.** One team's interface change should not break every consumer downstream. → Chapter 3

**Knowing it is right**
- **Measuring quality at all** — and separating a retrieval failure from a generation failure, because the fixes are different. → Chapter 4
- **Tracing across hops.** When an answer is wrong, you need the whole path, not one service's logs. → Chapter 4
- **Closing the feedback loop.** Signals that do not cause a change in the system are decoration. → Chapter 4
- **Conflicting answers.** Two RAGs, two answers, both defensible. Which is authoritative? → Chapter 4
- **Testing a system that is not deterministic.** Regression across a rebuild, contracts between teams, load, and the fact that the same input may not produce the same output. → Chapter 4

**Paying for it**
- **Attribution across boundaries.** Team A's assistant calls Team B's RAG; Team B's budget pays. → Chapter 5
- **A price signal for consumers,** who currently see none. → Chapter 5
- **The gateway** — what it is for, where it should sit, and what it costs you if you put it in the wrong place. → Chapter 5
- **Optimisation,** available in more places than most teams realise. → Chapter 5
- **The cost of the governance itself.** Evaluation, judging, tracing and re-embedding are not free. → Chapter 5

**Keeping it safe**
- **Prompt injection through retrieved content.** The RAG-native attack, delivered by your own ingestion pipeline. → Chapter 6
- **Rate limiting, quota fairness and denial of wallet.** → Chapter 6
- **Deletion that actually propagates** — into chunks, vectors, caches, feedback logs and traces — plus legal hold. → Chapter 6
- **Classification, residency and sovereignty.** Retrieval moves data across boundaries at machine speed. → Chapter 6
- **Environments and test data,** so that "let us test against production" stops being the only option. → Chapter 6
- **Audit.** Who saw what, and when. → Chapter 6

**Keeping it true**
- **Refresh economics.** Reprocessing everything is expensive; stale answers are wrong. → Chapter 7
- **Retiring knowledge** — purge it, or tombstone it and keep the lineage. → Chapter 7
- **Freshness as a declared, measured property,** not a uniform schedule. → Chapter 7
- **Detecting silent decay** before users find it. → Chapter 7
- **Healing without a human.** Most decay is mechanical and repetitive, and should be repaired automatically rather than queued for someone's morning. → Chapter 7
- **Incident response.** "The assistant said something wrong" needs a runbook, a rollback and a kill switch. → Chapter 7

**Data you do not control**
- **External and third-party sources** — bring the data in, reach it in place, or consume an agent over it. → Chapter 8
- **Data that must stay where it is,** for licensing, contractual or practical reasons. → Chapter 8
- **Consuming another team's knowledge** without copying it and recreating the monolith. → Chapter 8
- **Disconnected, on-premises and air-gapped estates.** No cloud is not a reason to go without. → Chapter 9

**Running it as an estate**
- **Discoverability.** Nobody can use what nobody can find, so they build it again. → Chapter 10
- **Requesting and granting access** without a meeting for every integration. → Chapter 10
- **Lifecycle of a RAG itself** — reorganisations, orphans, decommissioning. → Chapter 10
- **Proving it is working,** to the people who approve budgets. → Chapter 11
- **Getting there from here.** Nobody starts on a blank page. → Chapter 11

That is the bill. No team should face it alone, and no team should solve any of it twice.

---

## What the platform team provides for this chapter

| Platform provides | Team owns |
| --- | --- |
| The ownership rule and how boundaries are drawn | Deciding what belongs in its corpus |
| The federated reference architecture | Its own domain's knowledge quality |
| The registry that makes RAGs findable *(Chapter 10)* | Registering and describing its own |
| Guidance on when federating is *not* warranted | Choosing to adopt the paved road |

---

**Supporting material for this chapter**

- [Graph-based RAG: what it is, why it exists, and when it earns its complexity](graph-rag-explained.md)

---

[← Introduction](../README.md) | [Chapter 2 — Building a RAG: The Paved Road →](../chapter-2-building-a-rag-the-paved-road/chapter-2-building-a-rag-the-paved-road.md)
