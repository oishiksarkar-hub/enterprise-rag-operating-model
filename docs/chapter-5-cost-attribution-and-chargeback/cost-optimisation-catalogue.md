# The Cost Optimisation Catalogue

> Supporting material for [Chapter 5 — Cost Attribution and Chargeback](chapter-5-cost-attribution-and-chargeback.md)
>
> Every lever available in an agentic RAG estate, organised by where it applies, with an honest note on what each one costs in quality. Optimisation is not one activity in one place — it runs from ingestion through to the interface.

---

## How to use this

Two rules before any of it.

**Measure first.** Optimising the wrong stage is wasted effort. Use the cost breakdown from Chapter 5 to find the dominant line. In almost every estate that line is **generation input tokens**, which is why the retrieval-side levers matter more than they appear to.

**Quality is the constraint, not the objective.** Every lever below has a quality cost, stated. A saving that degrades answers is a false economy that will surface as a trust problem six months later. Re-run the evaluation gate from [Chapter 4](../chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) after any change — that is what the gate is for.

Levers are marked ★ where the saving-to-effort ratio is unusually good.

---

## 1. At ingestion

Costs here are periodic rather than per-request, but they are lumpy and easy to leave running.

**Ingest less.** ★ The cheapest chunk is the one never created. Exclude drafts, archives, duplicates, boilerplate-only documents and near-identical versions. Corpora routinely shrink 20–40% with no loss of answerable content — and retrieval quality *improves*, because there is less noise to rank through.
*Quality cost: none, if exclusions are chosen carefully. Review them.*

**Incremental over full re-crawl.** ★ Re-embedding an unchanged corpus nightly is pure waste. Change detection with content hashing means only modified documents are reprocessed.
*Quality cost: none. This is strictly better.*

**Right-size the embedding model.** Higher dimensions cost more to generate, store and search. Evaluate whether a smaller model meets your recall target on *your* corpus — frequently it does, and the difference is invisible to users.
*Quality cost: possible recall reduction. Measure before committing; re-embedding to undo it is expensive.*

**Batch embedding.** Offline pricing is materially cheaper than online. Ingestion is not latency-sensitive.
*Quality cost: none.*

**Tune chunk size against token economics.** Smaller chunks mean more of them but more precise retrieval, so fewer tokens reach the model. Larger chunks mean the opposite. The optimum is usually smaller than teams' first guess.
*Quality cost: varies by corpus. A/B against the evaluation set.*

**Deduplicate before embedding.** Near-identical chunks cost the same to embed and store while crowding out variety in results.
*Quality cost: none. Improves precision.*

---

## 2. At the index

**Retire on a schedule.** ★ Indexes grow monotonically unless something removes content. TTL and supersession rules ([Chapter 7](../chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md)) reduce storage, improve search latency and improve quality by removing stale material.
*Quality cost: negative — it improves.*

**Right-size replicas.** Provisioned for a launch spike and never revisited is the single most common invisible cost in retrieval infrastructure. Review quarterly against actual query rate.
*Quality cost: none, if headroom is kept.*

**Tier cold content.** Rarely-queried historical corpora can sit in cheaper storage with higher retrieval latency.
*Quality cost: latency on infrequent queries.*

**Reduce stored metadata.** Every field is stored per chunk. Fields nobody filters, sorts or displays are pure overhead at corpus scale.
*Quality cost: none, if genuinely unused. Confirm from trace data, not assumption.*

**Consolidate under-used indexes.** A team running six indexes each with a minimum footprint is paying six minimums. Consolidation with a source filter is often cheaper.
*Quality cost: slight precision loss; usually recoverable with filters.*

---

## 3. At retrieval

**Rerank instead of a large `top_k`.** ★★ The highest-value lever in most estates. Retrieving forty chunks and sending them all to generation is expensive *and worse* — the model must ignore most of them. Retrieve forty, rerank, send five.
*Quality cost: negative — it improves. Do this one first.*

> **"Rerank cheaply" is an assumption, not a given.** Reranking is a second model pass over every candidate, and whether it is cheap depends entirely on what you rerank with. A small local cross-encoder is genuinely inexpensive. A hosted ranking API is a per-call charge that may not be published anywhere convenient, and reranking forty candidates on every query is a per-query cost that scales with traffic. Using a general-purpose model as a reranker is the expensive option dressed as a clever one.
>
> The lever still works, because the input tokens you avoid at generation usually cost more than the reranking. But *usually* is doing real work in that sentence. Price your reranker against your own traffic before claiming the saving, and put the reranking cost on the same dashboard as the generation cost so the comparison stays visible.

