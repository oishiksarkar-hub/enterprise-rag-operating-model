# Chapter 4 — How Do I Know It Is Right?

> **The bridge.** Chapter 3 made the boundary secure. A secure interface returning wrong answers is still wrong — and in a mesh, a wrong answer travels, gets synthesised into other answers, and arrives at a human wearing the authority of three systems.
>
> **This chapter covers** measuring retrieval quality, finding the failing hop in a multi-agent request, acting on user feedback, resolving conflicting answers between teams, and testing a system that is not deterministic.

---

## 4.1 The Problem: Nobody Can Tell You Why the Answer Was Wrong

A user reports that the assistant gave a wrong answer about parental leave.

In a monolith you open one log and read the retrieved passages. In a federated estate, this happened:

```text
  user ──► workplace assistant
             ├──► People Ops agent
             │       └──► People Ops RAG          3 passages
             ├──► Payroll agent
             │       └──► Payroll MCP tool        2 records
             └──► Policy Library agent
                     └──► Policy RAG              4 passages
                           (one of them superseded in April)
```

Four teams, three retrievals, one synthesis. The answer was wrong. **Where?**

The candidate failures are genuinely different problems with genuinely different owners:

- **Retrieval missed.** The correct passage exists and was not returned. Owner: whichever RAG. Fix: chunking, embedding, ranking, or the query.
- **Retrieval hit, generation ignored it.** The passage was there and the model answered from its own priors anyway. Fix: prompt, model, or grounding enforcement.
- **The source document is wrong or superseded.** The system faithfully returned bad content. Fix: the document, and the retirement process in Chapter 7.
- **Two sources disagreed.** Both were returned. The model picked one, arbitrarily and invisibly.
- **Entitlement filtering removed the right answer.** The system worked exactly as designed, and the user was told something incomplete without being told it was incomplete.
- **Synthesis broke it.** Every component returned correctly and the top-level agent combined them into a claim none of them made.

Without a trace spanning all of it, this is unanswerable. Teams will each check their own component, each find it working, and the user's complaint will die unresolved. **That pattern — everybody's component is fine and the system is broken — is the characteristic failure of a federated architecture, and it is fatal to trust.**

---

## 4.2 The Principle: Observability at the Hop, Quality at the Owner

Two rules, and they are separate on purpose.

> **Every hop in the mesh emits a span, correlated under one identifier, so that any answer can be reconstructed end to end by anyone investigating it.**

> **Quality is measured and owned where the knowledge lives. The team that owns the corpus owns its answer quality — nobody else can judge it.**

The first is a platform responsibility and must be uniform, because a trace with a gap is not a trace. The second cannot be centralised: a central quality team cannot tell whether an answer about supplier onboarding is correct. Only the supplier team can.

The consequence: **the platform provides the instrument, the teams provide the judgement.**

---

## 4.3 The Pattern: One Trace, Every Hop

### The span model

Every component emits spans under a single trace identifier propagated across every MCP and A2A call.

```text
 TRACE  4f2a9c...
 │
 ├─ span  assistant.handle_request            1,840 ms
 │   user=alice@corp   question="parental leave, part-time?"
 │   │
 │   ├─ span  a2a.call  people-ops-agent         520 ms
 │   │   │  caller=workplace-assistant  principal=alice
 │   │   ├─ span  retrieval.search               110 ms
 │   │   │    filter=region:EMEA,class:internal
 │   │   │    candidates=42  returned=3
 │   │   │    top_score=0.81  min_returned=0.62
 │   │   │    doc_ids=[hr-114, hr-119, hr-203]
 │   │   └─ span  generation                     390 ms
 │   │        model=<m>  in=2,140  out=180  grounded=true
 │   │
 │   ├─ span  a2a.call  policy-library-agent     610 ms
 │   │   ├─ span  retrieval.search                90 ms
 │   │   │    candidates=88  returned=4
 │   │   │    doc_ids=[pol-22, pol-31, pol-08*, pol-40]
 │   │   │    * pol-08 superseded_by=pol-77  ⚠ flagged
 │   │   └─ span  generation                     480 ms
 │   │
 │   └─ span  synthesis                          640 ms
 │        inputs=3  conflict_detected=true
 │        conflict=[hr-119 vs pol-08]
 │        resolution=highest_authority_source
```

