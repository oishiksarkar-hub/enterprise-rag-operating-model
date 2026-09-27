# Securing MCP and A2A Interfaces

> Supporting material for [Chapter 3 — Exposing a RAG Securely](chapter-3-exposing-a-rag-securely.md)
>
> The implementation detail behind the boundary: token validation, the tool entitlement model, delegation chains for agent-to-agent calls, and what the platform ships so that no team writes any of this themselves.

---

## 1. Token validation

Every request carries a bearer token issued by the organisation's identity provider. The receiving service validates it before any application logic runs.

**The checks, all of them mandatory:**

| Check | Why it exists | What it prevents |
| --- | --- | --- |
| Signature | Token is genuinely from the issuer | Forgery |
| Issuer (`iss`) | From *your* provider, not any provider | Tokens minted by an attacker-controlled IdP |
| Audience (`aud`) | Minted for *this* service | Replay of a legitimate token against a different service |
| Expiry (`exp`) | Still valid | Indefinite reuse of a captured token |
| Not-before (`nbf`) | Not pre-dated | Clock-skew abuse |
| Revocation | Not withdrawn since issue | Continued access after an incident |

The audience check is the one most often omitted, and its absence is the most serious. Without it, any service that can obtain a token from a caller can present that token to any other service in the estate. A low-sensitivity tool becomes a credential-harvesting point for a high-sensitivity one.

**Signing keys** are fetched from the issuer's published key set and cached with a bounded lifetime. Cache expiry must be short enough to honour key rotation and long enough to survive a brief issuer outage.

**Clock skew** tolerance of a small number of seconds is reasonable. Anything larger is a design smell.

---

## 2. The identity cases, and what each one requires

### 2.1 A human, interacting live

```text
  User ──[OIDC token]──► Agent ──► MCP tool
```

The token's subject is the person. Their group memberships, roles and attributes drive both tool entitlement and data scope. This is the well-trodden case.

### 2.2 A workload acting for itself

```text
  Scheduled job ──[workload identity token]──► MCP tool
```

There is no user, and inventing one is a mistake. The job has its own identity with its own entitlements — almost always narrower than any human's, because a job does exactly one thing.

**Each workload gets its own identity.** A shared service account across five jobs destroys attribution: you cannot tell which job caused a spike, which one to revoke after an incident, or which cost centre to bill in [Chapter 5](../chapter-5-cost-attribution-and-chargeback/chapter-5-cost-attribution-and-chargeback.md).

**No static keys.** Platform-issued, short-lived, automatically rotated credentials, bound to the workload's runtime identity rather than to a file.

### 2.3 A workload acting on behalf of a human

```text
  User ──► Agent ──[agent identity + user context]──► MCP tool
```

Both identities are present, and they answer different questions:

- **The agent's identity** answers *may this agent call this tool at all?*
- **The user's identity** answers *which documents may this person see?*

Collapsing them is the defect to watch for. An agent with a broad service identity, retrieving on behalf of a user with narrow entitlements, will happily return documents that user has no right to — and the audit log will record only the agent, so nobody will notice.

Implementation options, in order of preference:

1. **Token exchange.** The agent exchanges the user's token for one scoped to the downstream service, preserving the user as subject and recording the agent as the actor. Standards-based, verifiable, and the correct answer where the IdP supports it.
2. **Dual tokens.** The agent's own token for the connection, plus the user's original token forwarded in a dedicated header for data scoping. Simpler, and acceptable if the downstream validates both independently.
3. **Signed assertion.** The agent asserts the user's identity in a structure signed by the agent's key, with the agent held accountable for the assertion. A fallback, and usable only between services with an established trust relationship.

**Never acceptable:** the user identity as an unsigned field in the request body.

### 2.4 An agent calling an agent, for a human

```text
  User ──► Agent A ──► Agent B ──► Agent C ──► MCP tool
```

The delegation chain. Team C must be able to establish:

- Who the ultimate principal is (the user)
- That every intermediary is a legitimate, known service
- That each step was authorised, not merely performed

**Chain requirements:**

- **The full chain travels with the request.** Each hop appends itself; no hop may rewrite or remove earlier entries.
- **Each hop is independently verifiable** — signed by the identity of the service that added it.
- **Depth is bounded.** A maximum chain depth, enforced, rejecting anything deeper. This limits blast radius and catches cycles.
- **Loop detection.** A chain containing the same service twice is rejected. In an agent mesh this is not hypothetical; two agents that each consider the other authoritative will loop until something stops them.
- **Entitlement is the intersection, never the union.** The effective permission is what the user is entitled to *and* what every intermediary is entitled to. A user with broad access reached through a restricted agent gets the restricted view. This is the property that keeps an agent from becoming a privilege-escalation route.

---

## 3. The tool entitlement model

### 3.1 Policy shape

Entitlement is granted per principal, per tool — never per server.

