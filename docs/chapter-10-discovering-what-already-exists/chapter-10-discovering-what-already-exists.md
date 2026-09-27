# Chapter 10 — Discovering What Already Exists

> **The bridge.** Chapter 1 broke the monolith into many team-owned RAGs. Nine chapters made each one buildable and governable. None of them helped anybody find one, and none of them gave a team one place to go.
>
> **This chapter covers** discoverability, requesting access, the ownership lifecycle including what happens to a RAG whose owner has left — and the portal through which all of it is actually experienced.

---

## 10.1 The Problem: Rebuilding What Already Works

A team in Logistics needs supplier risk knowledge. They have questions and no way to answer them:

- Does a RAG covering supplier risk exist?
- Who owns it, and is it current?
- What is in it — and what is deliberately not?
- May we use it, and how do we ask?
- Is it an MCP tool, an A2A agent, or both?
- What will it cost us?
- Can we depend on it?

They ask a few people. Someone thinks Procurement has something. That team has moved on. Two weeks pass, and Logistics does the rational thing: builds its own, over largely the same documents, without the domain knowledge Procurement accumulated, and slightly wrong in ways nobody will notice for a year.

Now there are two supplier risk RAGs, giving different answers, and the conflict detection in [Chapter 4](../chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) will eventually surface a contradiction that neither team can explain.

> **This is the monolith's fragmentation problem, returning in a new form. Decentralisation without discoverability is not a mesh. It is a scatter.**

And it is not merely wasteful. Four mechanisms from earlier chapters quietly assumed a record that does not exist:

| Chapter | Assumed | Without a registry |
| --- | --- | --- |
| 3 — versioning | You know who consumes your interface | You can never safely change it |
| 4 — authority | Someone declared which team is the source of truth | Conflicts are resolved by chance |
| 5 — chargeback | Identities map to cost centres, consumption is attributable | Statements cannot be produced |
| 6 — deletion | You know who else ingested your content | Deletion stops at your boundary |
| 7 — incidents | Every RAG has a reachable owner | An orphan keeps answering and nobody can stop it |

**The registry is not a convenience. It is the substrate the operating model runs on** — which is why it appears here, after everything that depends on it, rather than as an introductory nicety.

### And the ownership problem

Organisations reorganise. Teams merge, split and dissolve. People leave. A RAG built by a team that no longer exists keeps running, keeps answering, keeps costing money, and keeps being wrong — with nobody accountable and nobody entitled to switch it off.

**An orphaned RAG is the most dangerous object in the estate.** It has all the authority of a governed system and none of the ownership.

---

## 10.2 The Principle: If It Is Not Registered, It Does Not Exist

> **Every RAG, MCP server and agent in the estate is registered with a named owner, a declared scope, a stated freshness commitment, and a documented access path. Registration is a condition of operating, not an optional courtesy.**

Three clauses that determine whether this works.

**Registration is enforced, not requested.** A voluntary registry is out of date within a quarter and then actively harmful, because people trust it. Enforcement comes from the paved road: platform-provisioned infrastructure registers automatically, the conformance suite from [Chapter 3](../chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md) gates registration, and **an unregistered interface is not routable through the gateway.** The unregistered path simply does not function.

**Entries are generated, not written.** Anything a human must remember to update will be wrong. Freshness, availability, cost, consumers, conformance status — all derived from live telemetry. Humans supply only what cannot be derived: purpose, scope, authority, and who to call.

**It is a product, not a catalogue.** Its success measure is *time for a team to find and start using an existing capability.* A registry that is complete and unusable has failed. This is worth saying because internal registries habitually optimise for completeness of fields rather than speed of answer.

> **The registry is the data model. The portal is the product.** Everything in the rest of this chapter — the entry, the search, the access request, the lifecycle — is described as data and workflow because that is what has to be right underneath. But no team experiences a data model. They experience one internal site they log into, and whether the operating model in this book succeeds depends substantially on whether that site is good. Section 10.5 is about the site.

---

## 10.3 What an Entry Contains

