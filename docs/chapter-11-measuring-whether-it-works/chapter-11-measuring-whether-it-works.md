# Chapter 11 — Measuring Whether It Works

> **The bridge.** Ten chapters described an operating model. This one asks whether it is earning its place — and answers the question every organisation actually starts with: *where do we begin?*
>
> **This chapter covers** proving value, the maturity ramp that gives the numbers meaning, where to start, and how to migrate from whatever you have today.

---

## 11.1 The Problem: A Programme Nobody Can Justify

Eighteen months in. Several teams have RAGs. There is a platform team, a gateway, a registry. Spend is significant and growing.

Someone senior asks whether it is working, and the room has nothing.

**"Adoption is strong"** — is it? Eleven teams have built something. How many are used daily, and by whom?

**"Quality is good"** — measured how, against what, and moving which way?

**"It saves time"** — whose, how much, and against what baseline?

**"We're ahead of competitors"** — on what evidence?

The answers are vague because the measurements were never designed. The consequences are predictable and they arrive in a specific order:

**The programme is cut**, or frozen, because an unjustified cost is the easiest cost to remove.

**The wrong things are optimised.** Absent evidence, teams optimise what is visible. Model spend is visible. Whether anyone gets better answers is not. So the estate gets cheaper and less useful, and the second half is invisible.

**Nobody knows what to fund next.** More platform capability? More team enablement? Better models? Better content? These have very different returns and no way to compare them.

**Failure stays hidden.** A RAG nobody uses costs money and produces nothing, and looks identical on a status report to one answering a thousand questions a day.

> **A single metric will not do.** "Number of RAGs" rewards building over usefulness. "Cost savings" invites fiction. "User satisfaction" drifts without cause. Several metrics, read together, with a maturity ramp giving them meaning.

---

## 11.2 The Principle: Measure Outcomes, Adoption, Quality and Economics Together

> **Measure four dimensions continuously — is it used, is it good, is it affordable, and is it producing outcomes — and interpret them against a maturity stage, because the same number means different things at different stages.**

The last clause is what most measurement programmes miss. **A cost-per-answer of eighty pence is a success in month three and a failure in year two.** Without a stage, numbers are just numbers, and they get argued about rather than acted on.

Measure all four, or the estate optimises the measured one at the expense of the rest. Adoption alone produces usage of a system giving poor answers. Quality alone produces an excellent system nobody uses. Economics alone produces a cheap, useless estate. Outcomes alone are too slow to steer by.

---

## 11.3 The Metrics

### Dimension 1 — Adoption

| Metric | Why |
| --- | --- |
| Weekly active users, by team | The honest usage number; totals hide abandonment |
| Questions answered per week | Volume, and its trend |
| Repeat usage rate | **The best single adoption signal.** People who come back found it useful; people who tried it once did not |
| Teams with a registered, used RAG | Breadth of the estate |
| Consumers per RAG | Reuse — the payoff for Chapters 3 and 10 |
| Time to first RAG for a new team | **The platform's own score.** Months means the paved road is not paved |
| Shadow implementations found | Unregistered systems indicate the sanctioned path is too hard |

**Repeat usage and time-to-first-RAG are the two to watch.** The first tells you whether the product is good. The second tells you whether the platform is.

### Dimension 2 — Quality

| Metric | Why |
| --- | --- |
| Faithfulness and recall trend, estate-wide | Chapter 4's metrics, aggregated |
| Answer acceptance rate | Users' own verdict |
| Escalation-to-human rate | Where the system falls short |
| Stale-document retrieval rate | Chapter 7's decay signal |
| Conflict rate | Chapter 4's contradiction detection |
| Refusal rate | Healthy in moderation; scope problems at extremes |
| Incidents, by severity | Wrong answers that reached someone |
| Mean time to investigate | Whether the Chapter 4 investment pays off |

**Trends matter more than levels.** A faithfulness score of 0.87 is meaningless in isolation. Moving from 0.84 to 0.91 over a quarter is the finding.

### Dimension 3 — Economics

