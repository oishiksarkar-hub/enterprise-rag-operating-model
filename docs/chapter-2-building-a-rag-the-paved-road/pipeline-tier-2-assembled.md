# Tier 2 — Assembled

> Supporting material for [Chapter 2 — Building a RAG: The Paved Road](chapter-2-building-a-rag-the-paved-road.md)
>
> One of three tier documents. [Tier 1 — Managed](pipeline-tier-1-managed.md) · [Tier 2 — Assembled](pipeline-tier-2-assembled.md) · [Tier 3 — Bespoke](pipeline-tier-3-bespoke.md). All three follow the same eight sections so they can be read side by side.

---

## 1. When this tier is the right choice

Tier 2 is a deeply parameterised pipeline, built and versioned by the platform team, configured and run by the product team inside its own boundary. The team gets the knobs without the maintenance burden.

**Choose it when Tier 1 has been disqualified for a stated, tested reason** — structure-critical content, domain-specific ranking, mixed semantic and structured retrieval, an unsupported deployment target, or a published limit genuinely below your numbers.

**It is disqualified when** the requirement is not a configuration problem at all: a fundamentally different retrieval paradigm, a research workload, or a constraint no parameter can express. That is Tier 3, and it should be rare.

**The critical property of this tier is that it must be credible.** An under-specified Tier 2 is the fastest route to every demanding team abandoning the platform for Tier 3, and a Tier 3 population that grows is a platform product failure rather than a compliance failure. Three commitments define it, and breaking any one of them empties the tier:

**The team never maintains pipeline code.** They own configuration and the corpus. The platform owns the implementation, its dependencies, its patching and its upgrade path. The moment a team is forking the pipeline to get something done, Tier 2 has failed and should be fixed rather than tolerated.

**The pipeline runs inside the team's boundary.** Their project, their subscription, their namespace, their keys. Data does not transit a central platform-owned system. This is what makes the tier acceptable to teams with residency or isolation constraints.

**The output conforms to the same interface contract as Tier 1.** Same governed surface, same identity model, same telemetry shape. Consumers cannot tell which tier they are talking to.

Distribute it as a versioned library and a set of infrastructure modules — not as a hosted service, and not as a template to copy. A copied template is a fork on a delay.

---

## 2. What you configure

The pipeline is driven by a declarative configuration file held in the team's own repository. Everything below is a parameter, not a code change. The examples are illustrative shapes, not a schema to implement literally.

### Sources

```yaml
sources:
  - id: policy-library
    type: object-storage
    location: <bucket-or-share>
    include: ["**/*.pdf", "**/*.docx"]
    exclude: ["**/draft/**", "**/archive/**"]
    refresh: { mode: incremental, schedule: "0 2 * * *" }
    permissions: { mode: inherit-from-source }

  - id: engineering-wiki
    type: wiki
    space: ENG
    refresh: { mode: event-driven, topic: wiki-changes }
    permissions: { mode: static, groups: ["eng-all"] }
```

Required capabilities: multiple heterogeneous sources in one index; include and exclude patterns; per-source refresh policy; per-source permission model; event-driven as well as scheduled refresh.

### Extraction

```yaml
extraction:
  ocr: { enabled: true, languages: [en, de] }
  tables: preserve-structure        # preserve-structure | markdown | flatten
  layout: multi-column-aware
  suppress: [headers, footers, navigation]
  images: { extract_captions: true, describe: false }
  on_failure: quarantine            # quarantine | skip | fail-run
```

`on_failure: quarantine` should be the default. A document that cannot be extracted must be visible in the ingestion report, not silently absent.

### Chunking

This is where a thin pipeline is exposed. A single fixed-size strategy is not sufficient for the cases that reach this tier.

| Strategy | Use for |
| --- | --- |
| `fixed` | Uniform prose; simple and predictable |
| `recursive` | General default; respects paragraph and section boundaries |
| `structural` | Documents with meaningful hierarchy — specs, contracts, standards |
| `semantic` | Content where topic shifts matter more than length |
| `code-aware` | Source code; splits on function and class boundaries |
| `parent-child` | Retrieve small for precision, return large for context |
| `custom` | A team-supplied function, registered against a platform interface |

