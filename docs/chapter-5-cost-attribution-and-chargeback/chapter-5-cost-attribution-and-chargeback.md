# Chapter 5 — Cost Attribution and Chargeback

> **The bridge.** Chapter 4 gave the estate a trace for every request. That same trace already knows who asked, which teams did work, and what it cost. This chapter turns telemetry into an economic model — because an estate nobody can bill is an estate nobody will fund.
>
> **This chapter covers** untraceable spend, cross-team consumption, the gateway that makes attribution possible, optimisation, and the cost of governance itself.

---

## 5.1 The Problem: The Bill Arrives at the Wrong Team

A user in Sales asks the workplace assistant a question. It fans out to People Ops, Payroll and the Policy Library. Three retrievals, three generations, one synthesis.

Every cent of that is billed to the teams that ran the compute. Not one of them chose to spend it. Not one of them can decline. And the team whose user caused it sees nothing at all.

Multiply by a working estate:

**The owning team is punished for being useful.** The Policy Library builds an excellent RAG. Word spreads. Twelve teams integrate. Their cloud bill quadruples with no change to their own product and no additional budget. The rational response — and teams do reach it — is to make the interface less available. **A cost model that penalises sharing will produce an estate that does not share,** which destroys the entire premise of the mesh.

**The consuming team has no price signal.** Building an agent that calls eight RAGs for every message is free, from where they are standing. So they do. Nobody is behaving badly; there is simply no feedback.

**Finance sees one enormous line item.** "AI spend" grows 40% a quarter with no explanation of what drove it. The eventual response is a freeze, and the freeze lands on everyone including the teams delivering value.

**Nobody can answer the question that matters.** *What does it cost us to answer one employee question?* Without that, there is no business case, no ROI story, and no defensible budget.

> **A federated estate without cost attribution does not stay federated. It gets centralised again by Finance, for perfectly rational reasons.**

---

## 5.2 The Principle: Cost Follows the Identity That Caused It

> **Every unit of consumption is attributed to the verified identity that initiated the request, attributed asynchronously from telemetry, and reported to that identity's cost centre.**

Four clauses, each deliberate.

**The identity that initiated it.** Not the team that executed the work. The originating principal, carried through the delegation chain from Chapter 3, is the causal agent. That is who should see the cost.

**Verified.** The attribution key comes from the identity provider's token, not from a field the caller populates. A caller-supplied cost centre header is an invitation to mislabel spend — accidentally at first, deliberately once budgets tighten. **Cost attribution based on a self-declared header is not an accounting system; it is an honour system with a spreadsheet.**

**Asynchronously, from telemetry.** Attribution is not in the request path. Spans already carry principal, cost centre, model, tokens and duration. A batch process aggregates them. Nothing added to latency, nothing to fail in production, and — importantly — the instrumentation was already built in Chapter 4.

**Reported to the cost centre.** Not necessarily charged. Start with showback.

### Showback before chargeback

**Showback:** you consumed this, it cost this much, here is the breakdown. No money moves.

**Chargeback:** the amount is transferred to your budget.

Start with showback, and stay there for at least a couple of quarters. Showback alone changes behaviour — a team shown that its agent costs £4,000 a month will optimise it without any money moving, simply because it is now visible and attributable.

Going to chargeback too early does real damage. The data is not yet trusted, disputes consume everyone's time, and teams start avoiding shared services to avoid unpredictable bills — which is precisely the behaviour the model was meant to prevent. Move to chargeback only when the numbers have been stable and uncontested for a while.

---

## 5.3 The Pattern: Telemetry In, Statements Out

```text
  REQUEST PATH  (nothing added — no cost logic here)
  ─────────────────────────────────────────────────
    request ──► middleware emits span:
                  principal.subject
                  principal.cost_centre   ← from token, at the edge
                  principal.chain
                  service.team            ← who executed
                  gen.model, tokens_in, tokens_out, cached_tokens
                  retrieval.candidates, returned
                  duration
                        │
                        ▼
                  telemetry pipeline
                        │
  ─────────────────────────────────────────────────
  ASYNCHRONOUS  (batch, out of band)
                        │
                        ▼
             1. join spans by trace id
             2. price each unit of consumption
             3. attribute to originating cost centre
             4. attribute a second copy to executing team
             5. reconcile against the provider's invoice
                        │
                        ▼
             daily statement per cost centre
             daily statement per RAG owner
```