| Metric | Why |
| --- | --- |
| **Cost per answered question** | The headline. Should fall as the estate matures |
| Total spend, and its trend | The number Finance will quote |
| Governance as a share of spend | Against the figure named in Chapter 5 |
| Cost per team, per RAG | Attribution working |
| Platform cost per team served | Platform leverage — must improve with scale |
| Infrastructure utilisation | Waste, especially on-premises |
| Cache hit rate | The largest unexploited saving in most estates |

**Cost per answered question is the one to put on the first slide.** It is a unit economic, it trends, it is comparable across teams, and it invites decisions rather than panic.

### Dimension 4 — Outcomes

The hardest, the slowest, and the only dimension that justifies the programme.

| Metric | How |
| --- | --- |
| Time saved per query class | Measured, not estimated — time a sample of users on the old path |
| Deflection from support channels | Ticket volume against a baseline period |
| Time to competence for new joiners | A strong, under-used measure |
| Decision latency | Where the bottleneck was finding information |
| Error reduction | Where the old path was people working from stale documents |
| Access to previously unreachable knowledge | Questions that simply could not be answered before |

**Two rules for outcome measurement.**

**Baseline before you deploy.** After the fact, you are guessing, and everyone knows it. Time ten people doing the task the old way, before there is a new way. It takes an afternoon and it is the difference between evidence and assertion.

**Be conservative.** Inflated savings claims are found out, and when they are, the credible claims go with them. A defensible small number outperforms an impressive unsupportable one every time.

### Where these numbers actually come from

A measurement programme fails most often not because the wrong metrics were chosen but because nobody established, before promising them, whether they could be produced at all. The four dimensions above are not equally cheap, and the difference is worth stating explicitly.

**Falls out of the tracing and billing you already built.** Every adoption metric, every economic metric, and most quality metrics derive from the spans specified in Chapter 4 and the attribution specified in Chapter 5. If those are instrumented, these numbers are a query. If they are not, no amount of reporting effort will conjure them.

**Requires the evaluation harness.** Faithfulness, recall and conflict rate come from Chapter 4's evaluation sets, not from production traffic. They exist only if someone maintains the golden set, and they are only as good as that set's coverage.

**Requires users to tell you something.** Answer acceptance and escalation rate depend on a feedback mechanism being present in every consuming interface — which is a design decision made by the teams building those interfaces, not by the platform, and therefore one the platform has to ask for early.

**Cannot be derived from the system at all.** Time saved, deflection, time to competence, decision latency and error reduction require a baseline measured outside the system, before it exists, by a human. **There is no instrumentation that produces these retrospectively.** This is the whole reason the baseline rule above is stated so firmly: the outcome dimension is the one that justifies the programme, and it is the one that becomes permanently unavailable the moment you skip it.

The practical consequence for planning: the first three dimensions are an engineering task, and the fourth is a task you have to schedule *before* the engineering starts.

---

## 11.4 The Dashboard

One page. Leadership will not read more.

```text
  ENTERPRISE RAG ESTATE                        September 2026
  ══════════════════════════════════════════════════════════
  MATURITY  ███████░░░  Stage 3 — Federating

  ADOPTION                          QUALITY
    Weekly active users    3,840      Faithfulness      0.91  ▲
    Questions / week      48,200      Acceptance rate    87%  ▲
    Repeat usage             72%      Escalation         11%  ▼
    Teams with a RAG       14/31      Stale retrieval   1.2%  ▼
    Avg consumers / RAG      3.2      Incidents (P1/P2)  0/2

  ECONOMICS                         OUTCOMES
    Cost per answer      £0.061  ▼    Time saved / wk   1,180 h
    Total spend / mo     £  118k      Support deflection   23%
    Governance share        12%       New-joiner ramp    -31%
    Platform / team      £  2.4k  ▼   Time to first RAG   3 wk  ▼

  ATTENTION
    ▲ Logistics assistant: cost/answer 3.1× estate average
    ▲ 2 RAGs breaching freshness SLO for 14+ days
    ▲ 1 RAG with no reachable owner — suspension in 7 days
    ● Finance RAG: highest acceptance rate — practices worth copying
```

