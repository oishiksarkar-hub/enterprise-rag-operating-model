# Chapter 7 — Keeping Knowledge Fresh

> **The bridge.** Chapter 6 made the estate defensible at a moment in time. Corpora do not hold still. A RAG that was correct in March is fluently, confidently wrong in September, and nothing about it looks broken.
>
> **This chapter covers** refresh economics, retiring content, freshness commitments and how to measure them, detecting silent decay, healing without a human, and incident response.

---

## 7.1 The Problem: Wrong Answers That Look Exactly Like Right Ones

Staleness is the most dangerous failure mode in this book, because it is the only one with no symptoms.

An outage is obvious. A security breach eventually surfaces. A cost overrun appears on an invoice. A stale RAG produces a well-formed, well-cited, confident answer — sourced from a document that was superseded four months ago. Latency is normal. Error rate is zero. Faithfulness scores are excellent, because the answer *is* faithful to what was retrieved. The evaluation set from March still passes, because it was written in March.

And the more successful the RAG has been, the more thoroughly the wrong answer is believed.

**Three distinct ways it happens:**

**The corpus drifts.** Documents change and the index does not. The refresh job was failing silently, or the schedule was set for a rate of change that no longer applies.

**Content is superseded but not withdrawn.** The new policy is indexed. So is the old one. Both retrieve. The model picks — and old policies are often *more* retrievable, because they have accumulated more supporting material and cross-references.

**Content is deleted at source and survives in the index.** Deletion detection is the hardest part of incremental sync and the most commonly missing.

### The tension that makes this hard

> **Refreshing a large corpus costs real money. Not refreshing it produces wrong answers. Neither "refresh everything constantly" nor "refresh when someone complains" is a strategy.**

Full reprocessing of a substantial corpus means extraction, chunking and embedding across everything — a meaningful, recurring bill. So teams stretch the schedule. Nobody notices, because staleness has no symptoms. Until a customer is told something untrue.

The way out is to stop treating freshness as uniform. **Different content decays at different rates, and refresh effort should follow the decay rate, not the calendar.**

---

## 7.2 The Principle: Freshness Is a Declared, Measured Property

> **Every corpus declares a freshness requirement. The system measures freshness against it, detects decay before users do, and retires content deliberately rather than leaving it to accumulate.**

Three consequences.

**Freshness is a per-source SLO, not a global schedule.** A pricing table and a company history do not need the same cadence.

**Decay must be detected actively.** Nothing about a stale index raises an alarm on its own. Detection has to be built.

**Retirement is a first-class operation.** An index that only ever grows will eventually contain more obsolete material than current, and the obsolete material competes for retrieval.

### Freshness tiers

| Tier | Refresh | Suits | Staleness consequence |
| --- | --- | --- | --- |
| **Live** | Not indexed — queried at source | Prices, balances, stock, status | Immediate and material |
| **Near-real-time** | Event-driven, minutes | Active tickets, current incidents | Serious |
| **Daily** | Scheduled, nightly | Operational procedures, product docs | Noticeable |
| **Weekly** | Scheduled | Policies, standards, guidance | Moderate |
| **Slow** | Monthly or on change | Reference material, historical record | Minor |

**The Live tier deserves emphasis.** Some data should never be indexed at all. An account balance, an order status, current inventory — the correct pattern is an MCP tool querying the source system at request time, not a nightly snapshot. **Indexing volatile data is a design error that manifests as a freshness problem**, and no refresh cadence fixes it. Chapter 3's pattern already provides for this: the operational database sits behind a tool, inside the boundary.

**A change of tier is an evaluation trigger.** Moving content between tiers, or changing a tier's cadence, changes what retrieval returns and when. Re-run the evaluation set from [Chapter 4](../chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) against the change rather than assuming freshness work is quality-neutral — it very often is not.

Assign the tier per source, not per corpus. One RAG will span several.

---

## 7.3 The Pattern: Event-Driven Where It Matters, Scheduled Where It Does Not

```text
  SOURCE CHANGES
       │
       ├── emits an event  ──────────────┐   preferred
       │                                 │
       └── no event capability           │
                │                        │
                ▼                        │
         scheduled poll                  │
         with change detection           │
                │                        │
                ▼                        ▼
         ┌──────────────────────────────────┐
         │  CHANGE ASSESSMENT               │
         │  hash compare — did it really    │
         │  change, or just get touched?    │
         └──────────────────────────────────┘
                │
        ┌───────┼────────┬──────────────┐
        ▼       ▼        ▼              ▼
     created  updated  deleted     unchanged
        │       │        │              │
        │       │        │              └─ do nothing
        │       │        │                 (the common case,
        │       │        │                  and the saving)
        ▼       ▼        ▼
      index   re-chunk  tombstone
              re-embed  then purge
              replace   on policy
```