### Two ledgers, not one

This is the part usually got wrong, and the reason single-ledger models produce endless arguments.

**The consumption ledger** — what a cost centre *caused*, wherever it ran. Sales caused £820 yesterday: £310 in its own agent, £180 in People Ops, £220 in the Policy Library, £110 at the gateway.

**The service ledger** — what a team *spent serving others*. People Ops spent £4,100 yesterday, of which £3,600 was serving other teams, across nine consumers.

Both are true, and each answers a question the other cannot.

- The consumption ledger tells Sales what its assistant really costs, so it can decide whether the value justifies it.
- The service ledger tells People Ops it is running a shared service at scale, gives it evidence for funding, and protects it from being penalised for success.

**Chargeback, when it comes, moves money along the consumption ledger.** People Ops is reimbursed for serving others and carries only its own use. That is the arrangement that makes being useful safe.

### The statement

```text
  COST CENTRE: CC-4471  Sales Operations        25 Sep 2026

  Total                                              £ 821.40
  vs 7-day average                                     +18%  ▲

  BY SERVICE
    Sales Assistant (own)                            £ 312.00
    People Ops agent                                 £ 181.20
    Policy Library agent                             £ 216.40
    Customer Master MCP                              £   4.80
    AI gateway                                       £ 107.00

  BY UNIT
    Generation — input tokens      18.2 M            £ 402.10
    Generation — output tokens      1.9 M            £ 289.60
    Embedding (queries)             0.4 M            £   6.20
    Retrieval / infrastructure                       £  16.50
    Governance (eval, tracing)                       £ 107.00

  NOTABLE
    Sales Assistant: 41% of input tokens were repeated
    system-prompt content not served from cache.
    Estimated recoverable: ~£95/day.   [how to fix ▸]

  ANSWERS SERVED                                       12,480
  COST PER ANSWER                                      £ 0.066
```

Design notes that decide whether anyone reads it:

**Cost per answer is the headline number.** Absolute spend invites panic; unit economics invite decisions. A team can reason about whether an answer is worth seven pence.

**Anomalies are surfaced, not buried.** An 18% jump appears at the top.

**Every finding links to a remedy.** A statement that identifies waste without telling you how to fix it produces frustration, not savings.

**Daily, not monthly.** A month-old surprise is unfixable and unattributable to any particular change.

> **Reference implementation.** Span export to the telemetry backend, batch aggregation in the data warehouse (BigQuery, Redshift, Synapse, or an internal warehouse), reconciled against the provider's billing export, published to a dashboard and a daily digest. The mechanism is ordinary data engineering — the design work is in the identity model, not the plumbing.

---

## 5.4 The AI Gateway: What It Is, and What It Costs You

The gateway has been referred to in passing. It deserves a proper introduction, because a great deal rests on it.

**An AI gateway is a single control point through which model calls pass.** Not a RAG, not an agent, and not required to be a single deployment. It exists because a set of concerns are worthless unless applied uniformly:

- **Authentication and quota** per principal
- **Usage metering** — the authoritative token record, independent of what any application claims
- **Model routing** — logical names mapped to concrete deployments, so a model can be swapped without touching applications
- **Rate limiting and fairness** — one team's batch job cannot exhaust shared provider quota
- **Guardrails** — prompt-injection screening, output filtering, PII detection, applied identically everywhere
- **Caching** — shared prompt-prefix and response caching, which is often the largest single saving available
- **Failover** — provider outage handled centrally rather than eleven times

Without a gateway, every one of these is a per-team implementation, which means it is inconsistent, which means the estate's posture is the weakest team's.

### One gateway, or one per team?

This is the question that actually decides the design, and it is usually skipped because "the gateway" sounds singular. It need not be. Three topologies, and the default is not the obvious one.

**Per-team gateway — built by the platform, deployed and configured by each team.** The platform ships the gateway as a versioned, parameterised component, exactly as it ships the Tier 2 pipeline. Each team runs an instance inside its own boundary, with its own configuration, its own quota, its own keys. Policy, telemetry schema and metering format are set centrally; the deployment is local.