**Attention items are what make it a management tool rather than a report.** Numbers describe; exceptions demand a decision. Include the positive one — the best-performing RAG is a reusable practice, and copying it is cheaper than any platform feature.

---

## 11.5 The Maturity Ramp

Metrics need a stage to be read against. Five stages, each with an entry condition, a focus, and the realistic failure that ends it.

### Stage 1 — Proving It (one to three months)
**Focus:** one team, one corpus, one real use case.
**Looks like:** Tier 1 managed search. Minimal governance. A genuine business question answered.
**Measure:** does it work? do the intended users come back?
**Fails by:** choosing a showcase use case instead of a painful one. Pick a problem people complain about, not one that demonstrates well.

### Stage 2 — Making It Repeatable (three to nine months)
**Focus:** two or three more teams, and the platform's first deliverables.
**Looks like:** the paved road exists. The interface contract is specified and enforced. Tracing and evaluation are in place. Time to first RAG becomes a tracked number.
**Measure:** is the second team faster than the first? The third faster than the second?
**Fails by:** letting each team build its own way. If team three still takes four months, there is no platform — there is a consultancy.

### Stage 3 — Federating (nine to eighteen months)
**Focus:** agents consuming other teams' capabilities. The mesh becomes real.
**Looks like:** A2A traffic. The registry in use. Cost attribution producing statements. Conflicts detected and resolved by declared authority.
**Measure:** consumers per RAG. Cross-team traffic share. Registry-mediated discovery.
**Fails by:** cost attribution arriving too late. Teams penalised for serving others will close their interfaces, and the mesh stops before it starts. **Build Chapter 5 before Stage 3, not during it.**

### Stage 4 — Operating It (eighteen months onward)
**Focus:** reliability, economics and lifecycle.
**Looks like:** freshness SLOs met. Incident response rehearsed. Cost per answer falling. Decommissioning happening. Orphan detection running.
**Measure:** SLO attainment, cost-per-answer trend, MTTR, orphan count.
**Fails by:** treating it as finished. Corpora decay, owners leave, and an estate that stops being tended is worse than one that was never built, because it is trusted.

### Stage 5 — Compounding
**Focus:** knowledge as infrastructure.
**Looks like:** new capabilities assembled from existing ones. New teams productive in days. Content quality improving because the RAG keeps surfacing defects in it.
**Measure:** proportion of new capabilities built from existing registry entries. Marginal cost of the next use case.
**Fails by:** nothing external — this stage fails from neglect of Stage 4.

**Two honest notes.** The timescales assume real investment; halve the effort and more than double the duration. And **skipping stages does not work.** Organisations that start at Stage 3 — building a mesh before the paved road exists — produce eleven incompatible systems and a governance programme that arrives too late to govern anything.

---

## 11.6 Starting From Where You Are

Nobody greenfields this. Two starting positions, both common.

### Position A — Nothing yet

**Do not start with the platform.** A platform team building a paved road before anyone has walked anywhere paves the wrong road. Start with one team, get something working, and derive the road from what they actually needed.

**Choosing the first team.** Four criteria, in order: a real and painful information problem; a corpus that is reasonably clean and clearly owned; engineers willing to be first and to be measured; and a leader who will talk about the result honestly, including the parts that disappointed.

**Baseline before you build.** Time the current path. This afternoon of work is what makes the outcome claim defensible a year later.

**Extract the platform from the second and third implementations, not the first.** One instance is an anecdote. Three reveal what is genuinely common.

### Position B — A central RAG already exists

Most organisations arrive here: one team built something central, it works for its original scope, and it is straining under everything since added to it.

**Do not switch it off, and do not rewrite it.** Strangle it.