```yaml
chunking:
  strategy: parent-child
  child: { size: 400, overlap: 50 }
  parent: { size: 2000 }
  preserve: [section_path, document_title, effective_date]
  min_chunk_tokens: 50
```

`preserve` matters more than it looks. Carrying the section path and document title into each chunk is a cheap and disproportionately effective improvement to both retrieval and citation quality.

`custom` is the pressure valve. A registered extension point means a team with a genuinely unusual corpus can stay on Tier 2 rather than escaping to Tier 3 over a single requirement.

### Embedding

```yaml
embedding:
  model: <approved-model-id>
  version: <pinned-version>   # pinned; never floating
  dimensions: 768
  batch_size: 100
  task_type: retrieval_document
```

The version must be pinned. Floating versions mean the index degrades from a change you did not make and cannot see. Check the model's maximum input length before designing chunk sizes around it — the constraint runs in that direction, not the other.

### Index and retrieval

```yaml
index:
  type: hybrid              # vector | keyword | hybrid
  metadata_schema:
    - { field: department,     type: string,  filterable: true }
    - { field: effective_date, type: date,    filterable: true, sortable: true }
    - { field: classification, type: string,  filterable: true, required: true }
    - { field: superseded_by,  type: string }

retrieval:
  mode: hybrid
  top_k: 20
  rerank: { enabled: true, model: <reranker>, top_n: 5 }
  filters_required: [classification]
  boost:
    - { field: effective_date, function: recency, half_life_days: 365 }
```

Hybrid retrieval should be the default. Pure vector search reliably underperforms on exact identifiers, product codes and acronyms — precisely the terms enterprise users type.

`filters_required` is a safety mechanism: a query that omits a required filter is rejected rather than silently returning everything.

### Quality gates

```yaml
quality_gates:
  reject_empty: true
  reject_duplicate: { enabled: true, similarity_threshold: 0.98 }
  reject_boilerplate_ratio: 0.8
  min_extraction_confidence: 0.7
  alert_on_skip_rate: 0.05
```

Refusing to ingest a bad chunk is cheaper than explaining a bad answer.

### Retention and deletion

```yaml
retention:
  tombstone_on_source_delete: true
  purge_after_days: 30
  ttl:
    - { source: policy-library, field: effective_date, max_age_days: 1095 }
  deletion_propagation_sla_hours: 24
```

The mechanics of tombstoning versus hard purge are in [Chapter 7](../chapter-7-keeping-knowledge-fresh/chapter-7-keeping-knowledge-fresh.md); the deletion-request obligations are in [Chapter 6](../chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md).

---

## 3. What you still build yourself

Less than Tier 3, more than Tier 1, and mostly at the edges of the pipeline rather than inside it.

- **The configuration itself**, and the judgement behind every value in it
- **Domain metadata** — the fields in `metadata_schema` are your vocabulary, not the platform's, and getting them wrong is expensive to correct
- **Any `custom` chunking function**, against the platform's registered interface
- **Deployment of the pipeline into your own boundary**, using the platform's infrastructure modules
- **The evaluation set**, which at this tier carries more weight because you have more ways to be wrong
- **Tuning**, iteratively, against that evaluation set

What you never build: extraction, embedding orchestration, index management, telemetry emission, provenance capture, cost labelling. Those are the platform's, and if you find yourself writing them, raise it as a gap rather than solving it locally.

### What the pipeline emits, unconditionally

Not configurable. This is the price of using the platform pipeline, and what makes the estate observable as a whole.

**Provenance on every chunk** — source id, document id, location within document, extraction method and version, embedding model and version, ingestion timestamp, content hash.

**An ingestion report per run** — documents seen, processed, skipped with reasons, quarantined, chunks produced, size distribution, gate rejections, duration, cost.

**Trace context** propagated through every stage, so ingestion appears in the same tracing view as retrieval ([Chapter 4](../chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md)).

**Cost attribution labels** — team, system, environment, cost centre — on every resource and every model call ([Chapter 5](../chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md)).

---

## 4. Decisions you cannot undo

More configuration means more one-way doors, not fewer.

**Requires a full reindex to change:** the embedding model or its version; the number of dimensions; the chunking strategy and its sizes; anything in `preserve`.

