# Choosing Between Ingest, Tool and Agent

> Supporting material for [Chapter 8 — Data You Cannot Move](chapter-8-data-you-cannot-move.md)
>
> The decision in detail: a scored comparison, the hybrid patterns that resolve most awkward cases, caching policy, and a worked answer for each of the six situations in Chapter 8.

---

## 1. The three options, side by side

| | **A — Ingest** | **B — MCP tool in place** | **C — Agent over MCP** |
| --- | --- | --- | --- |
| Where the data lives | Your index | The source | The source |
| Retrieval quality | Whatever you build — usually best | The source's own capability | The agent's, usually good |
| Semantic search | Yes | Only if the source offers it | Usually |
| Freshness | Your refresh cadence | Always current | Always current |
| Availability dependency | None at query time | The source must be up | Source and agent must be up |
| Query latency | Lowest | Source latency | Source + generation latency |
| Ongoing cost | Storage + embedding + refresh | Per query, often metered | Per query + generation |
| Permission model | Yours, mapped at ingestion | The source's, per call | The agent's |
| Licence exposure | High — you hold a copy | Low | Low |
| Deletion burden | Yours to propagate | None | None |
| Domain knowledge required | Yours | Yours | The agent owner's |
| Integration effort | High | Medium | Low |
| Who maintains it | You | You | Someone else |

**The pattern in that table:** Option A buys retrieval quality and control at the price of ownership burden and legal exposure. Option C buys convenience and offloads domain knowledge at the price of control. Option B sits between, and is the most under-used of the three.

---

## 2. Disqualifying conditions

Work through these first. They eliminate options outright, and eliminating options is faster than comparing them.

**Ingest is ruled out if:**
- The licence or contract prohibits extraction or retention
- The source cannot be exported in bulk at all
- The data changes faster than any honest refresh cadence
- Copying it would create a second authoritative copy of data another team owns
- The volume is grossly disproportionate to the slice you need

**In-place (B or C) is ruled out if:**
- The source cannot meet your latency budget, even cached
- The source's availability is below what your use case tolerates, with no acceptable degradation
- Per-query cost at your volume exceeds the cost of holding a copy
- The source offers no query interface worth using and none can be built
- Your workload must function when disconnected from the source

**Agent (C) is ruled out if:**
- No agent exists and nobody will own building one
- You need the raw passages rather than a synthesised answer
- The additional generation latency is unacceptable

If exactly one option survives, you are finished. If several do, score them.

---

## 3. Scoring when several options survive

Weight by what actually matters for the use case, then score each surviving option 1–5.

| Criterion | Ask |
| --- | --- |
| Answer quality | Which produces the best answers for *these* questions? |
| Freshness need | How wrong is a day-old answer? |
| Latency budget | What will users tolerate? |
| Total cost over three years | Including your engineering time, not only infrastructure |
| Operational burden | Who is on call for this, and do they know? |
| Legal exposure | What happens if the relationship or terms change? |
| Reversibility | How hard is it to change this decision in two years? |

**Weight reversibility higher than instinct suggests.** Option B is the most reversible — you are holding nothing, so switching to A later is a build, not a migration. Option A is the least reversible, because you accumulate a corpus, consumers, and habits around it.

**Record the decision and its reasons.** Not ceremony: in two years someone will ask why the vendor system was not ingested, and "the licence prohibited it in 2026" is the answer that prevents the question being reopened annually. Record it in the registry entry ([Chapter 10](../chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md)).

---

## 4. Hybrid patterns

Most genuinely awkward cases are solved by a combination rather than by picking one option harder.

### 4.1 Metadata indexed, content in place ★

The most useful pattern in this document, and the usual answer to "the vendor system has terrible search."

```text
  YOUR INDEX                          THE SOURCE
  ──────────                          ──────────
  title                               full document content
  summary / abstract                  (never copied)
  identifiers, references
  classification, dates
  entities mentioned
  source_document_id  ─────────────►  fetched on demand

  1. semantic search over metadata → candidate documents
  2. fetch those documents' content through the tool
  3. generate from the fetched content
```

You get semantic discovery without holding the content. Licence exposure is usually acceptable, because metadata is rarely the licensed asset — but **check, because sometimes it is.**

Works well when documents have meaningful titles and summaries. Works poorly when the answer is buried in the body of a document whose metadata gives no hint of it.

### 4.2 Hot ingested, cold in place

Ingest the recent or high-traffic slice; reach the historical archive in place.

```text
  last 24 months    ──► ingested, full semantic search
  older than that   ──► tool, queried on demand
```