**Change assessment is where the money is saved.** A document whose modification timestamp moved but whose content hash did not needs no work. In most corpora this is the overwhelming majority of apparent changes, and skipping them turns an expensive nightly job into a cheap one.

**Event-driven beats scheduled wherever the source supports it.** Fresher, and dramatically cheaper — no polling, and work proportional to actual change rather than to corpus size.

**Deletion propagation is the part that gets missed.** A deleted document must be removed from the index within a stated SLA. If your source integration cannot detect deletions, you need a periodic reconciliation pass — enumerate the source, compare against the index, remove orphans. It is expensive, and it is the only alternative.

### Tombstone or purge

Two operations, and choosing wrongly is a compliance failure in one direction or the other.

**Tombstone** — the content stays, marked as withdrawn or superseded:

```text
  status: superseded
  superseded_by: pol-77
  superseded_on: 2026-04-14
  retrievable: false        ← excluded from normal retrieval
  auditable: true           ← available for historical queries
```

The right answer for decommissioned policy. You need to know what the rule was in March, because someone will ask, and an auditor certainly will. **The evolution of a policy is itself organisational knowledge**, and purging it destroys the ability to answer "what were we told at the time?"

**Purge** — the content is destroyed, everywhere, with evidence. The right answer for a right-to-be-forgotten request, or when retention expires.

**The default should be tombstone, with purge on explicit trigger** — a deletion request, a retention expiry, or a legal instruction. Tombstoning by default preserves the historical record; purging by default destroys it irrecoverably, and nobody notices until it is needed.

**Supersession chains matter.** When `pol-08` is superseded by `pol-77`, record it. Then a query that would have matched the old document can return the new one with a note, rather than nothing at all — which is a much better experience than silence, and it is only possible if the link was recorded at retirement time.

### TTL as a backstop

Some content should expire on age alone, whatever the source says:

```yaml
ttl:
  - source: incident-reports
    max_age_days: 730
    action: tombstone
  - source: quarterly-briefings
    max_age_days: 365
    action: tombstone
  - source: temporary-notices
    max_age_days: 90
    action: purge
```

TTL is a safety net against the source system that never marks anything obsolete — which is most source systems.

---

## 7.4 Detecting Decay Before Users Do

Since staleness has no symptoms, symptoms must be manufactured.

### Canary queries

A set of questions with known-correct answers, run continuously against the live system, with results compared to expectations.

```text
  every 15 minutes
        │
        ▼
  run the canary set against production
        │
        ▼
  compare to expected answers
        │
   ┌────┴─────┬──────────────┬─────────────┐
   ▼          ▼              ▼             ▼
  match    wrong doc      no result     changed
           retrieved                    answer
   │          │              │             │
   ok      ALERT          ALERT        INVESTIGATE
           supersession   deletion or  content changed —
           or ranking     index gap    expected, or decay?
           failure
```

**The canary set is not the evaluation set.** The evaluation set measures quality against a baseline, offline, at gate time. Canaries run continuously against production and answer a narrower question: *is this system still returning what it returned yesterday, and if not, why?*

**Good canaries** cover the corpus's most-asked questions, its highest-consequence questions, at least one question per major source, and known edge cases. **Twenty is plenty.** They run constantly, so they cost money constantly — and their frequency should track the corpus's rate of change rather than being uniform.

**A changed canary answer is not automatically a failure.** Content legitimately changes. The alert says *investigate*, and the outcome is either "correct — update the expected answer" or "decay — fix it." Both outcomes are useful; only the second is an incident.

### Freshness metrics

Alongside canaries, measure the property directly:

| Metric | Watch for |
| --- | --- |
| Index age per source — oldest unrefreshed document | A source whose refresh has silently stopped |
| Refresh success rate | Jobs failing without anyone noticing |
| Refresh lag against SLO | Cadence no longer matching rate of change |
| Stale-document retrieval rate | Documents past their freshness SLO being served |
| Superseded-document retrieval rate | Retirement not working |
| Deletion propagation time | Against the Chapter 6 SLA |
| Orphan count from reconciliation | Deletion detection failing |

**Stale-document retrieval rate is the single most useful number here.** It measures the thing that actually harms users — not whether the index is old, but whether old content is reaching answers. Chapter 4's retrieval spans already carry the freshness flags that produce it.

---

## 7.5 Healing Without a Human

Detection produces alerts. Alerts produce tickets. Tickets produce a queue that grows faster than anyone works through it, and the freshness programme quietly dies of backlog.