```text
  supplier-risk-knowledge                        ● healthy
  ─────────────────────────────────────────────────────────
  Owner        Procurement Operations
  Contact      #procurement-platform · on-call rota linked
  Domain       Supplier risk, compliance status, audit history

  SCOPE
    Contains   Supplier master records · risk assessments ·
               audit reports · sanctions screening results ·
               contract compliance findings
    Excludes   Contract terms (→ contracts-knowledge)
               Payment history (→ finance-ap-knowledge)
    Authority  Source of truth for supplier risk rating
               and compliance status

  ACCESS
    A2A agent  supplier-risk-agent        general questions
    MCP tools  search_supplier_documents  open to internal
               get_risk_assessment        restricted
               get_audit_findings         restricted
    Request    self-service for internal · approval for restricted
    Typical    approved within 1 working day

  OPERATIONAL                              (derived, live)
    Freshness  daily · SLO met 29/30 days
    Latency    p50 340 ms · p95 1.2 s
    Availability  99.7% (30 d)
    Quality    faithfulness 0.91 · recall 0.87 · trend flat
    Cost       ~£0.04 per query
    Conformance  Ch3 suite passing · last run 2 h ago

  CONSUMERS                                (derived)
    9 teams · 14 registered workloads
    top: logistics-assistant · risk-dashboard · audit-copilot

  DEPENDENCIES                             (derived)
    external: sanctions-screening-vendor (Ch8 Option B)
    internal: customer-master-agent

  LIFECYCLE
    Created 2025-11 · reviewed 2026-08 · next review 2027-02
    Status   active
```

Design points worth drawing out:

**"Excludes" is as valuable as "contains", and more often omitted.** It stops teams asking the wrong RAG and — crucially — tells them where to go instead. Half of "the RAG could not answer" reports are scope mismatches.

**Authority is declared here.** This is the field that makes Chapter 4's conflict resolution deterministic instead of arbitrary. Undeclared authority means the synthesising agent guesses.

**Operational data is live.** A self-reported freshness claim is aspiration. A measured SLO attainment is information — and a consumer deciding whether to depend on this capability needs the second.

**Consumers are derived from traces.** The list Chapter 3's versioning depends on, produced automatically from Chapter 4's telemetry. No team maintains it, so it is never stale.

**Dependencies expose the blast radius.** When the sanctions vendor is down, the registry can answer which capabilities degrade and which teams to tell — before they call.

---

## 10.4 Discovery and Access

### Finding it

**Search by question, not by name.** A team types "supplier sanctions status" and gets the right capability. They do not know what it is called; that is the whole problem. Semantic search over registry entries — the registry is itself a RAG, which is a pleasant symmetry and also the obviously correct implementation.

**Browse by domain** for orientation.

**Machine discovery, for agents.** An orchestrating agent queries the registry to find capabilities relevant to a request, filtered to those it is entitled to call. **This is what makes Chapter 5's "route rather than fan out" possible** — routing requires knowing what exists, and hard-coding the list in every agent does not scale past about four.

**Surface adjacent capabilities.** A team looking at supplier risk should see contracts and finance nearby. Most misses are one hop away.

### Requesting access

The registry is where a capability becomes usable, not merely visible.

```text
  find capability
        │
        ▼
  see entitlement tiers and what each permits
        │
        ▼
  request ── open tier ──────► granted automatically
        │
        └─ restricted tier ──► owner approves, with purpose
                                 │
                                 ▼
                        entitlement provisioned
                        · tool grant (Ch3)
                        · cost centre linked (Ch5)
                        · consumer recorded
                        · connection details issued
```

**Self-service for open tiers is what makes the mesh work.** If every access request needs a meeting, teams will copy data instead — and you have recreated the problem Chapter 1 solved.

**Approval for restricted tiers is what makes it safe.** The owner sees who is asking and why, and the purpose declaration is retained.

**Granting access registers the consumer**, which feeds versioning, deletion notification and dependency mapping. The bureaucratic-seeming step pays for itself in every one of those.

---

## 10.5 The Portal

Every capability in the preceding nine chapters is a thing a team must find, request, configure, monitor or hand over. Left as nine separate consoles and a wiki page, the paved road is not paved — it is a set of materials on a verge. The portal is where they become one experience.

### One sign-in, and the view follows entitlement

Authentication through the corporate identity provider, and **the portal shows each person what their entitlements permit and nothing else.** This is not a cosmetic decision. A platform engineer, a product engineer, a data owner and a finance analyst have almost no overlap in what they need, and a portal that shows everyone everything is one that nobody can navigate.

