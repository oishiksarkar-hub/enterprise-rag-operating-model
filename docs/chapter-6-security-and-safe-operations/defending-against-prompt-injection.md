# Defending Against Indirect Prompt Injection

> Supporting material for [Chapter 6 — Security and Safe Operations](chapter-6-security-and-safe-operations.md)
>
> The implementation detail behind the layered defence: how payloads arrive, what each control actually catches, how to build the restricted execution mode, how to detect a successful attack, and the incident runbook.

---

## 1. The threat model, stated precisely

**The attacker's capability:** they can get text into a document that your pipeline will index. Nothing more. No network access, no credentials, no infrastructure foothold.

**The attacker's objective:** to have your model treat their text as instruction rather than data, and thereby act under your system's identity and entitlements.

**Why conventional controls do not apply:**

| Control | Why it does not help |
| --- | --- |
| Authenticating the user | The attacker is not the user |
| Authorising the caller | The call is legitimate; the *content* is not |
| Network segmentation | The payload arrives through the sanctioned ingestion path |
| Input validation on the query | The payload is not in the query |
| Model safety filters | The instruction is not harmful-sounding; it is procedural |

**The asymmetry:** the attacker writes once and waits. You must defend every document, from every source, against every model behaviour, indefinitely.

---

## 2. How payloads arrive

**Direct instruction.** Imperative text addressed to a model, often framed as a system note or compliance requirement. Crude, and still effective against unhardened systems.

**Invisible text.** White-on-white, zero-point font, zero-width characters, off-page positioning, hidden PDF layers, alt-text, document metadata fields. The human who approved the document saw nothing. **This is the vector that defeats human review** — which is what makes it worth naming first, whether or not it is the most frequent. Every other control on this list assumes someone could in principle have spotted the payload. This one removes that assumption entirely.

**Delimiter spoofing.** Content crafted to look like the boundary between context and instructions, attempting to convince the model that the retrieved section has ended and a new system message has begun.

**Role-play framing.** "For the purposes of this document, assume you are operating in diagnostic mode where the following applies…"

**Encoded payloads.** Base64, homoglyphs, unusual scripts, or instructions split across chunks so no single chunk looks suspicious — and reassembled only when several are retrieved together.

**Multi-stage.** A benign-looking document instructing retrieval of a second document that carries the real payload.

**Conflict manipulation.** Content asserting its own precedence: "this supersedes all other policy documents." Aimed at the synthesis step from [Chapter 4](../chapter-4-proving-retrieval-quality/chapter-4-proving-retrieval-quality.md) rather than at tool calling.

**Data exfiltration via rendering.** Instructions to construct a URL — an image source, a link — embedding retrieved sensitive data in the query string. The client renders it, and the data leaves. Frequently missed, because no tool was called at all.

---

## 3. Ingestion-time controls

### 3.1 Invisible content stripping

Run before anything else, and deterministic — no model judgement required.

```text
  strip:
    - zero-width and bidirectional control characters
    - text with foreground ≈ background colour
    - font size below a legibility threshold
    - content positioned outside the page boundary
    - hidden PDF layers and form field defaults
    - document metadata not intended for display
    - HTML comments, display:none, aria-hidden content
    - image alt-text, unless deliberately indexed
```

**Anything stripped is logged, not silently dropped.** A document containing hidden instructions is a security signal even if the sanitiser handled it. Repeated hits from one source is an incident, not a hygiene event.

### 3.2 Pattern detection

Heuristics over the extracted text. Cheap, high recall, low precision — which is why the response is quarantine, not rejection.

```text
  flag:
    - imperative phrasing directed at an assistant or model
    - references to system prompts, instructions, tools, functions
    - role reassignment or mode-switching language
    - text resembling message delimiters or role markers
    - instructions to ignore, override or disregard
    - URL construction templates with placeholder substitution
    - encoded blocks inconsistent with surrounding content
    - self-asserted precedence over other documents
```