**This should be the default,** for the same reasons Tier 2 runs inside the team's boundary. There is no shared blast radius — a team's misconfiguration degrades that team. Traffic does not leave the team's perimeter, which matters where residency or isolation constraints apply. Latency is local. And the platform team is not operating a component whose outage stops the entire organisation, which is a considerably more comfortable position than the alternative.

The cost is real: you lose cross-team caching, cross-team quota pooling and the single provider-failover implementation. Those are genuine benefits, and they are the reason the next option exists.

**Central gateway — one logical control point for the estate.** Maximum uniformity, maximum leverage on shared caching and pooled provider quota, one place to change a policy. Also one place for everything to fail, which is the subject of the next section, and a component the platform team must operate at tier-one standard indefinitely.

**Hybrid — central for shared provider access, per-team for everything else.** The central layer holds the provider credentials, the pooled quota and the failover logic. The per-team layer holds policy enforcement, metering and guardrails. This gets most of the leverage with a much smaller central failure domain, and it is where estates tend to arrive after trying one of the other two.

> **Choosing between them.** Take the per-team default unless you can name the specific shared benefit you are buying — and be honest that a central gateway is a tier-one service with an on-call rotation, not a piece of internal tooling. Organisations that adopt the central topology by accident, because it was simplest to stand up first, discover its cost at the first outage.

> **Reference implementation.** Gateways of this kind are increasingly assembled from an existing API management platform rather than written — Apigee on Google Cloud, for instance, which supplies authentication, quota, routing, metering and failover already, leaving the AI-specific policies to be added on top. The screening layer is typically a separate managed service configured with a floor the teams cannot lower. Comparable positions exist on the other major platforms, and self-hosted API gateways cover the same ground on-premises.
>
> The point of the reference is the assembly, not the product: very little of a gateway is novel, and building one from nothing is usually a decision to re-implement API management with fewer features.

### Start with visibility, not with chargeback

Whichever topology you choose, do not turn on cross-charging first. The sequence matters more than most teams expect:

**First, make spend visible.** Statements produced, delivered, and nobody billed. This alone changes behaviour, because most excess spend is not deliberate — it is a `top_k` nobody revisited and a prompt nobody measured. Visibility also surfaces the attribution bugs, and there will be attribution bugs.

**Then, make spend attributed.** Teams see their own number and can reconcile it. Disputes happen here, which is exactly where you want them: before money moves.

**Only then, make spend charged.** By which point the numbers are trusted, the mechanism is understood, and the conversation is about budgets rather than about whether the system is lying.

Reversing this order produces a predictable outcome: the first statement is wrong, one team disputes it publicly, and the credibility of the whole programme goes with it.

### The single point of failure, if you centralise

This applies to the central and hybrid topologies. It is the main argument against them, and it must be stated plainly, because a central gateway that fails takes down every AI capability in the organisation simultaneously.

**Mitigations, in order of importance:**

**It is not one instance.** Multiple replicas across zones and regions, behind a load balancer, with health-based routing. "The gateway" is a logical control point, not a box.

**It is stateless.** Policy and quota state live in external stores; any replica can serve any request. A restart loses nothing.

**Policy is cached locally with a fallback position.** If the policy store is briefly unreachable, replicas serve from cached policy and fail closed for restricted tools, open for ordinary ones. Decide and document that split rather than discovering it during an incident.

**A documented bypass exists for declared emergencies.** A break-glass path with direct provider credentials, heavily audited, time-boxed, and automatically revoked. Absent a sanctioned bypass, teams will build unsanctioned ones during the first outage and never remove them.

**It is operated as a tier-one service.** Error budget, on-call rotation, load testing at multiples of peak, capacity headroom. An organisation that routes everything through a component it treats as internal tooling has made a choice it has not noticed.

**It stays thin.** Every feature added is latency added to every AI call in the company and another way for everything to break at once. Concerns that must be uniform belong in it. Everything else does not.

---

## 5.5 Optimisation: The Levers, and Where They Live

Cost visibility without a route to savings produces anxiety rather than efficiency. The full catalogue is in [The Cost Optimisation Catalogue](cost-optimisation-catalogue.md); the highest-value levers are these.

**Prompt caching.** In most estates the largest single saving and the least exploited. System prompts, tool definitions and retrieved context repeat enormously across requests. Providers charge a fraction for cached prefixes. Structuring prompts so the stable portion comes first is a small change with a large effect.