**Most decay is mechanical, repetitive and has exactly one correct response.** A refresh job that failed on a transient error should be retried, not triaged. A document whose extraction confidence collapsed after a source-format change should be re-extracted, not discussed. The only thing standing between detection and repair is a decision nobody needs to make.

> **If the response to a detected fault is always the same, it is not an alert. It is an unimplemented automation.**

### What can be healed automatically

| Fault | Automatic response | Escalate if |
| --- | --- | --- |
| Refresh job failed, transient error | Retry with backoff | Three consecutive failures |
| Refresh job failed, auth error | Rotate credential, retry once | Rotation fails |
| Source unreachable | Retry on a decaying schedule; mark the source's freshness degraded in the registry | Unreachable beyond the source's SLO |
| Extraction confidence below threshold | Re-extract with fallback settings — OCR on, alternative parser | Still below threshold |
| Document in the index, absent at source | Tombstone it | The orphan rate exceeds a floor — suggests a broken integration, not real deletions |
| Chunk count for a document changed by an implausible margin | Quarantine the new version, keep the old, raise it for review | Always — this one is a review, not a repair |
| Embedding model version drift detected between chunks | Queue the affected chunks for re-embedding at batch rates | The affected proportion is large enough to warrant a full rebuild |
| Canary returns the wrong document | Re-run to confirm, then check whether a superseding document exists; if so, tombstone the old one | No supersession found |
| Canary returns nothing | Re-run, then verify the document is still in the index; re-ingest it if not | Re-ingestion does not restore it |
| Index corruption or partial build | Roll back to the retained previous index and rebuild | Rebuild also fails |
| Freshness SLO breached for a source | Increase that source's refresh cadence one tier, with a cost note | The SLO is still breached at the higher tier |

**The right-hand column is the important one.** Automation without an escalation boundary becomes a system that repeatedly and invisibly fails to fix itself. Every healing action has a retry budget, and exhausting it produces a human alert carrying the full record of what was already attempted.

### The control loop

```text
   DETECT              DIAGNOSE            ACT              VERIFY
   canary          ─► classify the    ─► known remedy  ─► re-run the
   freshness metric   fault against      applied,          detecting
   job telemetry      a known set        recorded          check
   reconciliation          │                  │                │
                           │                  │           ┌────┴────┐
                           │                  │           ▼         ▼
                           │                  │        healed    still
                           ▼                  ▼           │      failing
                      unknown fault      retry budget     │         │
                           │              exhausted       │         ▼
                           └──────┬────────────┘          │    ESCALATE
                                  ▼                       ▼    with full
                             HUMAN ALERT             log + close   history
                             with full context
```

**Every automatic action is recorded as an event on the corpus**, visible in the registry and in the freshness dashboard. A source that healed itself forty times last month is not healthy — it is failing forty times and hiding it. **The self-healing rate is itself a metric**, and a rising one is a defect report against the source integration.

### Two rules that keep this safe

**Heal forward, never destructively.** Automatic tombstoning is acceptable because it is reversible. Automatic purging is not, ever. An automated system that can permanently destroy content will eventually do so on the strength of a bad signal — and the retention story you tell an auditor afterwards is not one you want to have to tell.

**Never auto-heal quality.** A canary returning a *different but plausible* answer is exactly the case where the content may legitimately have changed. Automation confirms and escalates; it does not decide. **Automate the mechanical, escalate the semantic** — the boundary between them is the whole design.

### Where this fits

Chapter 6's kill switch and Chapter 10's orphan detection are the same idea at different timescales: detect a condition, apply a known response, escalate when it does not hold. The platform should ship one remediation engine, not three — and the actions above should be configuration in it, not code in each team's pipeline.

---

## 7.6 When It Goes Wrong: Incident Response

At some point the assistant tells a customer something untrue, and someone senior asks what is being done about it. Having a runbook beforehand is the difference between an hour and a fortnight.

### The runbook

**1. Capture the trace id.** Before anything else. Without it, everything below is guesswork.

**2. Assess reach.** Query traces for the same question pattern or the same retrieved documents. How many users, over what period? **This number determines the response, and it is unavailable to organisations that did not build Chapter 4.**

**3. Stop the bleeding.** In increasing order of disruption:
   - **Tombstone the offending document** — targeted, fast, usually sufficient
   - **Add a retrieval filter** excluding the affected content
   - **Roll back the index** to the previous version — which requires having kept it, per Chapter 2's dual-index pattern
   - **Disable the affected tool or agent** — narrow scope
   - **Kill switch** — the RAG stops answering and says so

