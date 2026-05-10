# 06 — Conformance

This document defines what an implementation must do to claim conformance to SOΛ-MX10 v3.0. It also defines the **Conformance Statement** that conformant implementations must publish.

## 1. Levels

There are two conformance levels:

### 1.1 Level A — Core Conformance

A Level A implementation:

- **MUST** execute the seven-phase verification sequence (SYNC → VERIFY → ENHANCE → PLAN → EXECUTE → ATTEST → AUDIT) in the specified order
- **MUST** parse and validate Guarded Envelopes per [`02-guarded-envelope.md`](02-guarded-envelope.md)
- **MUST** enforce the layer precedence relation (Core ⊐ Developer ⊐ Enhancement ⊐ RuntimeHints)
- **MUST** implement schema attestation including capability-tag scoping
- **MUST** implement the Chain-Injection Firewall (both grammar and semantic stages)
- **MUST** implement the Fallback Preflight Gate per [`03-fallback-preflight.md`](03-fallback-preflight.md)
- **MUST** maintain a Symbolic Trace Ledger conformant to [`04-trace-integrity.md`](04-trace-integrity.md)
- **MUST** implement the Latency Guard with Minimal Safe Profile (no fail-open)
- **MUST** publish a Conformance Statement (§3)

### 1.2 Level B — Federated Conformance

A Level B implementation meets all Level A requirements **and**:

- **MUST** support external trace anchoring to a customer-controlled or independent trust authority
- **MUST** support cross-implementation ledger verification (its exported chain is verifiable by any other Level B verifier given public keys)
- **MUST** support per-tenant isolation of manifests, ledger namespaces, and signing keys
- **SHOULD** support post-quantum signature suites as a configurable option

Level B is intended for multi-tenant, regulated, or federated deployments where audit must cross organizational boundaries.

## 2. Conformance test points

A conformant implementation **MUST** correctly handle each of the following test points. This list is normative — failure on any point invalidates the conformance claim.

### 2.1 Envelope handling

| # | Test |
|---|------|
| C-E-01 | Reject envelope with unsupported `envelope_version` |
| C-E-02 | Reject envelope with invalid outer signature |
| C-E-03 | Reject envelope with `core_digest` mismatch at SYNC |
| C-E-04 | Strip enhancement segment that proposes Core override |
| C-E-05 | Strip segment whose scope is not a subset of role allowed_scopes |
| C-E-06 | Reject runtime_hint that attempts precedence elevation |

### 2.2 Schema attestation

| # | Test |
|---|------|
| C-S-01 | Reject schema whose digest does not match manifest entry |
| C-S-02 | Reject schema whose manifest signature does not verify |
| C-S-03 | Reject schema whose capability_tags exceed role allowed_scopes |
| C-S-04 | Detect and flag CAPABILITY_DELTA on previously attested schema |
| C-S-05 | Apply canonical normalization before digest computation |
| C-S-06 | Reject use of a revoked manifest entry |

### 2.3 Chain-injection

| # | Test |
|---|------|
| C-C-01 | Detect imperative override constructs (grammar) |
| C-C-02 | Detect wrapper-introduction patterns (grammar) |
| C-C-03 | Detect semantic intent to alter precedence |
| C-C-04 | Strip flagged segment before PLAN |
| C-C-05 | Commit MUTATION_ATTEMPT or CHAIN_INJECTION_* event |
| C-C-06 | Do not surface detection conversationally |

### 2.4 Fallback preflight

| # | Test |
|---|------|
| C-F-01 | Re-run all five FPG steps on every fallback trigger |
| C-F-02 | Abort fallback on any step failure (no partial fallback) |
| C-F-03 | Carry predecessor_trace_sig in fallback envelope header |
| C-F-04 | Reject fallback chain depth exceeding policy max |
| C-F-05 | Engage Minimal Safe Profile (not fail-open) on budget exhaustion during FPG |

### 2.5 Trace integrity

| # | Test |
|---|------|
| C-T-01 | Compute valid prev_hash on every event |
| C-T-02 | Reject events whose prev_hash does not match predecessor |
| C-T-03 | Strictly increasing seq with no gaps within a chain |
| C-T-04 | Genesis event uses prev_hash = "0" * 64 |
| C-T-05 | Running trace.sig recomputable from events alone |
| C-T-06 | Compaction preserves running trace.sig continuity |
| C-T-07 | Anchor commits do not alter chain semantics |

### 2.6 Uncertainty and stealth

| # | Test |
|---|------|
| C-U-01 | Commit UNCERTAINTY_FLAG when u > θ |
| C-U-02 | No conversational variation correlates with u |
| C-U-03 | Disclosure gated by trust predicate |
| C-U-04 | Trace events not surfaced conversationally |

### 2.7 Latency guard

| # | Test |
|---|------|
| C-L-01 | Verification completes within declared t_budget on synthetic baseline |
| C-L-02 | Engage Minimal Safe Profile on exhaustion |
| C-L-03 | Commit BUDGET_EXHAUSTED + MIN_SAFE_ENGAGED on exhaustion |
| C-L-04 | Do not fail open under any timing condition |

A test harness implementing the above is a separate work product and is not included in this specification.

## 3. The Conformance Statement

A conformant implementation **MUST** publish a Conformance Statement, accessible at a stable URL or in its public documentation. The statement **MUST** declare:

```yaml
conformance_statement:
  spec_version:    "3.0"
  level:           "A" | "B"

  implementation:
    name:          "<implementation name>"
    version:       "<implementation version>"
    vendor:        "<vendor or maintainer>"
    contact:       "<security contact>"

  cryptographic_choices:
    hash_function: "SHA-256" | "SHA-3-256" | "BLAKE3"
    signature_alg: "Ed25519" | "HMAC-SHA256" | "..."
    canonicalization: "RFC 8785" | "<other, with reference>"

  registry:
    type:          "signed-api" | "blockchain" | "kms-attested-cache" | "<other>"
    description:   "<one-line>"

  ledger:
    storage:       "<append-only storage type>"
    anchoring:     "internal-only" | "external"
    anchor_target: "<URI or 'n/a'>"

  latency_budget_ms: <integer>

  capability_taxonomy: "<reference to taxonomy used for capability_tags>"

  test_point_results:
    # one entry per test point, indicating pass/fail/notapplicable
    C-E-01: "pass"
    # ...

  out_of_scope:
    - "<any test point claimed not-applicable, with rationale>"

  attestation:
    signed_by:     "<key id>"
    signed_at:     "<RFC 3339 UTC>"
```

The Conformance Statement itself **SHOULD** be signed by the implementation maintainer and published alongside the implementation.

## 4. Use of the conformance claim

Implementations conforming to this specification **MAY** describe themselves as "SOΛ-MX10 v3.0 Level A conformant" or "Level B conformant." Use of the SOΛ-MX10 mark in marketing or product naming requires separate trademark license from XWHYZ Research / WHYLD.

False or unverifiable conformance claims should be reported via the disclosure process in [`SECURITY.md`](../SECURITY.md).

## 5. Versioning

This conformance section is versioned with the specification. A "v3.0 Level A" claim is specific to v3.0; conformance to a later version requires re-publishing the statement against that version's test points.
