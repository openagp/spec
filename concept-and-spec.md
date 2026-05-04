# AGP — Agent Governance Protocol

**Status:** v0.1 working draft · May 2026
**Authors:** ZAK / Zeron founding team
**Audience:** Engineering (Claude Code first reader), founding team, future standards-body collaborators
**License intent:** Apache-2.0 once published; the spec itself will be CC-BY

---

## TL;DR

**AGP is the cross-vendor protocol for agent governance.**

The way SAML standardized "who is this user" across identity vendors, and MCP standardized "what tools can this agent call" across model vendors — AGP standardizes "what is this agent allowed to do, who said so, and what did it actually do" across every AI agent vendor a customer uses.

ZAK ships the canonical AGP implementation. The spec is open. Vendors who ship AGP receivers become trivial to govern; customers who write AGP policy do it once and have it enforced everywhere. The standard is the moat.

This document defines:
1. **Why** AGP needs to exist (strategy context — short)
2. **Core concepts** — actors, terms, the three protocol flows
3. **v0 protocol specification** — message schemas, conformance levels, security model
4. **Reference implementation plan** — what Claude Code should build first, in what order
5. **Adoption strategy** — how a single startup drives industry adoption of an open spec
6. **Risks and open questions**

This is a working doc. Things marked **[OPEN]** need a decision; **[DEFER]** is intentionally deferred to a later version.

---

## 1. Why AGP

### 1.1 The fragmentation problem

Within 18–24 months, a typical F500 will run 50–200 AI agents across 10–30 vendors. Today, each vendor exposes:

- Its own audit format (Anthropic Trust Center JSON differs from OpenAI's audit log differs from Microsoft Purview's signals differs from a homegrown agent's plain-text logs).
- Its own policy surface, if any (Anthropic has rudimentary admin controls; Vanta has compliance checklists; most agents have nothing).
- Its own identity model (OAuth scopes per vendor, no concept of "this agent acts on behalf of this human, with this lease, until this revocation").

The customer needs a *unified* view. They cannot get one because no two vendors agree on what an "agent action" looks like, let alone what a policy is.

This is the same problem identity had pre-SAML, pre-OAuth: every SaaS vendor invented its own login. Okta's wedge was *not* "we are the best identity provider" — it was "we are the protocol-aggregator that turns a fragmented vendor mess into a single customer view." When SAML standardized the protocol, Okta's role evolved into the canonical implementation of the standard plus the customer-facing control plane.

AGP is that protocol for agent governance. ZAK is that control plane.

### 1.2 Why now

Three structural conditions make this the moment:

1. **Regulatory pressure is real and uniform.** EU AI Act, NIST AI RMF, ISO 42001, HIPAA AI guidance — every framework demands evidence that customers cannot produce without cross-vendor unification. The buyer pull will exist.
2. **No incumbent owns this.** Hyperscalers (Microsoft, Google, AWS) cannot credibly ship a vendor-neutral protocol — their entire commercial motion is to lock customers into their cloud. Foundation model vendors (Anthropic, OpenAI) cannot credibly govern *each other*. Compliance incumbents (Vanta, Drata) own the checklist but not the runtime. The vendor-neutral seat is structurally vacant.
3. **The first mover sets the spec.** Anthropic's MCP captured "tool calling" by being the first credible attempt to standardize — even though MCP is technically arguable, the network effects locked in. The same window is open for governance. Whoever ships a credible AGP v0 in 2026 sets the conversation for the next decade.

### 1.3 Why ZAK is positioned to drive this

- **Existing leverage.** ZAK runtime is in production. The Action Ledger, SCF mapping engine, and seven-layer enforcement model give us a working reference implementation before the spec is even published.
- **Vendor-neutrality is structural.** ZAK doesn't sell a model, doesn't sell a cloud, doesn't compete with Anthropic or OpenAI. We are the only credible neutral party.
- **Customer base is the right one.** US mid-market regulated SaaS (Tier 1 ICP) is exactly the buyer who feels the fragmentation pain first. They become the demand pull on vendors to ship AGP receivers.

The wedge is Action Ledger (already in MVP). The category-defining play is AGP. They are the same product, two horizons.

---

## 2. Core concepts

### 2.1 Actors

| Actor | Role |
|---|---|
| **Customer** | The organization running agents. Owns policy. Receives events. |
| **Vendor** | Provides agent capability (Anthropic, OpenAI, an internal agent platform, a SaaS that ships agents). Receives policy. Emits events. |
| **Plane** | The customer's governance control plane. ZAK is one such plane; in principle any AGP-compliant implementation can play this role. |
| **Registry** | Decentralized directory of AGP-compliant vendors and their public keys. |
| **Auditor** | Read-only consumer of events; may be internal compliance, external auditor, or regulator. |

### 2.2 Terms

- **Agent** — an autonomous or semi-autonomous AI process that takes actions on behalf of a customer. Identified by `agent_id` within a vendor's namespace.
- **Action** — a single discrete thing an agent does (a tool call, a model invocation, a write to a system, a decision). Identified by `event_id`.
- **Policy** — a customer-authored set of rules describing what agents may, must, or must not do.
- **Decision** — a vendor-side determination of `allowed | blocked | logged_only` for a specific action, in light of policy.
- **Event** — a canonical record of an action plus its decision, emitted by the vendor.
- **Ledger** — the customer-side append-only, hash-chained store of events (see ZAK Action Ledger PRD 01).
- **Conformance level** — the depth of AGP support a vendor implements (L1, L2, L3).

### 2.3 The three protocol flows

**Flow A — Event emission (L1):**
Vendor → Plane. The vendor emits AGP-canonical events for every agent action. This is the minimum viable level of AGP support.

**Flow B — Policy delivery (L2):**
Plane → Vendor. The customer (via plane) pushes AGP-formatted policy to the vendor. The vendor evaluates incoming actions against policy and reflects the decision in the events it emits.

**Flow C — Real-time decision (L3):**
Vendor → Plane (request) → Vendor (response), synchronous. For high-stakes actions the vendor cannot self-decide, the vendor calls back to the plane with the proposed action and the plane returns `allowed | blocked` within a budget (default 300ms). The vendor must apply that decision.

A vendor can implement L1 alone (passive observability), L1+L2 (governance), or all three (full AGP). Customers can prefer L3 vendors for sensitive workloads.

---

## 3. v0 protocol specification

This section is a sketch-level spec. It is detailed enough for an engineer to start a reference implementation. It is not a final RFC. Items marked **[OPEN]** are decisions for the standards working group once formed.

### 3.1 Versioning and naming

- Protocol identifier: `agp`
- Version: `0.1` (this document)
- All AGP messages carry an `agp_version` field. Receivers must reject unknown major versions and warn on unknown minor versions.

### 3.2 Transport

- **Default:** HTTPS with mTLS *or* bearer-token authentication (negotiated at vendor registration).
- **Encoding:** JSON for v0 (schema below). gRPC + protobuf is **[DEFER]**ed to v0.3.
- **Endpoints:** vendors expose two HTTPS endpoints to the plane:
  - `POST /agp/v0/events` — receives policy from the plane (Flow B)
  - `GET /agp/v0/policy` — plane reads vendor's currently active policy snapshot (for verification)
- Planes expose three endpoints to vendors:
  - `POST /agp/v0/events` — receives events from vendors (Flow A)
  - `POST /agp/v0/decision` — receives real-time decision requests (Flow C)
  - `GET /agp/v0/.well-known/agp` — capability discovery

### 3.3 Discovery (`.well-known/agp`)

Every plane and vendor exposes a discovery endpoint listing supported conformance levels, public keys, and capabilities.

```json
{
  "agp_version": "0.1",
  "actor_kind": "vendor",
  "actor_id": "anthropic.com",
  "conformance_levels": ["L1", "L2"],
  "public_keys": [
    {
      "key_id": "anthropic-2026-q2",
      "purpose": "event_signing",
      "alg": "Ed25519",
      "value": "base64..."
    }
  ],
  "endpoints": {
    "policy_receive": "https://api.anthropic.com/agp/v0/policy",
    "policy_read": "https://api.anthropic.com/agp/v0/policy"
  },
  "decision_budget_ms": 300,
  "supported_action_types": ["tool_call", "model_response", "session_open"],
  "registry_record": "agp-registry.org/anthropic.com"
}
```

### 3.4 Canonical event schema (Flow A — L1)

This is the single most important message in the protocol. Schema is intentionally aligned with ZAK's Action Ledger schema (PRD 01) so the canonical event *is* the Action Ledger event. Every AGP-compliant plane MUST be able to ingest this format directly.

```json
{
  "agp_version": "0.1",
  "schema_version": "1.0",
  "event_id": "evt_01JFXY8B...",            // ULID, vendor-generated, immutable
  "occurred_at": "2026-08-12T14:23:11.412Z",
  "actor": {
    "vendor": "anthropic.com",               // FQDN or registry-issued ID
    "agent_id": "agt_claude_3_5_sonnet",
    "human_principal": "user_hash_2d4f..."   // sha256 of upstream user id
  },
  "action": {
    "type": "tool_call",
    "tool_name": "browser.navigate",
    "target_resource": "https://example.com/customer-data",
    "input_summary": "...",                   // ≤ 2 KB redacted
    "output_summary": "...",
    "input_hash": "sha256:...",
    "output_hash": "sha256:..."
  },
  "policy": {
    "decision": "allowed",                    // allowed | blocked | logged_only
    "rule_id": "rule_pii_outbound_v3",
    "policy_hash": "sha256:...",              // hash of the policy doc applied
    "rationale": "no PII detected"
  },
  "lineage": {
    "trace_id": "trc_01JFXY8...",
    "parent_event_id": null
  },
  "signature": {
    "key_id": "anthropic-2026-q2",
    "alg": "Ed25519",
    "value": "base64..."
  }
}
```

**Required vs. optional:**
- Required at L1: `agp_version`, `schema_version`, `event_id`, `occurred_at`, `actor.vendor`, `actor.agent_id`, `action.type`, `signature`.
- Required at L2 (additionally): `policy.decision`, `policy.policy_hash`, `policy.rule_id`.
- Optional but recommended: `input_hash`, `output_hash`, `lineage`.

**SCF tagging:** the plane (not the vendor) tags events with regulatory framework controls. AGP does not require vendors to know about SCF.

### 3.5 Policy descriptor (Flow B — L2)

Customer-authored policy is expressed in AGP Policy DSL — YAML-shaped, declarative.

```yaml
agp_policy_version: "0.1"
policy_id: "acme-prod-policies"
policy_hash: "sha256:..."        # computed from canonicalized body
issuer: "acme.example.com"
issued_at: "2026-08-01T00:00:00Z"
applies_to:
  vendors: ["anthropic.com", "openai.com"]
  agents: ["*"]
  actions: ["tool_call", "model_response"]
rules:
  - id: rule_pii_outbound_v3
    when:
      action.type: tool_call
      action.target_resource:
        domain_not_in: ["acme.com", "*.acme.com"]
      action.input_summary:
        contains_pattern: "ssn|credit_card|email"
    then:
      decision: blocked
      reason: "PII outbound to external domain"
  - id: rule_log_all_database_writes
    when:
      action.tool_name:
        starts_with: "database.write"
    then:
      decision: logged_only
      annotate:
        scf_controls: ["DATA-08", "AUDIT-12"]
metadata:
  description: "Acme prod governance · v3"
  contact: "security@acme.com"
signature:
  key_id: "acme-2026-q2"
  alg: "Ed25519"
  value: "base64..."
```

**Conformance L2 vendor obligations:**
- Accept policy via `POST /agp/v0/policy` and respond `202 Accepted` with the `policy_hash` once active.
- Apply policy to all subsequent matching actions within an SLA (default: 60 seconds).
- Stamp `policy.policy_hash` on every event so the customer can verify which policy version was in force.
- Refuse policies signed by keys not registered for the customer.

**[OPEN]** — DSL grammar formalization. The shape above is illustrative; v0.2 will publish a JSON Schema for the DSL with a reference parser.

### 3.6 Real-time decision protocol (Flow C — L3)

For high-stakes actions, vendors call back to the plane synchronously.

**Request (vendor → plane):**
```json
{
  "agp_version": "0.1",
  "decision_request_id": "drq_01JFX...",
  "occurred_at": "2026-08-12T14:23:11Z",
  "actor": {
    "vendor": "anthropic.com",
    "agent_id": "agt_claude_3_5_sonnet",
    "human_principal": "user_hash_2d4f..."
  },
  "proposed_action": {
    "type": "tool_call",
    "tool_name": "email.send",
    "target_resource": "external@acme-customer.com",
    "input_summary": "Quarterly report attached",
    "input_hash": "sha256:..."
  },
  "context": {
    "trace_id": "trc_01JFX...",
    "policy_hash_in_use": "sha256:..."
  },
  "deadline_at": "2026-08-12T14:23:11.300Z",
  "signature": { "key_id": "...", "alg": "Ed25519", "value": "..." }
}
```

**Response (plane → vendor):**
```json
{
  "agp_version": "0.1",
  "decision_request_id": "drq_01JFX...",
  "decision": "allowed",
  "rule_id": "rule_external_email_review",
  "rationale": "external recipient on customer allowlist",
  "ttl_seconds": 0,
  "signature": { "key_id": "...", "alg": "Ed25519", "value": "..." }
}
```

**Vendor obligations at L3:**
- Honor the response decision exactly. `blocked` means the action does not execute.
- If the plane does not respond within `deadline_at`, fall back to the default behavior declared in policy (`policy.fallback`, e.g., `allow_with_log` or `block`).
- Emit an event for the action regardless (the event records both the decision and whether plane was reachable).

### 3.7 Identity, trust, and the Registry

AGP uses public-key cryptography end-to-end.

- Every actor (vendor, plane, customer) has at least one Ed25519 keypair.
- Public keys are advertised via `.well-known/agp` and registered in the AGP Registry.
- Events are signed by the vendor's `event_signing` key.
- Policies are signed by the customer's `policy_signing` key.
- Decision responses are signed by the plane.

**The AGP Registry** is a public, decentralized directory. v0 implementation: a public Git repository (`github.com/agp-registry/registry`) where each entry is a signed JSON file. Vendors and planes submit pull requests; a working group merges. Future versions move to an append-only ledger of its own (likely a transparency log).

**[OPEN]** — Registry governance. Working group composition. Conflict resolution. Decision: defer until first 5 vendors join; bootstrap via ZAK + Anthropic + OpenAI + 2 customer references.

### 3.8 Conformance and self-test

Every implementation MUST pass the AGP Conformance Test Suite (CTS) before claiming a conformance level.

CTS is a CLI tool (`agp-cts`) that:
- Runs a battery of test events against a vendor's `POST /agp/v0/events`-equivalent test endpoint.
- Validates schema conformance, signature correctness, policy-hash echoing, and decision-budget compliance.
- Outputs a signed conformance report.

Vendors who pass list themselves in the registry with their conformance level. Customers can require minimum levels in procurement.

**[OPEN]** — Whether the CTS itself is centrally administered (ZAK-run) or runs entirely on the vendor side (with reports verifiable by anyone). Default: client-side runner, anyone can verify reports — no central authority.

### 3.9 Backward compatibility, versioning, and deprecation

- **Additive-only at minor versions.** v0.1 → v0.2 may add fields; existing fields cannot change semantics.
- **Breaking changes only at major versions.** v0.x → v1.0 may break; minimum 12-month deprecation runway with both versions supported.
- **Graceful unknown-field handling.** Receivers ignore unknown fields they don't understand and log a warning; they do not reject messages on that basis alone.

### 3.10 Security model summary

| Threat | Mitigation |
|---|---|
| Vendor forges events | Events signed; plane verifies against vendor's registered key. Forgery requires key compromise. |
| Plane forges policy | Policies signed by customer; vendor verifies against customer-registered key. |
| Replay / reorder | `event_id` is unique; `occurred_at` is checkable; events join a customer-side hash chain (Action Ledger). |
| Vendor cannot reach plane (L3) | Policy declares `fallback`; vendor applies it. Event records that fallback was used. |
| Compromised registry entry | Registry entries are themselves signed; mirrors verify. |
| Schema drift | Versioned schema; receivers validate; CTS catches divergence. |
| Cross-vendor identity correlation | Each vendor has its own `actor.agent_id`; correlation happens plane-side via `human_principal` (which is a hash, not a plaintext identifier). |

---

## 4. Reference implementation plan

This section is the most important for Claude Code. It defines what to build, in what order, and how each piece relates to existing ZAK MVP work.

### 4.1 Repositories

We will create the following GitHub repositories:

| Repo | Purpose | Language | License |
|---|---|---|---|
| `openagp/spec` | The spec itself (this document, formalized) + JSON Schemas | Markdown / JSON | CC-BY 4.0 |
| `openagp/cts` | Conformance Test Suite CLI | Go | Apache-2.0 |
| `openagp/sdk-python` | Reference vendor + plane SDK (Python) | Python | Apache-2.0 |
| `openagp/sdk-typescript` | Reference vendor + plane SDK (TypeScript) | TypeScript | Apache-2.0 |
| `openagp/registry` | Public registry of compliant actors | YAML (GitOps) | CC0 |
| `openagp/examples` | Worked examples — mock vendor, mock plane, end-to-end demo | Python | Apache-2.0 |

`zak/` (the closed-source product repos) consume `openagp/sdk-python` as the reference plane implementation.

### 4.2 Build order — what Claude Code should build first

**Phase 0 — Spec formalization (Week 1, 1 engineer)**

Goal: turn this document into a machine-checkable spec with reference test fixtures.

1. Set up `openagp/spec` repo. Move sections 2–3 of this doc into the repo.
2. Author JSON Schemas for: canonical event, policy descriptor, decision request, decision response, discovery document.
3. Set up CI that validates every fixture against the schemas.
4. Publish 50 fixture event examples covering all `action.type` values.
5. Publish 10 fixture policy examples covering all rule patterns.
6. Document signature canonicalization (which fields are included; field ordering; whitespace handling). This MUST be precisely defined or signatures will not interoperate.

**Phase 1 — SDK and SDK conformance (Weeks 2–4, 1–2 engineers)**

Goal: a vendor or plane can build AGP support in a day.

1. `openagp/sdk-python`:
   - `openagp.events.Event` Pydantic model with schema validation.
   - `openagp.events.sign(event, key)` and `openagp.events.verify(event, public_key)`.
   - `openagp.policy.Policy` Pydantic model.
   - `openagp.policy.evaluate(action, policy) -> Decision` reference policy engine.
   - HTTP client + server scaffolds (FastAPI for server, httpx for client) implementing the four endpoints.
   - End-to-end example: a "mock vendor" service emits 100 events to a "mock plane" service, with signatures verified.
2. `openagp/sdk-typescript`:
   - Mirror of the Python SDK. Same shape, same tests, same fixtures.
3. CI: every commit runs cross-language interop tests (Python emits, TypeScript verifies; and vice versa).

**Phase 2 — CTS (Weeks 4–6, 1 engineer)**

Goal: any vendor can run `agp-cts validate-vendor --endpoint https://...` and get a pass/fail.

1. `openagp/cts` Go CLI:
   - `validate-vendor` command — runs schema, signature, replay, and policy-conformance tests against a vendor's test endpoint.
   - `validate-plane` command — runs the inverse.
   - `validate-fixture` command — runs locally against a JSON file.
2. Output: signed conformance report (JSON) listing which conformance level the implementation passed.
3. Distributable as a single static binary for Linux, macOS, Windows.

**Phase 3 — Plane integration in ZAK (Weeks 6–8, 1 engineer)**

Goal: ZAK speaks AGP natively. Existing Anthropic + OpenAI vendor connectors (PRD 03) become AGP-shaped under the hood.

1. ZAK's `POST /v1/events` endpoint also accepts AGP-formatted events (already substantially compatible — schemas are aligned).
2. ZAK exposes `.well-known/agp` for itself.
3. ZAK Action Ledger emits AGP-conformant events to any subscribed plane (chained planes scenario).
4. ZAK Console v1 (PRD 04) shows an "AGP-status" badge per vendor — green for L1+, yellow for L1 only, gray for non-compliant.
5. The ZAK `agp-cts` runner becomes part of the customer's vendor-onboarding playbook.

**Phase 4 — Registry bootstrap (Weeks 8–12, 0.5 engineer + founder time)**

Goal: 5 actors registered, including 2 customers.

1. `openagp/registry` repo with PR-based submission flow.
2. ZAK is the first registered plane.
3. Anthropic + OpenAI submitted as L1 (we author the entries; they confirm/co-sign or we ask them to take over).
4. 2 customer references registered.
5. Public website `openagp.io` (or equivalent) listing the registry.

### 4.3 What Claude Code should NOT build first

- **L3 reference implementation.** L3 (real-time decision) requires sub-300ms round-trip and adds significant complexity. Land L1 + L2 first.
- **Custom DSL parser beyond YAML.** v0 policy is YAML-shaped. A richer DSL (e.g., Rego-style) is a v0.3 conversation.
- **Federation between planes.** Plane-to-plane sync is post-v1.
- **GUI policy editors.** AGP is a protocol; UX is the plane's job.
- **Vendor-specific shims.** Anthropic and OpenAI specifically — those live in ZAK's connectors (PRD 03) until those vendors ship native AGP receivers.

### 4.4 Definition of done — v0.1 launch

The spec is ready for public release when:

- [ ] `openagp/spec` is published with schemas, fixtures, and a complete signature canonicalization spec.
- [ ] `openagp/sdk-python` and `openagp/sdk-typescript` pass cross-language interop CI.
- [ ] `openagp/cts` validates ZAK's plane and a mock-vendor implementation end-to-end.
- [ ] `openagp/registry` has 5 entries (ZAK + 2 vendors + 2 customers).
- [ ] An external engineer (not the spec authors) ships a vendor implementation in < 1 working week using the SDK.

Target: ~12 weeks of engineering. Aligns with ZAK's MVP shipping window — by the time v1 customers are paying, AGP v0.1 is public.

---

## 5. Adoption strategy

A single startup driving an open standard works only if the standard is genuinely better for everyone — and if early signals are unmistakable.

### 5.1 The four parties and what we offer each

**Customers** — get one policy language, one audit view, no per-vendor integration burden. Their compliance toil drops by 80%. Their procurement RFPs add "minimum AGP L1" as a default. *Pull on vendors.*

**Vendors** — get a published spec, free SDKs in two languages, a CTS that turns "AGP-compliant" into a checkbox, and a registry listing that prospects discover them through. The cost to ship L1 is < 1 engineer-week. *No reason to refuse.*

**Auditors and regulators** — get standardized evidence formats. EU AI Act conformity reports become mechanical. *Become advocates.*

**Foundation models** — get a path out of "we are responsible for governance" toward "the customer's plane governs us." Anthropic's recent governance posture (post Claude Security launch) suggests they want this shape. *Strategic alignment.*

### 5.2 The sequencing

| Quarter | Move |
|---|---|
| Q3 2026 | Spec v0.1 + SDKs published. ZAK is the first plane. |
| Q3 2026 | First customer deployments use AGP under the hood (transparent to them). |
| Q4 2026 | Co-author entries with Anthropic + OpenAI or, if they decline, ship adapters that make their products AGP-shaped from outside (we already have to do this for PRD 03). |
| Q1 2027 | Working group formed. Membership: ZAK + 2 vendors + 2 customers + 1 academic. |
| Q1 2027 | First customer RFP contains "vendor must be AGP L1 conformant." (We'll seed this with our reference customers.) |
| Q2 2027 | v0.2 of spec — incorporates working-group feedback, formalizes DSL grammar, opens governance. |
| Q3 2027 | First non-ZAK plane implementation appears (likely a hyperscaler responding to customer pull). |
| 2028 | AGP becomes the default expectation in regulated procurement. |

### 5.3 Why we publish the spec instead of keeping it proprietary

A common pushback: "if you publish the spec, anyone can build a competing plane."

That's the point. The moat is not the spec. The moat is:

1. **Reference implementation quality** — ZAK's plane is materially better than a from-scratch alternative.
2. **Customer Action Ledger lock-in** — by the time a competing plane arrives, customers have years of audit history in ZAK.
3. **SCF + ZAK runtime leverage** — competitors don't have these.
4. **Standards capture** — the working group's chair sets the future direction. We chair it.

A proprietary protocol fails the moment a vendor refuses to adopt it. An open protocol with us as the canonical implementation captures the customer demand AND positions us as the reference point.

This is the Okta playbook. It worked.

### 5.4 What "winning" looks like

- 50+ vendors registered as L1+ by end of 2027.
- AGP referenced in EU AI Act technical guidance (or US equivalent) as an example of conformity infrastructure.
- ZAK is the canonical plane, but is not the only plane — proving the protocol's neutrality.
- Customers say "is your X AGP-compliant?" the way they say "do you support SAML?"

---

## 6. Risks and open questions

### 6.1 Risks

- **R1 — Anthropic ships a competing protocol via MCP extension.** Likely the single biggest risk. *Mitigation:* engage Anthropic early; offer co-authorship; be willing to merge MCP-extension semantics into AGP if the technical merits warrant it. Better to be co-stewards of the standard than competitors.
- **R2 — Hyperscalers ship a captive variant.** Microsoft Purview-for-Agents could ship an "open-but-Azure-locked" governance protocol. *Mitigation:* speed. v0.1 in market within ~12 weeks. Customer pull crystallizes before hyperscalers can move.
- **R3 — No vendor adopts.** Without vendor adoption, AGP is shelfware. *Mitigation:* the SDK lowers cost to < 1 engineer-week. Customer RFP pull (sequencing §5.2) creates demand. Worst case, we adapter-shim non-compliant vendors and ship the events anyway — as PRD 03 already does.
- **R4 — The working group fights itself.** Standards committees historically slow to glacial. *Mitigation:* explicit governance model; ZAK chairs initially; biased toward shipping over consensus until v1.0.
- **R5 — Spec gets too complex.** Designing-by-committee bloats specs. *Mitigation:* every addition must come with two existing implementations; otherwise rejected.

### 6.2 Open questions

- **Q1 (eng / spec):** Signature canonicalization scheme — JCS (RFC 8785), JWS, or something custom? *Recommendation: JCS for canonicalization + Ed25519 detached signatures. Decision needed week 1 of Phase 0.*
- **Q2 (eng / spec):** Decision-request budget — 300ms default reasonable? Some workloads (HITL) need seconds; some (autocomplete) need < 50ms. *Recommendation: 300ms default, vendor can advertise different budget in `.well-known/agp`. Plane respects vendor's stated budget.*
- **Q3 (legal):** Spec license — Apache-2.0 vs. CC-BY vs. RFC-style? Patent grants? *Action: legal review. Recommendation: spec as CC-BY; SDKs as Apache-2.0 with explicit patent grant.*
- **Q4 (strategy):** Do we approach Anthropic and OpenAI before publishing v0.1, or publish first? *Recommendation: publish first. Easier for them to engage with a real artifact than a proposal.*
- **Q5 (strategy):** Do we name the standard "AGP" publicly or workshop the name? *Recommendation: workshop. The working name is fine internally; the public name benefits from broader input. Possible alternative names: AGCP (Agent Governance Control Protocol), OAGP (Open Agent Governance Protocol).*
- **Q6 (eng):** Registry — Git-based v0 is fine, but what's the v1 plan? *Defer. Decide once we have 20+ entries and Git friction becomes real.*
- **Q7 (eng):** Should events be batchable? `POST /agp/v0/events:batch` for high-throughput vendors? *Recommendation: yes, in v0.2. Single events in v0.1 to keep the spec tight.*
- **Q8 (strategy):** When do we propose AGP to a standards body (IETF, OASIS, W3C)? *Recommendation: not until v1.0 and 20+ adopters. Standards bodies are the destination, not the starting point.*

---

## 7. Glossary

- **Action Ledger** — ZAK's append-only, hash-chained, SCF-tagged event store. AGP events flow into it natively.
- **AGP** — Agent Governance Protocol. The standard defined in this document.
- **Conformance Level** — depth of AGP support: L1 (events only), L2 (events + policy), L3 (events + policy + real-time decision).
- **CTS** — Conformance Test Suite. CLI for self-testing and signed reports.
- **MCP** — Model Context Protocol. Anthropic's tool-calling standard. Adjacent but distinct from AGP.
- **Plane** — customer-side governance control plane. ZAK is one such plane.
- **Policy** — customer-authored rules in AGP DSL.
- **Registry** — public directory of AGP-compliant actors with their public keys.
- **SCF** — Secure Controls Framework. Crosswalk between regulatory frameworks; ZAK uses it for compliance evidence.
- **Vendor** — provider of agent capability. Implements AGP at chosen conformance level.

---

## 8. Document conventions and contribution

- **[OPEN]** — open question requiring decision.
- **[DEFER]** — explicitly out of scope for v0.1.
- **MUST / MUST NOT / SHOULD / MAY** — used per RFC 2119 conventions.

This document lives at `openagp/spec/concept-and-spec.md` once the repo exists. Until then, it lives in the ZAK working folder. Iterations follow the same versioning conventions defined in §3.9.

Contributions welcome by PR once the repo is public. Until then: discuss with founding team.

---

## 9. Appendix A — Worked example

Here is an end-to-end trace showing AGP in action for a single high-stakes action.

**Scenario:** Acme Inc. uses ZAK as its plane. Anthropic (L1+L2) and OpenAI (L1) are registered vendors. An Acme employee asks Claude to send an email summarizing a quarterly report to an external recipient.

**Step 1 — Policy push (one-time, periodic).**
Acme writes the policy below and ZAK signs and pushes it to Anthropic via `POST /agp/v0/policy`.

```yaml
agp_policy_version: "0.1"
policy_id: acme-prod
applies_to:
  vendors: ["anthropic.com"]
  agents: ["*"]
rules:
  - id: rule_external_email_review
    when:
      action.tool_name: email.send
      action.target_resource:
        domain_not_in: ["acme.com"]
    then:
      decision: blocked
      reason: "external email requires human review"
```

Anthropic acknowledges, returns the policy hash, applies it to all subsequent agent invocations within the SLA.

**Step 2 — Action attempted.**
Claude, mid-conversation, decides to call `email.send` with `target_resource: external@customer.com`.

**Step 3 — Vendor evaluates against active policy.**
Anthropic's AGP receiver matches the rule. Decision: `blocked`.

**Step 4 — Event emitted (Flow A).**
Anthropic POSTs to ZAK's `/agp/v0/events`:

```json
{
  "agp_version": "0.1",
  "event_id": "evt_01JFXY8B...",
  "occurred_at": "2026-08-12T14:23:11.412Z",
  "actor": {
    "vendor": "anthropic.com",
    "agent_id": "agt_claude_3_5_sonnet",
    "human_principal": "user_hash_2d4f..."
  },
  "action": {
    "type": "tool_call",
    "tool_name": "email.send",
    "target_resource": "external@customer.com",
    "input_summary": "Quarterly report summary attached",
    "input_hash": "sha256:..."
  },
  "policy": {
    "decision": "blocked",
    "rule_id": "rule_external_email_review",
    "policy_hash": "sha256:...",
    "rationale": "external email requires human review"
  },
  "lineage": { "trace_id": "trc_01JFX...", "parent_event_id": null },
  "signature": { "key_id": "anthropic-2026-q2", "alg": "Ed25519", "value": "..." }
}
```

**Step 5 — ZAK ingests, verifies, anchors.**
ZAK verifies Anthropic's signature against the registry. Adds the event to Acme's hash-chained Action Ledger. Tags with SCF controls (`AUDIT-12`, `DATA-08`). Surfaces in Acme's Console.

**Step 6 — Compliance evidence.**
Acme's compliance lead, two weeks later, generates an EU AI Act conformity report. The report cites this event as evidence of automated policy enforcement under Article 14 (Human Oversight). The auditor accepts the evidence because the event is signed, chained, and verifiable independently of ZAK.

**No bespoke integration was written.** Acme wrote one policy. Anthropic ran one policy. ZAK delivered one ledger. AGP made the wires interoperate.

---

## 10. Appendix B — How AGP relates to existing ZAK MVP work

| AGP component | Existing ZAK PRD | Relationship |
|---|---|---|
| Canonical event schema | PRD 01 — Action Ledger | Same schema. ZAK Action Ledger is the canonical AGP plane reference. |
| Vendor connectors | PRD 03 — Vendor Connectors L3 | Connectors translate non-AGP vendor signals into AGP events. Once vendors ship native AGP, connectors become thin pass-throughs. |
| Runtime emission | PRD 02 — Runtime Event Emission | ZAK Runtime emits AGP-shaped events natively. ZAK Runtime is itself an L1+ AGP vendor. |
| Console badging | PRD 04 — Console v1 | Console v1 displays per-vendor AGP conformance level (P1 in PRD 04). |
| Compliance evidence | PRD 05 — Compliance Pack | Bundles cite AGP-signed events; auditors verify via registry. |

The MVP ships AGP-compatible by accident-of-design — schemas were aligned from day one. AGP v0.1 is the formalization of work that's already happening, plus the SDK + CTS + Registry that makes it adoptable by others.

This is the intended sequencing. Build the wedge as if AGP were already a standard. When the wedge ships, the standard is real.

---

*End of document — v0.1, May 2026*