**Requires a schema migration and a reindex:** adding a required metadata field to an existing index, or changing a field's type.

**Effectively permanent:** the metadata vocabulary, once dashboards, filters and downstream consumers depend on the field names. Renaming a field is cheap in the index and expensive everywhere else.

**Constrained by an upstream choice you may not control:** chunk sizes are bounded by the embedding model's maximum input length. If that model is later replaced with one that accepts less, chunking has to change, and that is a reindex.

> **The versioning consequence.** Because these are reindex-class decisions, the platform's major-version policy is not an administrative detail — it is the thing that determines whether teams can stay current. See section 8.

---

## 5. What typically goes wrong, and the early symptom

| Symptom | What it means | Response |
| --- | --- | --- |
| Teams forking the pipeline | A missing parameter or extension point | Find the gap, add it, fold the fork back in |
| Everyone using `custom` chunking | Built-in strategies are inadequate | Promote the common custom implementations into the product |
| Teams stuck on old major versions | The upgrade path costs more than staying put | Improve migration tooling before shipping the next major |
| The Tier 3 population growing | Tier 2 is not credible | Interview the leavers; this is a product problem |
| Default configuration never used unmodified | The defaults are wrong | Re-derive them from what teams actually converge on |
| Retrieval worse after a tuning change | No evaluation set, or one that does not cover the change | Evaluation is not optional at this tier |
| Quality drifting with no change deployed | A floating model version, somewhere | Audit for unpinned versions across every team |

Tier 2 is a product with users. It succeeds or fails on the same terms as any other product: whether the people it is for choose it when they have alternatives.

---

## 6. What the platform provides, what the team owns

| Platform provides | Team owns |
| --- | --- |
| The pipeline, versioned, parameterised, supported | Configuration values and domain tuning |
| Infrastructure modules for deployment | Deploying into their own boundary |
| Approved embedding models, pinned, with a re-embedding runbook | Choosing among them; scheduling migrations |
| The registered extension interface for custom logic | Any custom function they register |
| Quality gates, provenance, ingestion reporting, telemetry | Acting on what the report says |
| Migration tooling and major-version support | Scheduling the upgrade |
| The governed interface contract | Conforming to it |

---

## 7. Reference implementation

> **Reference implementation.** A managed RAG framework sits at this tier — Google Cloud's RAG Engine, for instance, which exposes ingestion, transformation, embedding, indexing and retrieval as named, configurable stages rather than code you write. Comparable positions exist in the major agent frameworks and in self-assembled stacks built on a managed vector database.
>
> Worth establishing before choosing any of them: encryption with customer-managed keys and network perimeter controls are commonly supported, while **data residency commitments frequently are not**, and regional availability is uneven. That combination — strong on isolation, weak on residency — is exactly the kind of detail that decides a tier, and exactly the kind that is easy to misread because both subjects appear on the same documentation page.
>
> Treat every specific here as an example to verify rather than a specification to build against.

---

## 8. What you are signing up to operate

Real, and shared.

**Ongoing, team side:** the configuration repository; the evaluation set and the cadence of running it; the ingestion report; tuning; the pipeline's deployment in your own boundary and its failures.

**Ongoing, platform side:** the pipeline implementation, its dependencies and its security patching; the extension interface; migration tooling.

**The versioning contract**, because it is what the operating burden actually turns on:

- **Patch** — fixes with no effect on index output. Adopt freely.
- **Minor** — new capability, existing configuration unchanged in behaviour. Adopt at will.
- **Major** — changes that alter index output and therefore require reindexing.

**Support window:** the current major version and the one before it. The deprecation notice period must exceed the slowest consuming team's release cycle, or it is not a notice period.

**Upgrade support:** a migration guide, automated configuration conversion where feasible, and platform assistance for major-version reindexing. A major upgrade that leaves teams to work it out alone converts them into Tier 3 users.

**Deprecation notice appears in the ingestion report**, not only in a changelog nobody reads.

**Staffing reality:** roughly one engineer part-time per team for the pipeline itself, plus whatever the corpus deserves. The platform carries the rest — which is the entire argument for the tier existing.

---

[← Tier 1 — Managed](pipeline-tier-1-managed.md) | [Tier 3 — Bespoke →](pipeline-tier-3-bespoke.md)