**4. Diagnose** using Chapter 4's investigation runbook. Stale, superseded, wrong at source, conflict, or generation failure — these have different owners and different fixes.

**5. Communicate.** Who received the wrong answer, and do they need to be told? For a customer-facing error this is not an engineering decision.

**6. Fix at source.** If the document is wrong, the document gets fixed. Do not distort retrieval to work around bad content.

**7. Add a canary and an evaluation case.** Non-negotiable. An incident that does not become a test will recur.

**8. Ask why detection failed.** The most valuable output of the whole exercise. Why did a user find this before the system did? Usually the answer is a missing canary, a freshness tier set too slow, or a refresh job that had been failing for weeks.

### Capabilities this assumes

**A kill switch, per RAG, tested.** An untested kill switch is a hope. It must fail into an honest message, not a blank response or a timeout.

**Index rollback.** Retain the previous index version for a defined window after every rebuild. Chapter 2's dual-index cutover already produces this; the requirement is simply not to delete it too eagerly.

**Named ownership with out-of-hours cover.** Every RAG in the registry ([Chapter 10](../chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md)) has a named owner and an escalation path. **An orphaned RAG in an incident is the worst position to be in** — it is answering, it is wrong, and nobody has the authority or the knowledge to stop it.

**A user-visible trace reference.** If users cannot tell you which answer was wrong, you cannot investigate it.

---

## 7.7 The Economics, Made Explicit

The tension from §7.1, resolved into a set of decisions.

**Refresh in proportion to decay, not on a uniform schedule.** Weekly for policy, daily for operational content, event-driven for volatile content, never for content that should have been a live tool call. This alone typically halves refresh cost while *improving* freshness where it matters.

**Change detection before reprocessing.** Hash comparison is nearly free; re-embedding is not. Skipping unchanged documents is the largest single saving in the refresh budget.

**Event-driven wherever the source allows.** Work becomes proportional to change rather than to corpus size — a completely different cost curve as corpora grow.

**Retire aggressively.** Smaller indexes cost less to store, search faster, and return better results. Retirement is the rare lever that improves cost and quality simultaneously.

**Batch the periodic work.** Reconciliation passes and bulk re-embedding are not latency-sensitive.

**Budget re-embedding as a scheduled event.** Every year or two, forecastable, planned — not a surprise.

**And spend deliberately on canaries.** They cost money continuously and they are the only thing standing between a silent decay and a customer incident. Frequency by tier: often for volatile corpora, rarely for stable ones.

**Automatic healing is cheaper than the queue it replaces.** A retried job costs a few pence; an engineer diagnosing the same failure for the ninth time costs a morning. The saving does not appear in the infrastructure bill, which is precisely why it gets overlooked.

---

## 7.8 What the Platform Team Provides

| Platform provides | Team owns |
| --- | --- |
| Refresh orchestration — scheduled, event-driven, incremental | Declaring the freshness tier per source |
| Change detection and content hashing | Nothing; automatic |
| Deletion propagation and reconciliation passes | Confirming completion |
| Tombstone and purge mechanics, supersession chains | Deciding which applies to their content |
| TTL enforcement | Setting TTLs for their sources |
| The canary harness, scheduling and alerting | Writing and maintaining their canary set |
| The remediation engine and its library of automatic responses | Choosing which remedies apply to their sources, and their retry budgets |
| Self-healing event history and the self-healing-rate metric | Acting on a rising rate rather than tolerating it |
| Freshness dashboards and SLO tracking | Meeting their declared SLO |
| Index versioning, retention and one-command rollback | Deciding to roll back |
| Kill switch, per RAG, tested | Knowing when to pull it |
| Incident tooling and the runbook template | Their on-call rotation and their incidents |
| Cost reporting on refresh and canaries | Tuning cadence against cost |

---

## Where this leaves you

Seven chapters in, a team can build a RAG, expose it safely, prove it is right, account for what it costs, defend it, and keep it true.

Every one of those chapters has quietly assumed the same thing: **that the data is yours, and that you can bring it into your own boundary.**

Often it is not. The document management system belongs to a vendor and cannot be replicated. The regulatory database is licensed with terms forbidding bulk extraction. The partner's catalogue changes hourly. The mainframe holds forty years of records and nobody will sign off on copying them. Another team already owns a corpus you need, and Chapter 1 was explicit that you must not copy it.

What do you do when the knowledge is real, needed, and not yours to move?

---

[← Chapter 6 — Security and Safe Operations](../chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md) | [Chapter 8 — Data You Cannot Move →](../chapter-8-data-you-cannot-move/chapter-8-data-you-cannot-move.md)