```yaml
server: customer-master-mcp
tools:
  - name: search_customer_documents
    sensitivity: internal
    allow:
      - group: customer-service-agents
      - group: sales-all
      - workload: order-enrichment-job
    data_scope: user-entitlements      # filter derived from caller

  - name: get_customer_profile
    sensitivity: confidential
    allow:
      - group: customer-service-tier2
    data_scope: user-entitlements
    require: [ user_context ]          # no pure-workload calls

  - name: get_customer_pii
    sensitivity: restricted
    allow:
      - group: compliance-investigators
    data_scope: user-entitlements
    require: [ user_context, purpose_declaration ]
    audit: enhanced

  - name: update_customer_record
    sensitivity: restricted
    write: true
    allow:
      - workload: customer-master-admin-service
    require: [ user_context, change_reference ]
    audit: enhanced
```

Points worth drawing out:

**`require: [user_context]`** blocks a workload from calling a tool on nobody's behalf. Sensitive reads should always have a human principal answerable for them.

**`purpose_declaration`** asks the caller to state why. It does not prevent misuse, but it makes misuse visible in the audit log and creates an accountable record — which in regulated environments is the actual requirement.

**Write tools are separated and narrowly granted.** A read-only server is a very different risk object from one that can mutate the record. Where possible, keep them on separate servers entirely, so an over-broad grant on the read server cannot reach a write path.

### 3.2 Filtered discovery

When a caller lists tools, it receives only the tools it is entitled to call.

```text
  compliance-investigator lists tools:
    search_customer_documents
    get_customer_profile
    get_customer_pii

  sales agent lists tools:
    search_customer_documents
```

This is not cosmetic. An unauthorised tool that is visible is an information disclosure — the name and argument schema of `get_customer_pii` tells an attacker what exists and what it takes. It is also a reliability improvement: a model that can see a tool will eventually try it, and a stream of authorisation failures is noise that hides real signal.

### 3.3 Enforcement position

```text
  request
    │
    ├─ 1. validate token                    ─┐
    ├─ 2. resolve principal + chain          │  platform
    ├─ 3. authorise tool                     │  middleware
    ├─ 4. derive data-scope filter           │  (no team code)
    ├─ 5. start trace span, record usage    ─┘
    │
    ├─ 6. TOOL HANDLER                       ← team code begins here
    │      retrieval executed WITH the filter
    │
    ├─ 7. post-retrieval verification       ─┐  platform
    └─ 8. close span, emit usage + cost     ─┘  middleware
```

Steps 1–5 and 7–8 are platform middleware. The team writes step 6 and nothing else. **If a team can write a handler that forgets to authorise, the architecture is wrong** — the framework must make the insecure version unavailable rather than merely discouraged.

---

## 4. Data-scope filtering

Tool authorisation is binary. Data scope is not.

**The translation:**

```text
  principal attributes          index filter
  ────────────────────          ────────────
  groups: [emea-sales]     ──►  region IN ('EMEA')
  clearance: internal      ──►  classification IN ('public','internal')
  managed_accounts: [...]  ──►  account_id IN (...)
                                └─ combined with AND, applied
                                   as part of the query itself
```

**Design rules:**

- **Default deny.** Absent entitlement metadata on a document means it is not returned. A chunk with no `classification` is invisible, not public. This is why Chapter 2's pipeline makes classification a required field.
- **The filter is part of the query, never a wrapper around results.** See Chapter 3 §3.5 for why.
- **Dynamic entitlements are resolved per request**, not cached across requests. **The principle: the staleness of an entitlement decision is the window in which a revoked user retains access, so the cache lifetime is a security parameter and has to be chosen as one.** Cache the *lookup* if you must, with a lifetime short enough that you would be comfortable stating it in an incident report, and shorter still for anything that grants access to restricted material.
- **Post-retrieval verification remains** as a second line — every returned chunk re-checked against the principal. It should never fire. If it does, you have a filter bug, and it should page someone.

---

## 5. Rate limiting and abuse

Covered more fully in [Chapter 6](../chapter-6-security-and-safe-operations/chapter-6-security-and-safe-operations.md), but the interface is where it is enforced:

- **Per-principal quotas**, not per-source-IP — in an agent mesh, IP is meaningless.
- **Separate limits for expensive tools.** A generative A2A call and a keyword lookup should not share a budget.
- **Chain-aware limiting.** A user request that fans out to eight downstream agents is one user action; limits keyed only at the leaf will either throttle legitimate fan-out or miss genuine abuse.
- **Fail closed on limiter unavailability** for restricted tools, open for public ones. State the choice deliberately; do not inherit it from a library default.

---

## 6. What the platform ships

**Server boilerplate**, in the organisation's supported languages, with middleware pre-wired. A team clones it, declares tools in a manifest, writes handlers.

**Policy as configuration**, in the team's repository, reviewed as code, applied without redeployment.

**A conformance suite** any team can run against its own interface:

- Requests with no token are rejected
- Tokens with a wrong audience are rejected
- Expired tokens are rejected
- Unauthorised tools are absent from discovery and rejected on call
- Data-scope filtering is verified with a deliberately restricted test principal
- Delegation depth and loop limits are enforced
- Telemetry and usage records are emitted with correct attribution

**Passing the conformance suite is the condition of registration in the estate registry.** That is the enforcement point — not a policy document, but the fact that an unregistered interface is undiscoverable and therefore unused.

---

[← Back to Chapter 3](chapter-3-exposing-a-rag-securely.md)