Suits corpora with a steep access-recency curve, which is most of them. The ingestion cost falls sharply and the great majority of questions are served from the fast path.

### 4.3 Agent in front of a mixed back end

You expose one A2A agent. Behind it, your own index plus external tools. Consumers see one interface; you manage the complexity once rather than exporting it to every consumer.

This is the natural end state for a team with several sources, and it is what Chapter 8's boundary diagram depicts.

### 4.4 Cached in place

Option B with a result cache in front.

```text
  query ──► cache ──hit──► return
              │
             miss
              ▼
          source ──► cache ──► return
```

**Cache TTL is a correctness decision, not a cost decision.** Set it from how wrong a stale answer would be, then accept the cost consequence. Setting it from the cost side is how an in-place source quietly becomes a stale one — and you lose the main reason you chose B.

**Cache keys must include the entitlement context.** A cache shared across principals is a disclosure. This is worth stating twice because it is the most common serious bug in caching layers.

### 4.5 Ingest a derived view

Where the raw data cannot be copied but a derived form can: aggregates, summaries, non-identifying extracts, or a reduced projection.

Often permitted where full extraction is not, and often sufficient. **Check the licence specifically** — some prohibit derived works too, and the distinction between "derived work" and "index" is not always favourable.

---

## 5. Worked examples

### Vendor document management system
Twenty years of contracts. Search API, no bulk export, replication prohibited.

**Disqualified:** ingest, contractually.
**Chosen:** hybrid 4.1 — index contract metadata (counterparty, dates, type, value, status), fetch clause text in place.
**Why:** the vendor's keyword search cannot answer "which contracts have unusual indemnity terms," but semantic search over metadata narrows to a handful, and those are then fetched in full.
**Accepted:** vendor latency on fetch; a fallback message when the vendor is unavailable.

### Licensed regulatory database
Subscription permits query, not extraction. Updated continuously.

**Disqualified:** ingest, by licence and by volatility.
**Chosen:** Option C — the compliance team owns the relationship and builds the agent.
**Why:** using this source correctly requires knowing which guidance supersedes which and which jurisdiction applies. That knowledge belongs with compliance. Six teams needing it should consume compliance's agent, not each reimplement its expertise — and get it subtly wrong in six different ways.
**Accepted:** latency; a per-query licence cost attributed to the calling team.

### Partner product catalogue
Changes hourly. API available, permission granted.

**Disqualified:** ingest, by volatility.
**Chosen:** Option B with a fifteen-minute cache.
**Why:** prices and availability must be current; a nightly snapshot would be wrong all day. Fifteen minutes is the agreed tolerance.
**Accepted:** availability dependency; degraded answers stating that live catalogue data is temporarily unavailable — never silently omitted.

### Mainframe records
Forty years of data. Extractable in principle; no owner will approve it.

**Chosen:** Option B, with a strictly rate-limited read-only tool.
**Why:** the organisational objection is to bulk extraction and load on the system, not to occasional targeted reads. A narrow tool answering specific questions is approvable where a nightly export is not.
**Accepted:** poor search capability; the tool exposes a few specific lookups rather than general search. This is a real limitation and it is honestly better than nothing.

### Another team's corpus
You need customer master knowledge. Customer Master owns it.

**Chosen:** Option C, always.
**Why:** this is Chapter 1's ownership rule. Copying creates the drift, the second permission model and the orphaned deletion path that the whole book exists to prevent. Customer Master exposes an agent; you consume it.
**Accepted:** their latency and their availability. In exchange, their corrections reach you automatically and you never own their data quality.

### Enormous source, small slice
A data lake of which you need one narrow subject area.

**Chosen:** ingest — but only the slice.
**Why:** the disqualifier was proportionality, not permission. A filtered ingestion of the relevant subject area is Option A at an appropriate scale.
**Accepted:** the filter must be maintained as the lake evolves, or the slice silently stops matching what you need.

---

## 6. Revisiting the decision

None of these are permanent, and the conditions change.

**Review when:**
- Licence or contract terms change
- The source adds or removes export capability
- Query volume grows enough to change the economics
- Availability or latency stops meeting your needs
- Another team builds an agent over the same source — **switch to it**
- Your freshness requirement changes

**A standing review annually, per external source**, in the registry. Cheap, and it catches the decisions that were correct in 2026 and are quietly wrong by 2028.

**The interface contract is what makes this safe.** Because consumers reach you through the governed boundary from [Chapter 3](../chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md), you can move a source from B to A, or from A to C, and nobody outside notices. That is worth more than getting the decision right the first time.

---

[← Back to Chapter 8](chapter-8-data-you-cannot-move.md)