Span names here are written for readability. In a real estate, use the OpenTelemetry semantic conventions for generative AI rather than inventing your own — the reasoning, and the naming, is in [Tracing Across the Agent Mesh](tracing-across-the-agent-mesh.md).

The investigation that was impossible in §4.1 now takes a couple of minutes. `pol-08` was superseded and still retrieved, and a conflict with `hr-119` was detected and resolved towards the wrong source. Two owners, two distinct fixes, both identifiable.

### What every span must carry

**Non-negotiable, emitted by platform middleware:**

| Field | Why |
| --- | --- |
| trace id, span id, parent span id | Reconstruction |
| service, team, environment | Who owns this hop |
| principal, delegation chain | Who this was for |
| latency | Where the time went |
| status and error | Where it broke |
| token usage, model, cost | Doubles as the cost record — [Chapter 5](../chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md) |

**Retrieval spans additionally:**

| Field | Why |
| --- | --- |
| query, and the rewritten query if rewriting occurred | The model often searches for something other than what the user asked |
| filters applied | Distinguishes "not found" from "filtered out" |
| candidates considered, results returned | A high candidate count with few returns points at ranking |
| score distribution | Uniformly low scores mean the corpus does not contain the answer |
| document and chunk identifiers | The evidence trail |
| freshness and supersession flags | Catches Chapter 7 failures at the point of use |

**Generation spans additionally:** model and version, prompt template version, whether output was grounded in retrieved content, and any safety filter activity.

> **Reference implementation.** OpenTelemetry, with context propagated in request headers across MCP and A2A hops, exported to the organisation's tracing backend — Cloud Trace, X-Ray, Application Insights, or a self-hosted collector. The model is portable; the backend is an implementation detail. Span naming conventions, propagation across protocol boundaries, sampling policy and PII handling are in [Tracing Across the Agent Mesh](tracing-across-the-agent-mesh.md).

### The rules that make traces usable

**Propagation is mandatory and automatic.** A component that drops trace context breaks the chain for everything below it. This belongs in the boilerplate from Chapter 3, not in team code, and conformance testing should verify it.

**Sample intelligently.** Trace every error, every low-confidence answer, every answer a user gave negative feedback on, and every conflict — plus a small percentage of normal traffic. Full tracing of everything is unaffordable and mostly useless; the interesting requests are not random.

**Never put retrieved content in spans.** Document identifiers only. Retrieved content is subject to the entitlements of the person who retrieved it, and a tracing backend is typically readable by operations staff who do not have those entitlements. **Tracing is a very common accidental route around access control.** Resolve identifiers to content only through an authorised lookup.

**Retention is bounded and classified.** Traces contain identity and query text. Queries can themselves be sensitive. Apply retention and access policy accordingly — and remember traces when a deletion request arrives ([Chapter 6](../chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md)).

---

## 4.4 Measuring Quality: The Evaluation Set

You cannot improve what you do not measure, and in RAG the measurement must separate retrieval from generation — because the fixes are completely different.

### The four metrics

Measured over a fixed evaluation set of representative questions with known-correct sources.

| Metric | Question it answers | When it drops, fix |
| --- | --- | --- |
| **Context recall** | Did retrieval return the passages needed? | Chunking, embedding, ranking, query rewriting |
| **Context precision** | Were the returned passages relevant, and ranked well? | Reranking, `top_k`, filters |
| **Faithfulness** | Is the answer supported by what was retrieved? | Prompt, grounding enforcement, model |
| **Answer relevance** | Does the answer address the question asked? | Prompt, query understanding |