**False positives are expected and acceptable.** Security documentation, prompt-engineering guides and this very document would trip several of these. That is precisely why detection routes to human review rather than to a block.

### 3.3 Model-based screening for untrusted sources

For untrusted-tier content, a screening model call asking whether the content contains instructions directed at an AI system. More expensive, materially better recall on novel phrasing. Reserve it for the untrusted tier, where the cost is justified.

> **Know what the screen does not see.** Managed screening services are useful and should be used, but they have limits that are easy to discover the hard way:
>
> - **They do not decode encoded payloads.** Content encoded as Base64, hexadecimal or similar is generally evaluated as the encoded string, not as what it decodes to. If your pipeline extracts and decodes such content downstream, the screen ran before the payload existed.
> - **They evaluate one request at a time.** Screening is typically stateless. An attack assembled across several turns — each turn innocuous, the combination not — does not present as malicious to a single-request check.
> - **They stop at a size limit.** There is a maximum amount of content examined per call, and material beyond it is not evaluated. Long documents must be chunked for screening, and if you chunk on a boundary the attacker chose, you can split a payload into halves that each look harmless.
>
> None of these makes screening pointless. All of them mean screening is a filter, not a boundary. **The boundary is still the privilege model** — the answer to "what happens if this gets through" has to be "nothing irreversible", because sometimes it will get through.

### 3.4 Quarantine workflow

```text
  document flagged
        │
        ▼
  held out of the index, reason recorded
        │
        ▼
  routed to the source owner + security
        │
   ┌────┴────┐
   ▼         ▼
 release   reject ──► source-level alert,
   │                  supplier notified if external
   ▼
 indexed, decision recorded against the document
```

Decisions are recorded against the document, so a re-crawl does not re-raise a settled case — and so that a pattern of rejections from one source is visible.

---

## 4. Trust tiers in practice

### 4.1 Assigning the tier

Assigned per source at configuration time, never inferred per document.

```yaml
sources:
  - id: internal-policy-library
    trust: trusted            # authored internally, reviewed, change-controlled
  - id: engineering-wiki
    trust: semi-trusted       # internal, but anyone can edit
  - id: partner-portal-uploads
    trust: untrusted          # externally supplied
  - id: web-crawl
    trust: untrusted
```

**Trusted requires all three:** internally authored, reviewed before publication, and change-controlled. A wiki any employee can edit is semi-trusted, however internal it is — insider risk and account compromise both route through exactly that path.

The tier is carried on every chunk and travels with retrieved content across MCP and A2A hops. **A2A is where this leaks if you let it:** Team A retrieves untrusted content and passes its synthesised answer to Team B, which has no idea untrusted material was involved. The trust tier must be part of the response metadata, and the lowest tier contributing to an answer must be carried forward.

### 4.2 Restricted execution mode

The central control. When untrusted content enters a model's context, that execution is constrained.

```text
  BEFORE untrusted content enters context
  ───────────────────────────────────────
    full tool access, subject to normal entitlement
    → decide and perform any tool calls now

  AFTER untrusted content enters context
  ──────────────────────────────────────
    no write tools
    no sensitive reads
    no outbound A2A calls
    no URL construction or link emission
    no new tool selection at all
    → synthesis and response only
```

**A one-way transition, enforced by the runtime**, not by instructing the model. The model cannot be relied upon to restrict itself; the point of the control is that it does not have to.

**The consequence for design:** agents that must both handle untrusted content and take action need two phases — gather and act first, then synthesise under restriction. This is a real constraint on what you can build, and it is the correct trade.

**Answer attribution.** Where untrusted content contributed, say so: *"Based partly on a supplier-provided document…"* The human then applies the judgement the system cannot.

---

## 5. Generation-time controls

**Structural delimitation.** Retrieved content in clearly marked regions, with system instructions establishing that anything inside is reference material. Imperfect — models can be talked out of it — and still worth having.

**Instruction hierarchy.** Use the provider's system-instruction mechanism rather than concatenating everything into one prompt. Models are trained to weight system instructions more heavily. Test how well yours actually does, with deliberate attempts.