**Cut `top_k` to what is actually used.** Trace data shows which retrieved chunks influenced the answer. Typically only the top few. Everything below is paid-for input tokens with no effect.
*Quality cost: low if guided by measurement, real if guessed.*

**Cache retrieval results.** ★ Enterprise query distributions are heavily skewed — a small set of questions accounts for a large share of traffic. Cache keyed on the normalised query **and the entitlement filter**.
*Quality cost: staleness, bounded by cache TTL. The entitlement key is mandatory — a cache shared across principals is a disclosure.*

**Hybrid instead of pure vector, where it suits.** Keyword matching is far cheaper than embedding a query, and better for exact identifiers and product codes.
*Quality cost: none. Usually improves.*

**Cache query embeddings.** Repeated queries need not be re-embedded.
*Quality cost: none.*

**Classify scope before retrieving.** ★ A cheap classifier that identifies out-of-scope questions avoids a full retrieve-and-generate cycle that was going to produce a poor answer anyway.
*Quality cost: negative — refusing clearly is better than answering badly. Watch the false-refusal rate.*

---

## 4. At generation — where the money is

**Prompt caching.** ★★ Usually the largest single available saving, and usually unexploited. System prompts, tool definitions and stable instructions repeat across nearly every request, and providers charge a small fraction for cached prefixes.

Structure prompts so the stable portion comes first:

```text
  ┌─ STABLE (cached) ──────────────────┐
  │  system instructions               │
  │  tool definitions                  │
  │  few-shot examples                 │
  │  domain guidance                   │
  ├─ VARIABLE (full price) ────────────┤
  │  retrieved context                 │
  │  conversation history              │
  │  the user's question               │
  └────────────────────────────────────┘
```

*Quality cost: none. This is structural.*

> **Two things about caching that surprise people.**
>
> **There is a minimum size, and prompts below it cache nothing.** Providers impose a floor on the cacheable prefix — a minimum token count below which the request is simply billed at full price. The floor varies by model and by provider, and it is high enough that a short system prompt will not clear it. If your stable prefix is small, the lever does not exist for you yet; the fix is to establish whether you are above the threshold before designing around the saving.
>
> **Explicit caching is not free while it sits there.** Where the cache is implicit — the provider detects a repeated prefix automatically — you pay only the discounted input rate. Where you create and hold a cache explicitly, you also pay for the *storage* of those tokens, charged per unit of time held. That changes the arithmetic completely: the saving is proportional to how often you hit the cache, but the storage cost accrues whether anyone queries or not. A cache held for a workload with low traffic can cost more than it saves.
>
> The rule that follows: explicit caching pays off for high-frequency, high-volume prefixes and loses money on everything else. Measure hit rate against hold time before turning it on, and turn it off for workloads that went quiet.

**Model tiering.** ★ Match model strength to task:

| Task | Model class |
| --- | --- |
| Query rewriting, expansion | Small |
| Scope / intent classification | Small |
| Routing between agents | Small |
| Entity and metadata extraction | Small |
| Reranking | Purpose-built reranker |
| Final synthesis | Strong |
| Evaluation judging | Mid to strong, version-pinned |

*Quality cost: none if the split is chosen well. Evaluate each substitution independently.*

**Compress retrieved context.** Summarise or extract only relevant passages before generation.
*Quality cost: real — compression discards detail, and the discarded part is sometimes what mattered. Use where fidelity permits, not on regulatory content.*

**Cap and summarise conversation history.** Naively resending full history means cost grows quadratically with turn count. Summarise beyond a window.
*Quality cost: loss of early detail. Keep an explicit summary rather than truncating silently.*

**Constrain output length.** Output tokens are priced higher than input — commonly several times higher, and the multiple varies substantially between models and between tiers of the same model family. Do not carry a single ratio in your head across an estate that uses more than one model; check it per model, because the gap between the cheapest and most expensive ratio you are running is usually wider than people assume, and it changes which workload is worth optimising.

Where a model charges a premium for extended reasoning, those tokens are billed as output too, which can make a "thinking" model far more expensive per answer than its headline rate suggests. Verbose answers cost more and are frequently less useful.
*Quality cost: none if limits are sensible; truncation if too tight.*

**Stream, and allow early termination.** Users who have their answer can stop generation.
*Quality cost: none. Improves perceived latency.*

**Batch non-interactive work.** ★ Evaluation runs, bulk classification, backfills — all substantially cheaper at batch pricing.
*Quality cost: none. Latency only.*