The diagnostic power is in the combinations:

- **Low recall, high faithfulness** — the system is honestly answering from inadequate evidence. A retrieval problem.
- **High recall, low faithfulness** — the evidence was there and the model ignored it. A generation problem, and the more alarming of the two, because it is confidently wrong.
- **Everything high, users still unhappy** — the evaluation set does not resemble real questions. Rebuild it from actual traffic.

### The evaluation set is the asset

The set matters more than the metrics computed over it. Properties of a good one:

- **Derived from real questions**, taken from logs, not imagined by the team
- **Includes the hard cases** — ambiguous phrasing, questions spanning two documents, questions with no answer in the corpus
- **Includes questions that must be refused**, and checks that they are
- **Includes entitlement cases** — the same question from two principals, with different correct answers
- **Versioned in the team's repository**, reviewed like code
- **Grows when things go wrong.** Every incident adds a case. This is how the set stops being a snapshot and becomes a regression suite.

**A minimum viable set is small — of the order of fifty questions.** Teams delay because they imagine needing five hundred. Fifty real questions, honestly graded, will tell you more than any amount of intuition.

**The set exists to be run at defined moments, not occasionally.** At a minimum: before a capability is first exposed, so there is a baseline to argue from; after any change that alters what goes into the index — parser, chunking, embedding model, metadata schema — because each of those changes retrieval whether or not anyone intended it; after any material change to the corpus itself; and on a schedule, so that slow drift is visible without anyone having to suspect it. **A capability that has never been evaluated has no claim to being correct, and one evaluated only at launch has no claim to still being correct.**

### Gating deployment

> **A change to the live RAG — new chunking, new embedding model, new prompt, new retrieval parameters, a bulk corpus change — is a deployment, and it passes the evaluation gate or it does not ship.**

"Deploying to production" here does not mean shipping application code. It means **changing the system that is answering questions right now.** Re-indexing a corpus with different chunking changes every answer it gives, with no code review anywhere in sight. That is the change most likely to cause a regression and least likely to be governed, which is exactly why it must be gated.

The gate: run the evaluation set against a candidate index, compare against the current baseline, block on regression beyond a threshold, publish the comparison. Combined with Chapter 2's dual-index cutover, this makes even an embedding model migration a routine, reversible operation.

---

## 4.5 The Flywheel: Feedback That Actually Changes Something

Most feedback mechanisms collect signal and do nothing with it. A thumbs-down button with no downstream process is theatre, and users work out quickly that pressing it changes nothing.

> **Feedback is only worth collecting if there is a committed path from a signal to a change in the system.**

The loop, closed:

```text
   ┌───────────────────────────────────────────────────┐
   │                                                   │
   ▼                                                   │
 1 SIGNAL                                              │
   explicit ratings · corrections · escalation to      │
   a human · abandoned sessions · rephrasings ·        │
   conflict flags · low-score retrievals               │
   │                                                   │
   ▼                                                   │
 2 TRIAGE            ← the step everyone skips         │
   pull the trace. classify the failure:               │
     retrieval miss · generation ignored evidence ·    │
     source wrong · source superseded · conflict ·     │
     filtered by entitlement · out of scope            │
   │                                                   │
   ▼                                                   │
 3 ROUTE                                               │
   each class has ONE owner and ONE kind of fix        │
   │                                                   │
   ▼                                                   │
 4 ACT                                                 │
   re-chunk · tune ranking · fix the DOCUMENT ·        │
   retire the superseded source · adjust the prompt ·  │
   correct the entitlement metadata · teach the        │
   system to refuse                                    │
   │                                                   │
   ▼                                                   │
 5 CAPTURE                                             │
   add the case to the evaluation set  ───────────────►│
   so this specific failure can never silently return  │
                                                       │
 6 VERIFY ──────────────────────────────────────────────┘
   re-run the gate. did it fix the case without
   breaking the others?
```