The same entitlement resolution that governs retrieval in Chapter 3 governs the portal. There is no second access model to maintain, and no possibility of the two disagreeing — which is the failure mode of every internal tool that rolls its own permissions.

### What the tabs are

**Discover.** The default landing view, and the one described in section 10.4. Ask a question in plain language, get capabilities. Browse by domain. See what is adjacent. For most people this is the only tab they ever open, and it should behave accordingly.

**Build.** The paved road, made navigable. The tier decision from Chapter 2, the three tier documents, the configuration templates, the infrastructure modules, and a path that provisions a working Tier 1 capability without anyone opening a ticket. This tab's success measure is time to first RAG, which is also the platform team's own score in Chapter 11.

**Operate.** One view per capability a team owns: freshness against its declared commitment, retrieval quality trend, latency, availability, conformance status, open incidents, the ingestion report. These are the numbers from Chapters 4, 6 and 7, gathered rather than scattered. **A team should never have to visit three consoles to answer "is my RAG healthy?"**

**Spend.** The statement from Chapter 5, per team and per capability, with the drill-down into what caused it and the optimisation levers that apply. Read-only for most, and reconcilable by the people who will dispute it.

**Govern.** For owners and for the platform: access requests awaiting approval, review dates approaching, orphan signals, conformance failures, deletion propagation status, reorganisation impact reports. This is the tab that turns the lifecycle in section 10.6 from a policy into a queue somebody works through.

**Learn.** The decision support — which tier suits which corpus, what goes wrong and its early symptom, the evaluation protocol, the runbooks. This exists because the knowledge to choose well is part of the platform's product, and a platform that ships capability without judgement produces teams that adopt it and misuse it.

### Onboarding is a flow, not a document

A new team arriving should be able to go from nothing to a registered, governed, observable capability without a meeting. The flow the portal has to carry:

```text
  1  DECLARE        what knowledge, whose, what questions
  2  CHOOSE TIER    guided by the disqualifiers, not by preference
  3  PROVISION      infrastructure, connectors, keys, boundary
  4  REGISTER       automatically, as a condition of provisioning
  5  CONFIGURE      from a template, in the team's own repository
  6  EVALUATE       against their own query set, before exposure
  7  EXPOSE         the governed interface, conformance suite passing
  8  PUBLISH        entry visible, access tiers open for request
```

Every step produces registry state. **Registration is not a form at the end; it is a by-product of having used the paved road at all** — which is what makes the enforcement in section 10.2 credible rather than punitive.

### Offboarding is a flow too, and it is the one that gets skipped

The reason orphans exist is that leaving is nobody's project. The portal has to make it as routine as arriving:

```text
  a person leaves     → owner group updated automatically via IdP
                         if the group empties, the orphan signal fires

  a team reorganises  → impact report lists every affected entry
                         each requires transfer, merge or decommission

  a consumer stops    → entitlement revoked, consumer list updated,
                         provider's versioning obligation reduced

  a capability ends   → the nine-step decommissioning workflow,
                         ending in a tombstoned entry, not silence
```

The fourth of those is covered in section 10.6. The first three are the ones that quietly do not happen, and each has a portal surface because otherwise it has no surface at all.

### The portal does not arrive complete

Building all six tabs before the first team ships would be a mistake, and it is a common one. The portal should grow with the estate, roughly in step with the maturity stages in Chapter 11:

| Stage | What the portal is |
| --- | --- |
| **First capabilities** | A page. Who owns what, and how to ask. Manually maintained, and that is fine for five entries |
| **Several teams** | Discover and Build, backed by a real registry with automatic registration. This is the point at which manual maintenance fails |
| **An estate** | Operate and Spend added, fed from the telemetry that Chapters 4 and 5 already require. Nothing new is instrumented; it is gathered |
| **Governed at scale** | Govern and Learn added. Access approval, orphan detection and review dates become queues rather than good intentions |
| **Mature** | Machine discovery is the dominant use. Agents query the registry more than humans do, and the portal's API matters more than its pages |

**Build each tab when the pain it removes is being felt**, not before. A portal with empty tabs teaches people that the portal is not where the answers are, and that lesson is expensive to unteach.

---

## 10.6 The Lifecycle: Preventing Orphans

Discoverability handles what exists now. This handles what happens over years.

### Ownership must be a team, never a person

