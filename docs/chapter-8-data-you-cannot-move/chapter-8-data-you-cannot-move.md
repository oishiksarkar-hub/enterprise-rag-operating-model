# Chapter 8 — Data You Cannot Move

> **The bridge.** Every chapter so far assumed the data was yours and could come into your boundary. Frequently it cannot — it belongs to a vendor, a partner, another team, or a system nobody will let you copy.
>
> **This chapter covers** external and third-party sources, data that must stay where it is, and consuming another team's knowledge without copying it.
>
> **What it is not about.** If the knowledge you need belongs to another team inside your organisation, the answer is already settled: that team owns it, exposes it behind the governed interface in [Chapter 3](../chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md), and you call it. No chapter is needed for that. This chapter is for the harder case — where there is no team to ask, because the owner is a vendor, a licence, a mainframe or a system that will never expose anything, and where copying is prohibited, impossible or wrong.

---

## 8.1 The Problem: The Knowledge Is Real and It Is Not Yours

A team building a RAG discovers that a meaningful part of what it needs sits outside its reach.

**A vendor platform.** The document management system holds twenty years of contracts. The vendor offers a search API and no bulk export. Replication is contractually prohibited.

**Licensed data.** The regulatory or market-data subscription permits query, not extraction. Ingesting it would breach the licence, and the licence is not negotiable.

**Volatile external data.** A partner catalogue changing hourly. Any snapshot is wrong before it finishes indexing.

**Legacy systems.** Forty years of records on a mainframe. Technically extractable, organisationally impossible — no owner will sign off, and the risk of an extraction job on that system is not worth the benefit.

**Another team's corpus.** Chapter 1 was explicit: you must not copy it. So how do you use it?

**Scale and economics.** A source so large that ingesting it is disproportionate to the value of the small slice you need.

The instinctive response is to copy anyway, because it is the pattern everyone knows. That produces the failures Chapter 1 set out to prevent — a second copy drifting from the first, a second set of permissions, deletion requests reaching one and not the other — plus new ones: licence breach, contractual exposure, and a snapshot that misrepresents a live system.

> **The question is not "how do I get this data in?" It is "should this data come in at all, and if not, what is the right way to reach it?"**

---

## 8.2 The Principle: Choose Deliberately Between Three Options

The whole chapter is one decision, made once per source, on the evidence.

> **For every source, choose deliberately: ingest it, reach it in place through a tool, or consume a reasoning interface over it. All three are legitimate. Choosing by default is not.**

```text
   THE SOURCE
        │
        ▼
   ┌────────────────────────────────────────────────┐
   │  Can it be copied, legally and practically?    │
   └────────────────────────────────────────────────┘
        │ no                              │ yes
        ▼                                 ▼
   ┌─────────────────────┐      ┌──────────────────────────┐
   │ Does someone own    │      │ Is it stable enough that │
   │ the domain logic    │      │ a periodic snapshot is   │
   │ over it?            │      │ honest?                  │
   └─────────────────────┘      └──────────────────────────┘
      │ yes        │ no            │ yes            │ no
      ▼            ▼               ▼                ▼
  ┌────────┐  ┌────────┐      ┌────────┐      ┌────────────┐
  │   C    │  │   B    │      │   A    │      │  B or C    │
  │ AGENT  │  │  MCP   │      │ INGEST │      │  in place  │
  │ over   │  │  tool  │      │        │      │            │
  │ MCP    │  │in place│      │        │      │            │
  └────────┘  └────────┘      └────────┘      └────────────┘
```

### Option A — Ingest

Bring the content in, index it, own it. Everything in Chapters 2 through 7 applies unchanged.

**Choose when:** you may copy it, it is stable enough that a snapshot is honest, and you need semantic search over its full content.

**You accept:** refresh responsibility, storage and embedding cost, permission mapping, and deletion propagation back to the source's rules.

### Option B — MCP tool in place

The data stays where it is. You expose — or consume — a tool that queries it live and returns content to the model.

**Choose when:** copying is prohibited or impractical, the data is volatile, or the source already has good query capability.

**You accept:** dependence on the source's availability and latency, the source's own search quality (which may be poor), and the fact that you cannot do semantic retrieval over content you have never embedded.

**What you gain:** always current, no licence exposure, no refresh cost, no second permission model, and deletion is automatic because there is nothing to delete.

**Note what this is.** Retrieval still happens — it happens *at the source*, and the results still land in the model's context. This is RAG. It is retrieval-in-place rather than retrieval-from-your-index, and calling it "not real RAG" causes teams to dismiss the best available option for the wrong reason.