**Model tiering.** Not every step needs the strongest model. Query rewriting, classification, routing and extraction run acceptably on small models at a fraction of the price. The strong model is for the final synthesis, where quality is visible.

**Reranking instead of large `top_k`.** Retrieving fifty chunks and sending them all to the model is expensive and usually *worse* — the model must ignore most of them. Retrieve broadly, rerank, send five. Better answers and lower generation cost together, which is rare enough to be worth prioritising — provided the reranker itself is cheap, which is an assumption to test rather than accept. See the [cost optimisation catalogue](cost-optimisation-catalogue.md).

**Context compression.** Summarise or trim retrieved passages before generation, where fidelity permits.

**Batch where latency permits.** Evaluation runs, bulk enrichment and re-embedding do not need interactive latency. Batch pricing is substantially cheaper.

**Right-size retrieval infrastructure.** Index replicas provisioned for a launch spike and never revisited are a common, invisible, ongoing cost.

**Stop paying for questions you should refuse.** Requests outside a RAG's scope cost full price to answer badly. Cheap upfront scope classification saves money and improves quality at once.

---

## 5.6 The Cost of Governance Itself

Chapters 3, 4 and 6 all add cost, and it is easy to leave out of the model — until Finance finds it and asks why "AI spend" includes things nobody can name.

| Governance activity | What drives the cost |
| --- | --- |
| Evaluation runs | Retrieval + generation for every question in the set, every gate |
| Judge model | Often the largest governance line; an LLM call per graded dimension |
| Trace storage | Volume × retention; grows quietly with adoption |
| Cost pipeline | Warehouse storage and daily aggregation |
| Re-embedding | Periodic full reprocessing — lumpy, large, forecastable |
| Guardrail screening | A filtering call on every request, both directions |
| Canary queries | Continuous synthetic traffic ([Chapter 7](../chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md)) |

**Budget it explicitly, as a named line.** Size it against what it protects: the more consequential the answers, the more of the spend belongs to knowing they are right. Start from a stated proportion of total AI spend rather than from whatever is left over — a low double-digit percentage is a defensible opening figure for most estates, and the number matters far less than the fact that it is named. That is the price of knowing whether the system is correct, safe and affordable — and it is unambiguously worth paying. What is not worth doing is leaving it unlabelled, because unexplained cost is the fastest route to a governance programme being cut.

**Keep it proportionate to risk.** A RAG over public marketing content does not need the evaluation cadence of one over regulatory guidance. Tie governance intensity to data classification, and say so in the budget.

---

## 5.7 What the Platform Team Provides

| Platform provides | Team owns |
| --- | --- |
| The attribution pipeline — spans to statements | Keeping their identities and workloads correctly registered |
| Mandatory labelling standards, enforced in the landing zone | Applying them; untagged resources are non-conformant |
| Both ledgers, daily statements, dashboards | Reading them and acting |
| Anomaly detection and alerting on spend | Responding to their own anomalies |
| The gateway — HA, metering, routing, caching, guardrails, failover | Using it; no direct provider credentials |
| Shared prompt-cache configuration and guidance | Structuring prompts to exploit it |
| The optimisation catalogue and cost-review tooling | Applying it to their own workload |
| Reconciliation against the provider invoice | Disputing attribution they believe is wrong |
| Forecasting models for planning | Their own demand forecast |

---

## Where this leaves you

The estate is now buildable, exposable, measurable and accountable. Someone can say what an answer costs and who caused it.

None of which makes it safe.

Everything so far has assumed the content is trustworthy — that a retrieved passage is simply information. It is not. A retrieved passage is **untrusted input that will be placed directly into the context of a model that can call tools.** A document containing instructions rather than facts is an attack, delivered through the front door, by a system designed to fetch it.

And that is one item on a list: rate limiting and quota abuse, deletion requests that must reach vectors and caches and traces, data classification and residency, safe test environments, and the question of what to do at two in the morning when the assistant has told a customer something untrue.

Is it safe, and can you keep it safe?

---

**Supporting material for this chapter**

- [The Cost Optimisation Catalogue](cost-optimisation-catalogue.md) — every lever, where it applies, what it saves, and what it costs in quality

---

[← Chapter 4 — Proving Retrieval Quality](../chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) | [Chapter 6 — Security and Safe Operations →](../chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md)
