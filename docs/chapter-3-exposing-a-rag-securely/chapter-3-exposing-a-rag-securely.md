# Chapter 3 — How Do I Expose It Without Losing Control?

> **The bridge.** Chapter 2 got a RAG built. Chapters 1 and 2 both leaned on an "interface contract" — a governed surface that is the only way in — and neither one specified it. This chapter does.
>
> **This chapter covers** how a RAG is reached, establishing who is calling, deciding what they may do, handling non-human identity, and changing an interface without breaking its consumers.

---

## 3.1 The Problem: A Working RAG Is Not Yet an Asset

A team finishes Chapter 2 and has something that answers questions well. The obvious next step is to let other people use it. That step is where federated estates are quietly lost.

Consider what "letting people use it" usually means in practice.

**A colleague asks for access to the index.** Reasonable request, terrible outcome. They now query the vector store directly, which means every rule the owning team enforces — entitlement, filtering by classification, citation, logging — is bypassed, because those rules live in the application, not in the store.

**Another team wants the source data instead.** So it gets copied. Now there are two corpora, drifting apart from the day of the copy, with two sets of permissions and one deletion request that will only ever reach one of them.

**An agent is pointed straight at the underlying database.** It works immediately, which is the danger. There is now a path to the data that the owning team does not control, does not see, and cannot revoke without breaking something they did not build.

**An internal endpoint is shared informally.** No versioning, no contract, no authentication beyond network position. Six consumers appear. The team can no longer change anything.

Each of these is fast and locally sensible. Collectively they reproduce exactly the failure Chapter 1 set out to escape — except worse, because now the sprawl is *between* teams and no single team can fix it.

**The underlying error is treating exposure as a distribution problem.** It is not. It is a boundary problem. What is being published is not data; it is a *governed capability*, and it must arrive with its governance attached.

---

## 3.2 The Principle: The Only Path In Is a Governed Interface

One rule, and the rest of this chapter is its consequences.

> **A RAG is never reachable directly. It is reachable only through an interface that carries identity, enforces authorisation before retrieval, and emits telemetry — and there is no second path.**

Three clauses, each load-bearing.

**Never reachable directly.** Not the vector store, not the index, not the database behind it, not an unauthenticated internal endpoint. If a second path exists, the interface is decorative and governance is optional in practice.

**Carries identity, authorises before retrieval, emits telemetry.** These are not features of the service behind the interface. They are properties *of the interface itself*, which is why they hold no matter which tier from Chapter 2 is behind it and no matter how the owning team implemented it.

**No second path.** This is the clause organisations concede first and regret longest. One exception for one urgent integration becomes the precedent.

What this buys is the most valuable property in the architecture: **the inside can change completely and nothing outside notices.** Chapter 2's tier migrations, re-embedding, chunking changes, even a cloud move — all become internal detail. That freedom exists only because the boundary is absolute.

### Two interface shapes

Two protocol families have settled into the ecosystem, and they answer different questions.

**MCP (Model Context Protocol) — a capability surface.** You publish *tools*: named, typed, discoverable operations. The caller's model decides when to invoke them and with what arguments. Your side executes and returns results. You do not control the surrounding reasoning.

**A2A (Agent-to-Agent) — a reasoning surface.** You publish an *agent*. The caller sends a request in natural language and receives an answer. Your agent decides internally how to answer — which tools, how many retrieval passes, what to synthesise. You own the reasoning.

The distinction is not technical preference. It is about **where the domain judgement lives.**

| | Expose an **MCP tool** | Expose an **A2A agent** |
| --- | --- | --- |
| Best when | The operation is well-defined and self-contained | Answering needs domain reasoning and multiple steps |
| Caller supplies | Structured arguments | A question, in their own words |
| You return | Retrieved content, with citations | A synthesised answer, with citations |
| Reasoning owned by | The caller | You |
| Domain expertise needed by the caller | Considerable | None |
| Cost you incur | Retrieval only | Retrieval and generation |
| Changes safely when | Arguments and semantics stay stable | Your internal approach changes freely |
| Typical failure | Caller composes tools in ways you did not anticipate | Caller cannot tell why an answer came out as it did |

**A working rule.** If a competent outsider could use your surface correctly without knowing your domain, expose a tool. If getting a good answer requires knowing which sources supersede which, how the entities relate, and what the caveats are — expose an agent, and keep that knowledge on your side of the line.

Many teams should expose both: tools for precise, composable operations, and an agent in front for callers who just have a question.

> **Reference implementation.** MCP servers over HTTP with streaming; A2A over the agent framework of your platform. On Google Cloud that is typically Cloud Run for the MCP server and Vertex AI Agent Engine or Agentspace for the agent; on AWS, Bedrock Agents; on Azure, AI Foundry agents; on-premises, a self-hosted MCP server behind the internal service mesh. The contract is identical in all four cases, which is precisely the point.

---

## 3.3 The Pattern: What the Flow Actually Looks Like

This is where the rule becomes concrete, and where the common architecture diagram gets it wrong.