**Its real limitation** is retrieval quality. If the source offers only keyword search, that is what you get. A mitigation worth knowing: index the *metadata* — titles, summaries, identifiers, classifications — semantically in your own index, use that to identify candidate documents, then fetch their content through the tool. Semantic discovery, in-place content.

### Option C — Agent over MCP

Someone builds an agent that owns the domain reasoning over the source. You consume the agent through A2A.

**Choose when:** using the source well requires knowledge you do not have — which sources supersede which, how the entities relate, what the caveats are — or when several consumers would otherwise each reimplement that knowledge.

**You accept:** the agent's cost and latency, and less control over how the answer was formed.

**What you gain:** the domain knowledge stays with whoever has it. This is Chapter 1's ownership rule applied to data you do not own — and it is the *default* answer for consuming another team's knowledge.

**Who builds it?** Whoever owns the relationship with the source. For a vendor platform, the team that owns that vendor relationship. For another team's corpus, that team. Not you. If nobody owns it and several teams need it, that is a gap the platform team should name and fill, and it is exactly the kind of thing the registry in [Chapter 10](../chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md) makes visible.

### Frequently: a combination

A mature system mixes all three. Your own documents ingested. The vendor contract system reached by tool. The regulatory interpretation obtained from the compliance team's agent. One answer, three access patterns, one governed interface out the front.

> **Reference implementation — the systems this actually means**
>
> The abstractions above are deliberate, but in practice this chapter is usually about four or five named systems. Concretely:
>
> | Source | Usual answer | The thing that decides it |
> | --- | --- | --- |
> | **SharePoint / OneDrive** | **A — ingest**, via a first-party connector | Every major managed search product ships a SharePoint connector with permission-aware sync. Build one only if yours does not. |
> | **Confluence** | **A — ingest** | Same. Watch page-restriction inheritance, which is where the ACL mapping goes wrong. |
> | **Google Drive / Workspace** | **A — ingest** | Connector available; sharing-link permissions need an explicit decision. |
> | **Slack / Teams** | **A**, selectively | Channel-scoped. Ingest decision channels, not social ones — and check your records-retention position first. |
> | **ServiceNow / Jira** | **B — tool in place** | Volatile, and the useful questions are structured queries rather than semantic ones. |
> | **Salesforce / CRM** | **B — tool in place** | Record-level security is intricate and best left to the source. |
> | **A vendor DMS with no bulk export** | **B**, or the metadata hybrid | Contract, not technology. |
> | **Mainframe / legacy core** | **B**, narrow read-only tools | Nobody will approve a bulk extract. |
>
> **The ACL mapping is the hard part, and it is the same problem in every one of them.** SharePoint has site, library, folder, item and unique-permission inheritance, plus sharing links that grant access outside any group. Confluence has space permissions, page restrictions, and inherited restrictions that are not visible on the page itself. Google Drive has "anyone with the link". Each of these is a route by which a document ends up retrievable by someone the source would never have shown it to.
>
> Three rules, whichever system it is:
>
> 1. **Capture the effective permission, not the declared one.** Ask the source who can actually read this item, rather than reconstructing it from group memberships yourself. Reconstruction is where the bugs are.
> 2. **Re-sync permissions on their own cadence.** Content changes and access changes are separate events. A nightly content sync that only refreshes ACLs when a document changes will serve revoked content indefinitely.
> 3. **Anything you cannot map, you do not index.** Default deny, per Chapter 3. A document whose permissions could not be resolved is quarantined, not published as internal.
>
> **Where the ACL model cannot be honoured at all, the source becomes Option B.** That is not a failure — it is the decision framework working. Reaching it in place means the source enforces its own access control, which is the outcome you were trying to reproduce anyway.
>
> *The same reasoning applies to any equivalent platform; the names above are the ones that come up most often, not a recommendation.*

A deeper treatment — including the metadata-index hybrid, caching policy, and a worked decision for each of the six situations in §8.1 — is in [Choosing Between Ingest, Tool and Agent](choosing-ingest-tool-or-agent.md).

---

## 8.3 The Pattern: The Boundary Does Not Change

Whichever option is chosen, the consumer sees the same thing. **This is the property that makes the choice safe to revisit later.**

```text
   Consumer
      │
      ▼
   ┌──────────────────────────────────────────────┐
   │   YOUR GOVERNED INTERFACE                    │
   │   authenticate · authorise · scope · trace   │
   └──────────────────────────────────────────────┘
      │
      ├──► own index                 (A · ingested)
      ├──► vendor MCP tool           (B · in place)
      ├──► partner MCP tool          (B · in place)
      └──► compliance team's agent   (C · reasoning)

   The consumer cannot tell which. It must not need to.
```

But three things do change behind the boundary, and each needs a deliberate answer.