**Cache full responses for identical requests.** Same question, same entitlement, same corpus version.
*Quality cost: staleness. Invalidate on index update; key on entitlement.*

---

## 5. Across the agent mesh

Mesh-level waste is invisible from any single component and often exceeds everything above.

**Route rather than fan out.** ★★ An orchestrator that calls every available agent on every request multiplies cost by the number of agents. A cheap routing step that selects the one or two relevant agents is dramatically cheaper *and* produces better answers, because there is less irrelevant material to synthesise.
*Quality cost: routing errors. Trace them; add missed cases to the evaluation set.*

**Bound delegation depth.** Chapter 3's depth limit is a cost control as much as a security one. Each hop adds a full retrieve-and-generate cycle.
*Quality cost: none at sensible depths.*

**Call tools, not agents, where reasoning is not needed.** An A2A call incurs the remote team's generation cost. If you only need the passages, call the MCP tool and synthesise once.
*Quality cost: you take on the domain reasoning. See Chapter 3 §3.2 on where judgement should live.*

**Detect and stop loops.** Two agents that each defer to the other will burn budget until something intervenes. Loop detection is a cost control with a very short payback period.
*Quality cost: none.*

**Deduplicate fan-out.** Three agents independently calling the same fourth agent within one trace should not pay three times. Cache within the trace.
*Quality cost: none.*

**Parallelise, do not serialise.** No direct cost saving, but shorter traces mean less accumulated context and fewer timeout retries — and retries are paid for twice.
*Quality cost: none.*

---

## 6. At the gateway

**Shared cache across teams.** ★ Common prefixes — organisational guardrail instructions, shared tool definitions — cached once for everyone rather than per team.
*Quality cost: none.*

**Provider arbitrage.** Route logical model names to whichever provider or region is most economical for the required capability.
*Quality cost: subtle behavioural differences between providers. Evaluate before routing production traffic.*

**Committed-use and provisioned throughput.** Predictable baseline load is cheaper under commitment than on-demand.
*Quality cost: none. Risk is over-commitment; forecast conservatively.*

**Quota enforcement as loss prevention.** A runaway agent with no quota can spend a month's budget overnight. Per-principal quotas are the cheapest insurance available.
*Quality cost: legitimate workloads throttled if limits are too tight. Alert before enforcing.*

**Reject early.** Requests failing authorisation, exceeding size limits or missing required context should be rejected at the gateway, before any model call.
*Quality cost: none.*

---

## 7. In governance

Governance cost is worth paying. It is still worth sizing.

**Tier by data classification.** ★ Match evaluation cadence, tracing depth and guardrail intensity to risk. Public marketing content does not warrant the regime applied to regulatory guidance.
*Quality cost: none, if the tiering is honest.*

**Keep the evaluation set tight.** Fifty well-chosen questions beat five hundred redundant ones and cost a tenth as much to run. Prune questions that never discriminate between candidates.
*Quality cost: none if pruning is evidence-based.*

**Do not judge every dimension every time.** Full four-metric judging on every gate is expensive. Run cheap deterministic retrieval metrics — recall, precision — on every change, and the expensive LLM-judged metrics on a schedule and before significant releases.
*Quality cost: slower detection of faithfulness regressions. Acceptable if retrieval metrics are continuous.*

**Sample traces on interest, not at random.** Chapter 4's sampling policy is a cost lever as well as a signal-quality one.
*Quality cost: none. Improves signal.*

**Tier trace retention.** Span structure retained longer, query-text stores retained briefly.
*Quality cost: reduced retrospective investigation depth.*

**Reduce canary frequency on stable corpora.** Continuous canary queries ([Chapter 7](../chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md)) cost real money. Frequency should track rate of change, not be uniform.
*Quality cost: slower detection on corpora that were not changing anyway.*

---

## 8. A review cadence

Optimisation is not a project. Make it routine.

**Weekly, automated:** anomaly alerts on spend per cost centre; top-five movers.

**Monthly, per team:** cost per answer trend; the three largest lines; the one lever with the best return this month.

**Quarterly, platform-wide:** infrastructure right-sizing; commitment coverage; prompt-cache hit rate across the estate; corpora that have grown without a corresponding increase in value; the governance line as a share of total.

**On every material change:** re-run the evaluation gate. **Cost and quality are measured together or the number is meaningless** — a cheaper system that answers worse has not been optimised, it has been degraded.

---

[← Back to Chapter 5](chapter-5-cost-attribution-and-chargeback.md)