```text
   Consumer
   (a user's assistant, another team's agent, an automated workflow)
        │
        │  request + verifiable identity token
        ▼
   ┌──────────────────────────────────────────────┐
   │        THE BOUNDARY  (MCP tool or A2A agent) │
   │                                              │
   │   1. Authenticate      who is this, provably │
   │   2. Authorise tool    may they call THIS    │
   │                        tool at all           │
   │   3. Derive scope      which corpus subset   │
   │                        is this identity      │
   │                        entitled to           │
   │   4. ─── retrieval happens only now ───      │
   │   5. Filter results    post-retrieval check  │
   │   6. Emit telemetry    trace, usage, cost    │
   └──────────────────────────────────────────────┘
        │
        │  text + citations. Never vectors. Never raw records.
        ▼
   Consumer
```

And behind the boundary, inside the owning team's perimeter:

```text
   ┌─ owning team's perimeter ──────────────────────┐
   │                                                │
   │   MCP tool / A2A agent  ──►  retrieval layer   │
   │                                │               │
   │                                ├──► index      │
   │                                ├──► documents  │
   │                                └──► operational│
   │                                     database   │
   │                                                │
   │   Nothing in this box is addressable from      │
   │   outside it. The only door is the top edge.   │
   └────────────────────────────────────────────────┘
```

Three things this diagram is deliberately asserting:

**The flow terminates at an interface, not at a store.** A consumer-facing diagram that ends at a bucket, an index or a table has drawn the failure. The store is interior. If it appears on the outside of the boundary, the boundary is not real.

**The operational database is inside.** If an agent needs live data — an order status, a current balance — it does not connect to the database. The owning team exposes a tool that answers that question, and *that* tool talks to the database. The team keeps the ability to change schema, enforce row-level rules, apply rate limits, and see who asked.

**Agents compose with other agents through the same boundary.** A team agent that needs another domain's knowledge calls that domain's A2A agent or MCP tool. It does not get a read replica. This is what makes the mesh navigable — every edge in the graph is a governed edge.

---

## 3.4 Authentication: Identity Is Asserted by the Issuer, Never by the Caller

The most common weakness in early agent architectures is a caller that identifies itself in a field it controls.

```json
{ "user": "alice@corp.example", "query": "..." }
```

This is not identity. It is a claim, trivially forgeable, and any authorisation built on it is decoration. Yet it appears constantly, because in the prototype phase it works.

> **Identity must arrive as a token issued by a trusted identity provider, cryptographically verifiable by the receiver, and never assembled by the sender.**

In practice: OIDC tokens, validated on every request against the issuer's signing keys, checking issuer, audience, expiry and signature. The audience check matters more than it is given credit for — it is what stops a token minted for one service being replayed against another.

### Human identity is the easy half

The estate is mostly non-human. Four distinct cases, each with a different answer:

**A user, interacting live.** The token represents the person. Their entitlements apply directly. Straightforward, and the case everyone designs for.

**A workload acting on its own behalf.** A nightly reconciliation job, a batch enrichment process. There is no user. It must have its own identity — a workload identity issued by the platform, with its own entitlements and its own name in the audit log. Giving it a service account shared across five jobs destroys attribution for both security and cost.

**A workload acting on behalf of a user.** An agent answering Alice's question. Two identities matter: the agent's (may this agent call this tool at all?) and Alice's (which documents may *she* see?). Both must be present. Collapsing them into one is how an agent with broad service-level access ends up returning documents Alice was never entitled to — the single most common serious defect in these systems.

**An agent calling an agent on behalf of a user.** The delegation chain. Alice asks Team A's agent, which asks Team B's agent. Team B must be able to establish that Alice is the ultimate principal, that Team A is a legitimate intermediary, and that Alice actually authorised this path. Delegation chains, their verification and their depth limits are covered in [Securing MCP and A2A Interfaces](securing-mcp-and-a2a-interfaces.md).

**Never issue long-lived static credentials to workloads.** Short-lived tokens from platform-managed workload identity, automatically rotated. A static key in a configuration file is a breach with a delay on it.

---

## 3.5 Authorisation: Stop the Call Before Anything Is Retrieved

Authentication established who is calling. Authorisation decides what they may do — and *when* that decision is made is the whole ballgame.

### Tool-level authorisation comes first

An MCP server typically exposes several tools. A reasonable customer-domain server might offer:

```text
  search_customer_documents     read  · broad audience
  get_customer_profile          read  · restricted
  get_customer_pii              read  · tightly restricted
  update_customer_record        write · very few callers
```

> **A client authorised for one tool on a server is authorised for exactly that tool. Entitlement is per tool, never per server.**

This sounds obvious and is routinely violated, because the default implementation authorises the *connection*. Once a caller is on the server, every tool it advertises is callable. That is one over-broad grant away from a write tool being reachable by a reporting agent.

Two consequences worth stating explicitly:

**Tool discovery is filtered too.** When a caller lists available tools, it should see only the tools it may call. An unauthorised tool the caller can see is an invitation and an information leak — the existence and signature of `get_customer_pii` is itself a disclosure.