**Entitlement across the boundary.** With ingested content you filter against your own metadata. With an external source, *whose* permissions apply? The answer must be both: your interface authorises the call, and the external call carries an identity the source can authorise against. **The failure to avoid is calling an external system under a single shared service account** — everyone who reaches your interface then has the access of that account, and the source's own access control is neutralised.

**Availability.** An in-place source is now in your critical path. It will be down at some point. Decide in advance: degrade with a partial answer that says what is missing, serve a bounded cache, or fail honestly. **Never silently omit a source** — an answer missing a third of its evidence, presented as complete, is worse than an error.

**Trust tier.** External content is untrusted by default. Chapter 6's restricted execution mode applies, and it applies to content arriving through a tool exactly as it does to ingested content. The ingestion-time sanitisation, however, does not — there was no ingestion. **Screening must happen at the point of retrieval instead**, which is the main security cost of reaching data in place.

---

## 8.4 Governance for Sources You Do Not Control

The concerns from earlier chapters do not disappear when the data is external. They get harder, and they need explicit answers.

**Security.** Credentials to external systems are managed secrets with rotation, never configuration files. Per-user or per-purpose identity where the source supports it. Outbound egress controls and allow-listing — an MCP tool reaching an external endpoint is a data-exfiltration path if it is not constrained. Retrieved external content is screened before it enters a model context.

**Observability.** External calls appear in the trace as spans like any other hop, carrying source, latency, result count and errors. **Set the expectation early that external latency dominates** — a vendor API at two seconds makes your sub-second retrieval irrelevant, and the trace is what tells you that before a user does.

**Cost.** External sources often bill per query. That cost is real, attributable to the originating identity, and belongs in Chapter 5's statements alongside token spend. A team whose agent calls a metered vendor API on every message should see that line. Caching is the primary lever, and its TTL is a correctness decision, not a cost decision.

**Licensing and contract.** Before anything is built: does the licence permit this use? Are there query volume limits? May the results be retained, cached, or shown to end users? Does an AI-mediated use case fall within the terms at all? **A licence breach discovered after deployment is a legal problem, not an engineering one**, and it is the most likely way this chapter goes wrong.

**Data flow.** Sending a query to an external system sends data outward. If the query contains personal or confidential information, you have made a cross-boundary transfer. Chapter 6's classification and residency rules apply to the query, not only to the results — and this is the direction teams forget.

**Boilerplate.** None of this should be reinvented per source. The platform ships a connector framework with credentials, retries, timeouts, circuit breaking, caching, tracing, cost metering and content screening already wired. A team integrating a new external source writes the source-specific call and gets the rest.

---

## 8.5 What the Platform Team Provides

| Platform provides | Team owns |
| --- | --- |
| The decision framework and its documentation | Making the choice, and recording why |
| Connector framework — credentials, retries, circuit breaking, caching, tracing, metering | The source-specific integration |
| A catalogue of supported connectors for common platforms, with their ACL mapping already solved and tested | Requesting a new one rather than building it |
| Secret management and rotation for external credentials | Declaring which sources they use |
| Egress controls and endpoint allow-listing | Requesting additions |
| Content screening at retrieval for in-place sources | Setting the trust tier |
| Availability patterns — degradation, cached fallback | Choosing their degradation behaviour |
| External query cost metering into showback | Managing their consumption |
| Registry entries for shared external agents and tools | Registering what they expose |
| A licence and contract review checklist | Obtaining approval before building |

**One platform observation worth acting on.** When three teams independently integrate the same vendor, that is a signal, not a coincidence. The platform should notice — the registry makes it visible — and promote it to a single shared Option C agent. One integration, one credential, one cost line, one team maintaining it, and the domain knowledge accumulating in one place rather than diverging in three.

---

## Where this leaves you

The estate can now reach knowledge it does not own, without pretending to own it.

All eight chapters have assumed something else, though, and it has been assumed so consistently that it has barely been visible: **that there is a cloud.**

Managed search services. Serverless platforms. Hosted model endpoints. Every reference implementation has named one of three providers.

For a great many organisations that assumption does not hold. A defence agency. A central bank. A hospital network under data-protection rules that exclude external processing. A manufacturer whose plant systems sit on an isolated network by design. A government running its own sovereign cloud.

These organisations have the same problem — vast legacy knowledge, scattered across systems, that people need answers from. They are not entitled to give up on it.

What changes when there is no cloud at all?

---

**Supporting material for this chapter**

- [Choosing Between Ingest, Tool and Agent](choosing-ingest-tool-or-agent.md) — the decision in detail, hybrid patterns, caching, and worked examples

---

[← Chapter 7 — Keeping Knowledge Fresh](../chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md) | [Chapter 9 — On-Premises and Air-Gapped →](../chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md)