```text
  owner_team:      procurement-operations
  owner_group:     grp-procurement-platform      ← IdP group
  escalation:      procurement-engineering-lead   ← role, not a name
  business_owner:  head-of-procurement-ops        ← role, not a name
```

Resolved through identity-provider groups, so it follows the organisation automatically. A named individual in a registry entry is a future orphan with a date on it.

### Orphan detection

Automated, because nobody will notice manually:

| Signal | Meaning |
| --- | --- |
| Owner group empty or dissolved | Certain orphan |
| No commits to the repository in 180 days | Probable abandonment |
| Review date passed by 90 days | Governance lapse |
| On-call rotation unstaffed | Unsupportable in an incident |
| Freshness SLO breached continuously | Refresh has stopped and nobody is watching |
| Conformance suite failing for 30 days | Drifting out of the security baseline |

Escalate: to the owning group, then to the escalation contact, then to platform governance. **A capability with no reachable owner is suspended** — it stops answering and says why. That sounds severe and it is the only workable policy: an unowned system that is still trusted is more dangerous than one that has visibly stopped.

### Reorganisations

When teams merge, split or dissolve, every affected registry entry needs an explicit decision: **transfer, merge, or decommission.**

The registry produces the list, which is the hard part. Reorganisations rarely fail on willingness; they fail because nobody can enumerate what needs reassigning. A query against the registry produces that enumeration in seconds.

**Merging two RAGs is not copying one index into another.** Reconcile scope, resolve conflicting authority declarations, combine evaluation sets, unify freshness tiers, and notify both consumer sets. It is a project, and pretending otherwise is how merged corpora start contradicting themselves.

### Decommissioning

A RAG that is genuinely no longer needed should be removed properly, not left running.

```text
  1. propose         reason, evidence of low or no use
  2. notify          every registered consumer, with alternatives
  3. deprecate       registry marked; callers warned in responses
  4. wait            a stated period — typically 90 days
  5. verify          traffic ceased; any stragglers contacted
  6. disable         stops answering; infrastructure retained
  7. retain          index kept, per retention policy
  8. remove          infrastructure destroyed
  9. tombstone       registry entry remains, marked decommissioned,
                     pointing at whatever replaced it
```

**Step 9 matters more than it looks.** Someone will search for this capability in two years. A tombstoned entry saying *decommissioned, superseded by X* is a far better answer than silence — silence gets interpreted as "nothing exists" and produces a rebuild.

---

## 10.7 What the Platform Team Provides

| Platform provides | Team owns |
| --- | --- |
| The registry itself, with semantic search and a machine API | Registering their capability |
| The portal, its tabs, and the entitlement-filtered views | Using it rather than building around it |
| Automatic registration through the paved road | Purpose, scope, exclusions, authority |
| Derived operational fields — freshness, quality, latency, cost | Meeting the commitments they declared |
| Consumer and dependency graphs from trace data | Nothing; automatic |
| Access request workflow and entitlement provisioning | Approving requests for their capability |
| Orphan detection and escalation | Keeping ownership current |
| The onboarding flow, end to end, without a meeting | Walking it |
| The offboarding flows — people, teams, consumers, capabilities | Triggering them honestly |
| Reorganisation impact reports | Deciding transfer, merge or decommission |
| The decommissioning workflow | Executing it |
| Gateway enforcement — unregistered means unroutable | Staying registered and conformant |

---

## Where this leaves you

The estate is now complete and coherent. Capabilities can be built, exposed, measured, paid for, defended, kept current, extended to data you do not own, run without a cloud, and — finally — found and reused.

That is a substantial programme of work. Which raises the question someone will ask, probably a week after the first chapter is implemented:

**Is any of it working?**

Not "is the system up." Is the organisation better off? Are people getting answers they could not get before? Is the estate getting cheaper per answer, or more expensive? Are teams adopting this, or quietly building around it? Is quality improving or drifting? Has any of this earned the investment, and what should be funded next?

And — the question that decides whether any of this ever begins — **where does an organisation that has none of this actually start?**

---

[← Chapter 9 — On-Premises and Air-Gapped](../chapter-9-on-premises-and-air-gapped/chapter-9-on-premises-and-air-gapped.md) | [Chapter 11 — Measuring Whether It Works →](../chapter-11-measuring-whether-it-works/chapter-11-measuring-whether-it-works.md)