**The check runs before the handler.** Not inside it, not as an early return buried in application code. In the request pipeline, in one place, uniformly, for every tool.

### Then data-scope authorisation — still before retrieval

Being allowed to call `search_customer_documents` does not mean being allowed to search *all* customer documents.

The entitlement the identity carries — groups, roles, region, clearance, relationship to the record — is translated into a **retrieval filter**, and that filter is applied as part of the query itself.

```text
  PERMITTED

     identity ──► entitlements ──► filter ──► query ──► results
                                                        (already
                                                         lawful)

  FORBIDDEN

     identity ──► query ──► results ──► filter ──► fewer results
                             ▲
                             │
                      the system has now read data on
                      behalf of someone not entitled to it,
                      and the filter is the only thing
                      standing between that and disclosure
```

The second pattern fails in ways the first cannot. Result counts leak. Relevance scores leak. Latency leaks. A bug in the filter is a disclosure rather than an error. And in most jurisdictions the unauthorised *access* has already occurred, regardless of what was returned.

> **The rule: entitlement determines what is retrieved, not what is returned.**

A post-retrieval check is still worth having as a second line — defence in depth — but it can never be the first line.

### The honest constraint

This pattern demands that the identity's entitlements be expressible as index filters. Which in turn demands that entitlement metadata be attached at ingestion, which is why Chapter 2's pipeline treats classification as a required field and permission mode as a per-source configuration setting. **Exposure security is decided at ingestion time.** A corpus indexed without entitlement metadata cannot be safely exposed later without reprocessing.

---

## 3.6 Versioning: The Contract Has Consumers Now

The moment a second team depends on your interface, you have accepted a compatibility obligation. Teams that skip this discover it as an outage in somebody else's system.

**What is versioned:** tool names, argument schemas, response schemas, error semantics, and the *meaning* of results. The last is the one that bites — a tool that silently starts returning summaries instead of passages has broken its contract without changing a single type.

**What is not versioned:** anything inside the boundary. Index structure, chunking, embedding model, retrieval strategy, which tier you are on. Change freely.

**The rules:**

- Additive changes — new optional arguments, new response fields — need no new version. Consumers must tolerate unknown fields.
- Breaking changes — removing or renaming, tightening validation, changing meaning — require a new version served alongside the old.
- Two versions concurrently, minimum. **The deprecation window has to exceed the slowest consumer's release cycle** — that is the principle; the number is whatever that turns out to be in your organisation. A quarterly-release consumer needs more than a quarter, or you have handed them an impossible deadline and they will simply miss it. Six months is a common landing point for that reason, not a rule. Deprecation is surfaced in the response metadata, not only in a document, because the consumer who reads your documentation is not the consumer you are worried about.
- **You must know who your consumers are.** Which is a registry problem, and the reason [Chapter 10](../chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md) exists.

---

## 3.7 What the Platform Team Provides

Nothing in this chapter should be implemented eleven times by eleven teams. Authentication, authorisation and telemetry are exactly the concerns that must be identical everywhere or they are worthless.

| Platform provides | Team owns |
| --- | --- |
| MCP server and A2A agent boilerplate, with authN/authZ/telemetry pre-wired | Which tools to expose, and what they do |
| Token validation as middleware — issuer, audience, expiry, signature | Nothing; this must not be team code |
| The tool-entitlement model, policy schema, and filtered discovery | Granting entitlements over their own tools |
| Workload identity issuance, rotation, delegation-chain verification | Declaring which workloads they run |
| Entitlement-to-filter translation helpers | Mapping their domain's entitlement metadata |
| Contract linting and compatibility checks in CI | Keeping their contract compatible |
| Interface conformance tests, runnable by any team | Passing them before publishing |
| Registration into the estate registry | Keeping their entry accurate |

**The test of whether this is working:** exposing a RAG through a fully governed interface should be *less* work than exposing it insecurely. If the secure path is the slower path, teams will take the other one, and no policy document will stop them.

---

## Where this leaves you

The boundary is now specified rather than assumed. Identity arrives verifiably, entitlement is decided before a document is touched, and the interface can evolve without breaking its consumers.

None of which says anything about whether the answers are any good.

A perfectly secured interface returning subtly wrong answers is a perfectly secured liability — and in a mesh it is worse than in a monolith, because the wrong answer now travels. Team A's agent takes Team B's answer as fact, synthesises it with its own, and passes it upward. By the time a human sees it, the error has been laundered through three layers of confident prose and the citation trail, if there is one, points at a passage nobody checked.

So: how do you know it is right, and how do you find out where it went wrong?

---

**Supporting material for this chapter**

- [Securing MCP and A2A Interfaces](securing-mcp-and-a2a-interfaces.md) — token validation, tool entitlement policy, delegation chains, and the platform boilerplate

---

[← Chapter 2 — Building a RAG: The Paved Road](../chapter-2-building-a-rag-the-paved-road/chapter-2-building-a-rag-the-paved-road.md) | [Chapter 4 — Proving Retrieval Quality →](../chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md)
