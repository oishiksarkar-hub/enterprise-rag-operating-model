# Tracing Across the Agent Mesh

> Supporting material for [Chapter 4 — Proving Retrieval Quality](chapter-4-proving-retrieval-quality.md)
>
> How to instrument a federated RAG estate so that any answer can be reconstructed end to end: span conventions, context propagation across MCP and A2A, sampling, what must never be recorded, and the runbook for investigating a bad answer.

---

## 1. Why this is harder than normal service tracing

Tracing microservices is well-understood. Three things make an agent mesh different.

**The call graph is decided at runtime by a model.** You cannot draw it in advance. The same question asked twice may take different paths. Traces are therefore not just useful for debugging — they are the only record of what the system actually did.

**Hops cross organisational boundaries.** Team A's agent calls Team B's. Propagation depends on a team you do not control honouring a convention. One team that drops context blinds everyone below it.

**The payload is the sensitive part.** In conventional tracing you can often log request bodies. Here, retrieved content is entitlement-controlled, and query text can be sensitive in itself. Tracing becomes a potential access-control bypass.

---

## 2. Span conventions

Uniform naming is what makes cross-team queries possible. Without it you have per-team traces, not estate traces.

```text
  agent.request            top-level handling of a user request
  a2a.call                 outbound call to another team's agent
  a2a.serve                inbound handling of an A2A request
  mcp.call                 outbound MCP tool invocation
  mcp.serve                inbound MCP tool handling
  retrieval.search         a query against an index
  retrieval.rerank         reranking stage
  generation               a model call producing text
  synthesis                combining multiple sources into one answer
  ingestion.<stage>        pipeline stages (extract, chunk, embed, index)
```

> **Adopt the standard names where a standard exists.** The above is illustrative, and the names are chosen for readability in this text. In a real estate, prefer the OpenTelemetry semantic conventions for generative AI, which already define operation names for the common cases — among them `invoke_agent` for an agent handling a request, `execute_tool` for a tool invocation, `embeddings` for an embedding call, and a span naming pattern of `{operation} {target}` such as `execute_tool search_policies`. Retrieval spans follow the database conventions rather than the GenAI ones.
>
> The reason to care is not conformance for its own sake. It is that agent frameworks, model SDKs and observability backends are converging on these names, which means a standard-named span arrives pre-parsed in your backend and a custom-named one does not. Where no standard attribute exists — `service.team`, `rag.id`, `request.cost_centre` — use your own, in your own namespace, and document it.
>
> These conventions are still moving. Expect attribute names to change between releases, and pin the version of the convention you are instrumenting against so that a rename is a planned migration rather than a silent break in your dashboards.

### Required attributes on every span

```text
  service.name             logical service
  service.team             owning team            ← attribution
  service.environment      dev | test | prod
  rag.id                   which RAG, if applicable
  principal.subject        ultimate principal (user or workload)
  principal.type           user | workload | delegated
  principal.chain          ordered list of intermediaries
  request.cost_centre      resolved from principal  ← Chapter 5
```

`service.team` and `request.cost_centre` are what let the same trace data serve both the investigation in Chapter 4 and the showback in Chapter 5. Instrument once, use twice.

### Retrieval span attributes

```text
  retrieval.query_original     as the user expressed it
  retrieval.query_executed     after rewriting/expansion
  retrieval.filters            entitlement + explicit filters
  retrieval.candidates         considered before ranking
  retrieval.returned           returned after ranking and cutoff
  retrieval.top_score
  retrieval.min_returned_score
  retrieval.doc_ids            identifiers ONLY
  retrieval.chunk_ids
  retrieval.stale_flags        docs past freshness SLO
  retrieval.superseded_flags   docs with a successor
  retrieval.index_version
  retrieval.embedding_model    model + version
```

Recording both the original and executed query matters more than it seems. A large share of unexplained retrieval misses turn out to be the rewriting step, not the index — the model searched for something other than what the user asked.

`candidates` versus `returned` separates two very different failures: few candidates means the corpus or the filter is the problem; many candidates and poor results means ranking is.