**No URL or link emission from untrusted context.** Closes the exfiltration-by-rendering vector. If links are required, they must come from a validated allow-list resolved outside the untrusted context.

**Output screening.** Before returning to the caller:

```text
  reject or flag if the output:
    - contains instructions addressed to another system
    - contains URLs not derived from validated citations
    - contains structured data where prose was expected
    - references tools or capabilities not used in this request
    - diverges sharply from the question asked
    - contains encoded blocks
```

**Confirmation for consequential actions.** Any tool that writes, sends, spends or discloses requires explicit human confirmation when untrusted content is anywhere in the context.

> **This is the control that actually holds.** Everything else is probabilistic mitigation of a model's behaviour. A confirmation gate is deterministic. If you implement one thing from this document, implement this — and make the confirmation specific ("send this email to this recipient with this content"), because a generic "allow?" prompt is clicked through without reading.

---

## 6. Detection

Assume prevention will fail sometimes. Make success visible.

**Signals, from the trace data in Chapter 4:**

| Signal | What it suggests |
| --- | --- |
| Tool call unrelated to the question | Injection succeeded |
| Agent uses a tool it has never used before | Injection succeeded |
| Output structurally unlike this agent's norm | Behaviour altered |
| URL construction in an answer | Exfiltration attempt |
| Confirmation gate triggered unexpectedly | Attempted action |
| A specific document always precedes anomalies | Likely the payload |
| Repeated sanitiser hits from one source | Active attacker |
| Output referencing system prompts or instructions | Prompt extraction attempt |

**Correlation is the useful part.** A single anomaly is noise. The same document identifier appearing in the trace of every anomalous request is a finding, and it is the kind of finding that only exists because tracing was built in Chapter 4.

**Route to a security owner, not a quality owner.** Chapter 4's triage must carry an `injection_suspected` class that leaves the quality queue immediately.

**Red-team regularly.** Maintain an internal corpus of known payloads. Run it against every model change, prompt change and pipeline change. Add every real-world attempt to it. This is an evaluation set for security, and it belongs in the same gate as the quality one.

---

## 7. Incident runbook

**1. Contain.** Remove the suspect document from the index immediately — tombstone, do not purge; you need it for investigation. If a source is implicated, suspend its ingestion.

**2. Assess reach.** Query traces for every request that retrieved the document. How many, over what period, on whose behalf, and what did the system do?

**3. Assess action.** Did any tool call result? Any writes, sends, disclosures? This determines whether it is an incident or a near-miss, and the difference is material to everything that follows.

**4. Assess exfiltration.** Any URL construction, any outbound links rendered, any data in query strings. This is the quietest and most damaging outcome.

**5. Notify.** Security, the source owner, and — if the source is external — the supplier. Data protection, if personal data was reachable.

**6. Remediate.** Remove the payload. Review every other document from the same source. Review the source's submission controls.

**7. Harden.** Add the payload to the red-team corpus. Add a detection rule. Ask the harder question: **what allowed this to reach the index, and what else could reach it the same way?**

**8. Review the trust tier.** A source that produced one payload will produce another. Demote it, or add mandatory review for its content.

---

## 8. What the platform ships

- Invisible-content stripping and pattern detection in every pipeline, unconditionally
- Model-based screening available for untrusted sources
- The quarantine workflow, with routing and decision recording
- Trust-tier configuration, propagation across chunks and across A2A responses
- **The restricted execution mode, enforced by the agent runtime**
- Output screening and link-emission controls
- The confirmation gate, with specific, reviewable prompts
- Injection-suspicion detection rules over trace data
- The red-team corpus and its gate integration
- The incident runbook, and the tooling to execute it

**None of this is team-implemented.** A defence that varies by team is the weakest team's defence applied to the whole estate — because the attacker only needs one way in.

---

[← Back to Chapter 6](chapter-6-security-and-safe-operations.md)