```text
  1  PUT THE INTERFACE IN FRONT OF IT
     The existing RAG gains a governed MCP/A2A boundary.
     Consumers move to the interface. Nothing else changes.
     ► This alone is worth the effort — it buys the freedom
       to change everything behind it.

  2  MOVE ONE DOMAIN OUT
     The domain with the clearest owner and the loudest
     complaints. That team builds its own RAG and exposes
     the same contract. Consumers are unaffected.

  3  ROUTE
     Queries for that domain go to the new RAG. The old
     corpus keeps its copy until confidence holds.

  4  RETIRE THAT SLICE
     Remove the domain's content from the central index.
     It gets smaller, faster and better — immediately.

  5  REPEAT
     Domain by domain, in order of pain.

  6  WHATEVER REMAINS
     Becomes one more team-owned RAG, or is decommissioned
     per Chapter 10 if it turns out nothing was left.
```

**Why this works.** Every step is small, reversible, and independently valuable. Consumers never experience a migration. The central team is relieved of load rather than dispossessed — which matters enormously, because they are the people who must cooperate for any of it to happen.

**The order is by pain, not by ease.** The domain whose users complain most is the domain whose team will do the work.

### The minimum platform to launch

Do not build Chapters 2 through 10 before the first team ships. The launch minimum:

- **A working Tier 1 path** — one supported way to build a RAG without inventing a pipeline
- **The interface contract**, specified, with boilerplate that implements it
- **Identity and tool-level authorisation** — Chapter 3's core; retrofitting security is expensive and retrofitting it across eleven teams is worse
- **Tracing** — instrument from day one. It cannot be added retrospectively to answer a question about last month.
- **A registry**, even a minimal one. A list with owners and scopes beats nothing, and it starts the habit.

**Everything else follows demand.** Chargeback when cross-team traffic appears. Freshness tooling when a corpus goes stale. The on-premises path when a team needs it.

**But note the two that cannot wait: tracing and identity.** Both are architecturally invasive, both are nearly impossible to retrofit across an estate, and both are the things a team under delivery pressure will skip. Make them free, in the boilerplate, from the first day.

---

## 11.7 What the Platform Team Provides

| Platform provides | Team owns |
| --- | --- |
| The estate dashboard and its metric definitions | Their own metrics and trends |
| Automatic collection from telemetry — no manual reporting | Nothing; derived |
| Baseline measurement tooling and method | Baselining before they build |
| The maturity model and stage assessment | Knowing their stage |
| Migration patterns and strangler tooling | Executing their migration |
| Benchmarking across teams, and surfacing good practice | Adopting what works elsewhere |
| The quarterly review pack for leadership | Their contribution to it |

**One deliberate absence.** Nobody should ever fill in a spreadsheet for this. Every number above is derivable from Chapters 4, 5, 7 and 10 — which is the quiet argument for having built them. **Manual metric collection decays within two quarters, always**, and then the dashboard is worse than nothing because it is trusted and wrong.

---

## The end of the argument

Eleven chapters, one thesis.

A single enterprise RAG cannot work, for reasons that are structural rather than technical — embedding spaces that cannot be shared, access control that cannot be centralised, corpora that nobody owns, and knowledge that decays faster than any one team can tend it.

So ownership follows the data. The team that produces it owns the RAG over it, exposes it behind a governed interface, and shares nothing but text and citations.

That decision is correct, and it creates a new problem set — multiplied by every team in the organisation. How to build without each team inventing a pipeline. How to expose without losing control. How to know it is right. Who pays. How to keep it safe. How to keep it true. What to do about data you cannot move. What to do without a cloud. How anyone finds any of it. Whether it is working at all.

Each of those has an answer, and the answers share a shape: **the principle is centralised, the execution is not.** One embedding registry, many corpora. One interface contract, many interfaces. One tracing standard, many services. One security baseline, many teams. One paved road, many destinations.

That is what the platform team is for. Not to own the knowledge — it cannot, and should not try. **To make the right thing the easy thing, eleven times over, so that eleven teams can own their own knowledge without eleven times the work.**

The organisations that get this right will not be the ones with the best models. Models are a commodity and they improve for everyone simultaneously. They will be the ones whose knowledge is owned, current, discoverable, governed and reusable — because that is the part no vendor can supply, and it is the part that compounds.

---

[← Chapter 10 — Discovering What Already Exists](../chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md) | [Back to the start ↑](../README.md)