### Generation span attributes

```text
  gen.model, gen.model_version, gen.prompt_template_version
  gen.tokens_in, gen.tokens_out, gen.cached_tokens
  gen.grounded                 answer constrained to context
  gen.citations_returned
  gen.refused                  and gen.refusal_reason
  gen.safety_filtered
  gen.estimated_cost
```

`gen.prompt_template_version` is the attribute teams omit and then need. When quality shifts on a Tuesday, the first question is what changed, and a prompt edit is the most common answer and the least likely to be in a release note.

**Record the provider separately from the model.** The same model name is frequently served through more than one route — a first-party endpoint, a cloud provider's hosted version, an internal gateway — with different pricing, different quotas and different failure behaviour. If you record only the model, you cannot tell those apart when one of them degrades. The OpenTelemetry conventions carry a dedicated provider attribute for exactly this reason; use it, and treat provider-plus-model as the unit of analysis rather than model alone.

> **The token double-count trap.** Once an agent framework, a model SDK and a gateway are all instrumented, the same call can emit token counts at three layers. Summing spans naively then produces a cost figure that is two or three times reality — and, worse, produces it consistently, so it looks credible.
>
> Decide once which layer is authoritative for billing-grade token counts. The gateway is usually the right answer, because it sees every call including the ones a framework does not know it made. Mark that layer explicitly, and have the showback in Chapter 5 read only from it. Keep the other layers' counts for diagnosis, but never let them into a cost total.

### Synthesis span attributes

```text
  synth.input_sources
  synth.conflict_detected
  synth.conflicts              pairs of contradicting sources
  synth.resolution_strategy    authority | recency | surfaced_to_user
  synth.sources_used           which inputs influenced the output
```

---

## 3. Propagating context across boundaries

### The mechanism

W3C Trace Context headers, on every outbound call, in both directions:

```text
  traceparent: 00-4f2a9c...-b7e1...-01
  tracestate:  team=people-ops,chain_depth=2
```

For MCP and A2A, carry them as transport headers, not inside the message body. Body-level propagation breaks the moment any intermediary re-serialises the message — which something always eventually does.

### Making it non-optional

Propagation cannot be a convention teams are asked to follow. Three enforcement layers:

1. **The boilerplate from Chapter 3 propagates automatically.** Outbound clients inject; inbound middleware extracts. A team writing an ordinary handler gets this for free and cannot easily lose it.
2. **Conformance testing verifies it.** Chapter 3's suite asserts that an inbound trace id appears on outbound calls. Failing means failing registration.
3. **Gap detection runs continuously.** A monitor looks for spans whose declared parent never appears. A team producing orphans gets a ticket, not a silent blind spot.

### Delegation chain and depth

`tracestate` carries chain depth, which serves two purposes: enforcing the depth limit from Chapter 3, and making runaway fan-out visible. A request that touches thirty agents is either a very unusual question or a loop, and either way someone should see it.

---

## 4. What must never enter a span

> **Retrieved content is entitlement-controlled. A tracing backend is not.**

This is the single most common accidental access-control bypass in these systems. Operations staff who legitimately need trace access do not thereby acquire the entitlements of every user whose requests they can see.

**Never record:**

- Retrieved chunk text or document content
- Full generated answers
- Personal data appearing in queries or results
- Tokens, credentials or authorisation headers
- Raw embeddings

**Record instead:** identifiers, counts, scores, flags, model metadata, timings.

**Query text is a judgement call.** It is enormously useful for diagnosis and can itself be sensitive — a query naming an individual under investigation, or revealing a medical concern. Options, in order of preference:

1. Store query text in a **separate, access-controlled store**, referenced from the span by identifier. Tracing shows the shape; only authorised investigators resolve the text.
2. Store a **redacted** query with detected entities masked.
3. Store a **hash**, allowing "this exact query failed forty times" without revealing it.

Pick one deliberately and document it. The default of "log the query" is a decision too, just an unexamined one.

**Content resolution for investigation** goes through an authorised lookup: given a trace, an entitled investigator can retrieve what was returned, and that retrieval is itself audited. Investigating an incident should leave a record.