**Triage is the step that makes the difference.** Without it, feedback is a pile of complaints of mixed causes that nobody can act on, and it accumulates until it is ignored. With it, every signal becomes a routed, owned item with a known kind of fix.

**Fixing the document is a legitimate outcome, and often the right one.** A meaningful share of "the RAG is wrong" reports are "the policy document is wrong, or out of date, or contradicts another document." The RAG has done something useful: it surfaced a content defect that had been sitting there unnoticed for two years. Route it to the document owner. Do not distort retrieval to compensate for bad source material.

**Grading the retrieval, not only the answer.** When a user says the answer was wrong, the trace already records which passages were returned. Ask the reviewer the sharper question — *were the right passages retrieved?* A yes/no there splits the failure into retrieval or generation immediately, and that single distinction is worth more than any amount of free-text complaint.

---

## 4.6 When Two Teams Disagree

A federated estate guarantees this eventually. People Ops says parental leave is sixteen weeks. The Policy Library says twelve. Both retrieved correctly from their own corpora. Both are, from their own perspective, right.

Today the synthesising agent picks one, silently, and the user has no idea a choice was made.

**Three things are needed, in order of importance.**

**1. Detect it.** The synthesising agent must compare claims across sources and flag contradictions rather than smoothing them into fluent prose. An undetected conflict is far more dangerous than a reported one, because it is indistinguishable from a confident correct answer.

**2. Resolve it by declared authority, not by chance.** The registry in [Chapter 10](../chapter-10-discovering-what-already-exists/chapter-10-discovering-what-already-exists.md) records which team is the source of truth for which subject. Parental leave entitlement: People Ops. Payment terms: Finance. Where authority is declared, resolution is deterministic and explainable.

**3. Surface it when authority does not settle it.** If both sources are authoritative in their own scope, or authority is undeclared, the answer says so: *"Sources disagree. People Ops states sixteen weeks; the Policy Library states twelve. The People Ops document is more recent."* **A user told that sources disagree is better served than a user given a confident guess** — and it turns an invisible quality problem into a visible content problem with an owner.

**And then fix the underlying content.** A detected conflict is a defect report against the corpus. It goes to the two owners with the trace attached. The registry's authority declarations are how the organisation stops producing the same conflict repeatedly.

---

## 4.7 Testing, Which Is Not the Same as Evaluation

Evaluation answers *are the answers good?* Testing answers *does the system behave as specified?* Teams routinely build the first and assume it covers the second. It does not, and the gap is where most production surprises live.

> **A non-deterministic system still has deterministic parts, and those are the parts that break. Test them the ordinary way.**

### The layers

| Layer | Tests what | Deterministic? | Runs |
| --- | --- | --- | --- |
| **Unit** | Chunking boundaries, metadata extraction, filter construction, citation formatting | Yes | Every commit |
| **Ingestion regression** | A fixed sample corpus produces the expected chunk count, fields and provenance | Yes | Every pipeline change |
| **Index regression** | A rebuilt index returns the same documents for the same queries as the one it replaces | Yes | Every rebuild |
| **Contract** | The interface still honours its published schema and error semantics | Yes | Every deploy, both sides |
| **Security conformance** | Chapter 3's suite — token, entitlement, scope, delegation | Yes | Every deploy |
| **Evaluation** | Answer quality against the four metrics | No | The deployment gate |
| **Adversarial** | The red-team payload corpus (Chapter 6) | Partly | Every model or prompt change |
| **Load** | Behaviour at and beyond expected concurrency | Yes | Before launch, quarterly after |

**Index regression is the one almost nobody builds, and the one that catches the most.** A re-embedding, a chunking-parameter change or a pipeline upgrade silently alters what retrieves. Run the evaluation set's queries against the old and new indexes and diff the returned document sets. A change is not necessarily a regression — but an *unexplained* change is, and without the diff you will not know it happened until a user does.

**Contract testing has to run on both sides.** The provider verifies it still meets the published contract; each registered consumer verifies its expectations still hold. Chapter 10's consumer list is what makes the second half possible — and it is why versioning in Chapter 3 is enforceable rather than aspirational.

**Load testing a RAG is not load testing a web service.** Three things behave differently: the vector index degrades non-linearly past a concurrency threshold rather than gracefully; model provider rate limits produce throttling that looks like a bug; and cost scales with load, so a load test spends real money and should be budgeted and bounded. Test the retrieval path at full rate and the generation path at a sampled rate.

### Handling non-determinism

The same question may legitimately produce different text twice. That does not make testing impossible; it changes what you assert.

- **Assert on structure, not prose.** Citations present, correct document identifiers, required fields populated, no refusal.
- **Assert on retrieval, which *is* deterministic.** Given a fixed index and a fixed query, the returned set should be stable. Most regressions are retrieval regressions, and retrieval can be tested exactly.
- **Assert on distributions, not single runs.** Run the sample *n* times; require the pass rate to hold. A flaky answer that passes eight times in ten is information, not noise.
- **Pin everything pinnable** — model version, prompt template version, judge version, index version. An unpinned dependency turns every test failure into an investigation of the test.

### Test data

Testing needs a corpus that is realistic and safe, which production content usually is not. Chapter 6 §6.6 covers the environment and test-data strategy; the requirement it must satisfy is this one. **A team that cannot test without production data will test in production**, and then the evaluation gate is the only control standing between a bad change and a user.

---

## 4.8 What the Platform Team Provides

| Platform provides | Team owns |
| --- | --- |
| Tracing instrumentation in the boilerplate; automatic propagation | Nothing — this must not be optional |
| The trace backend, retention policy, and an answer-investigation view | Investigating their own hops |
| The evaluation harness and the four metrics, runnable in CI | Their evaluation set and its questions |
| Judge-model hosting, versioning and cost control | Interpreting scores in their domain |
| The deployment gate, wired into the pipeline | Their thresholds, and the decision to ship |
| Feedback capture widgets and the triage queue | Triaging and acting on their own signals |
| Conflict detection in the synthesis boilerplate | Declaring authority; resolving content conflicts |
| The test harness — unit, ingestion, index-diff, contract and load — wired into CI | Writing tests for their own corpus and interface |
| Contract test execution against every registered consumer | Keeping their contract tests current |
| Quality dashboards per RAG, aggregated for the estate | The trend line for their own RAG |

**A note on the judge model.** Faithfulness and relevance are typically scored by an LLM. That costs money, and the cost is easy to overlook because it is not user-facing. Pin the judge's version — an unpinned judge makes your quality trend meaningless, because you cannot tell a change in the system from a change in the grader. This cost belongs in Chapter 5.

---

## Where this leaves you

Quality is now measurable, failures are locatable, feedback changes something, and disagreements are surfaced rather than guessed at.

All of which costs money, and the bill has a structural problem.

The retrieval that People Ops performed for Alice's question was paid for by People Ops. So was the generation. But Alice was using the workplace assistant, owned by a different team, and the Policy Library agent also ran, billed to a third. **Three teams paid for one user's question, none of them chose to spend it, and none of them can explain the line item.** Now multiply that across an agent mesh where a single request can fan out through eight hops.

Who pays?

---

**Supporting material for this chapter**

- [Tracing Across the Agent Mesh](tracing-across-the-agent-mesh.md) — span conventions, context propagation across MCP and A2A, sampling, PII handling, and the investigation runbook

---

[← Chapter 3 — Exposing a RAG Securely](../chapter-3-exposing-a-rag-securely/chapter-3-exposing-a-rag-securely.md) | [Chapter 5 — Cost Attribution and Chargeback →](../chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md)