---

## 5. Sampling

Full tracing of every request is unaffordable and mostly uninformative. Sample on interest, not at random.

**Always trace, at 100%:**

- Any request that errored
- Any request with negative user feedback
- Any request where a conflict was detected
- Any retrieval whose top score fell below a floor
- Any request where a tool call was refused on authorisation
- Any request on a restricted tool
- Any request exceeding a latency or cost threshold
- All traffic in dev and test

**Sample at a low rate:** ordinary successful production traffic, for baselines and trend lines.

**Retroactive tracing.** Buffer spans briefly at the edge and decide whether to keep once the outcome is known. This is what lets you keep every trace with negative feedback even though feedback arrives after the request has completed. Without it, the most valuable traces are exactly the ones you discarded.

---

## 6. Retention and deletion

| Data | Typical retention | Notes |
| --- | --- | --- |
| Span structure, timings, ids | 30–90 days | Cheap, high diagnostic value |
| Query text store | Shorter, policy-driven | Classify as user-generated content |
| Traces attached to incidents | Incident lifetime + audit period | Exempt from routine expiry |
| Aggregated metrics | Long | No personal data |

**Deletion requests reach traces.** A right-to-be-forgotten request must remove the subject's identity and any personal data in query text from the trace store, as well as from chunks, vectors, caches and feedback logs. Traces are the component teams forget, and the one an auditor will ask about. See [Chapter 6](../chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md).

Design for this by keeping principal identity as a stable pseudonymous identifier resolvable through a separate mapping. Deleting the mapping entry severs the link across every trace at once, without rewriting trace storage.

---

## 7. The investigation runbook

A repeatable procedure for "the assistant said something wrong."

**1. Get the trace id.** Every user-facing answer carries one — surfaced in the interface, or recoverable from a session identifier. If users cannot give you a trace id, fix that first; everything else depends on it.

**2. Read the shape before the detail.** Which agents were called? Any errors? Any refusals? Unusual depth or fan-out? Often the anomaly is visible in the structure alone.

**3. Check for detected conflicts.** If `synth.conflict_detected` is true, you likely have your answer, and it is a content problem with two owners.

**4. Check retrieval before generation.** For each retrieval span:
   - Was `retrieval.returned` zero or very low? Nothing was found.
   - Were scores uniformly low? The corpus probably lacks the answer.
   - Do `stale_flags` or `superseded_flags` appear? A Chapter 7 failure at the point of use.
   - Did `query_executed` differ materially from `query_original`? Suspect the rewriting step.
   - Were filters unexpectedly narrow? The answer may have been correctly withheld — which is a different conversation with the user, not a bug.

**5. Then check generation.** Was `grounded` true? Did citations come back? If the right content was retrieved and ignored, this is a generation defect and the more serious class.

**6. Resolve identifiers to content** through the authorised lookup, and read what was actually returned. Very often this ends the investigation immediately: the passage says exactly what the user complained about, and the document is the problem.

**7. Classify and route** using Chapter 4's triage categories.

**8. Add the case to the evaluation set.** Non-optional. An incident that does not become a test case will recur.

---

## 8. Estate-level views the platform should provide

Traces are per-request. Leadership and platform teams need the aggregate.

- **Answer path map** — which agents call which, derived from actual traces rather than documentation. It will not match anyone's architecture diagram, and that difference is itself the finding.
- **Hop latency breakdown** — where time is spent across the mesh; usually a small number of hops dominate.
- **Orphan span monitor** — teams not propagating context.
- **Conflict rate by source pair** — which corpora habitually contradict each other. A persistently high pair is a content-governance problem, not a technical one.
- **Refusal and authorisation-failure rate** — spikes indicate a misconfigured entitlement or an agent attempting things it should not.
- **Stale-document retrieval rate** — feeds the freshness SLO in [Chapter 7](../chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md).
- **Cost per trace, by originating team** — the direct input to [Chapter 5](../chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md).

---

[← Back to Chapter 4](chapter-4-proving-retrieval-quality.md)
