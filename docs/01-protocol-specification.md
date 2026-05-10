# 01 — Protocol Specification

## 1. Conformance language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this specification are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

## 2. The verification sequence

Every turn processed by a SOΛ-MX10-conformant orchestrator **MUST** flow through seven phases in this exact order:

```
SYNC → VERIFY → ENHANCE → PLAN → EXECUTE → ATTEST → AUDIT
```

The order is deterministic. Phases **MUST NOT** be reordered, parallelized across phase boundaries, or skipped. Within a phase, sub-steps **MAY** run in parallel where stated.

### 2.1 SYNC

The orchestrator **MUST**:

1. Compute `core_digest = H(core_instructions)` where `H` is the protocol's hash function (see §6).
2. Compare against the last attested `core_digest` for this fleet/role.
3. On mismatch, **MUST** raise `CORE_DRIFT` and **MUST** revert to the last trusted core unless the change is accompanied by a signed policy update from an authorized issuer.

### 2.2 VERIFY

The orchestrator **MUST**, in any order:

1. Run the **Schema Attestation Engine** over every tool/function schema in the inbound envelope (§3).
2. Run the **Chain-Injection Firewall** over the instruction vector (§4).
3. Run a bounded **Uncertainty Scan** producing a score `u ∈ [0, 1]` (§5).

VERIFY **MUST** complete within the share of the latency budget allocated to it by the implementation.

### 2.3 ENHANCE

The orchestrator **MUST** apply approved enhancement segments in declared scope only. Any segment proposing to mutate Core (override, replace, supersede, "ignore prior") **MUST** be stripped. The stripped event **MUST** be committed to the trace ledger as `MUTATION_ATTEMPT`.

### 2.4 PLAN

The model produces a plan. If the plan declares a fallback to a secondary model or vendor, the orchestrator **MUST** invoke the **Fallback Preflight Gate** (see [`03-fallback-preflight.md`](03-fallback-preflight.md)) before allowing the fallback to proceed.

### 2.5 EXECUTE

Tool calls **MUST** execute only if:
- The schema is attested (§3) AND
- The arguments conform to the schema in strict mode (no `additionalProperties` unless the schema declares them) AND
- The capabilities required are within the role's allowed scope.

Any failure **MUST** abort the call and commit `EXECUTE_DENIED` to the trace ledger.

### 2.6 ATTEST

Outputs **MUST** be labeled with:
- Tool provenance (which schema, which signature)
- Freshness window (timestamp + validity)
- The trace event ID that witnessed the call

### 2.7 AUDIT

The orchestrator **MUST** commit a turn-closing event to the trace ledger including:
- Final `trace.sig` for the turn
- Aggregate uncertainty score
- Any violations or drops not already individually committed

## 3. Schema attestation

A tool/function schema is **attested** if and only if all of the following hold:

1. The schema, after canonical normalization (§3.1), produces a digest matching an entry in the active **manifest**.
2. The manifest entry's signature verifies under a key trusted by the role's policy.
3. The manifest entry has not been revoked.
4. The capability tags declared by the manifest entry are a subset of the role's allowed scope.

Unattested schemas **MUST NOT** be invoked. Capability deltas (a previously attested schema now requesting broader capabilities) **MUST** trigger `CAPABILITY_DELTA` and require re-attestation under the new capability set.

### 3.1 Canonical normalization

Before digesting, schemas **MUST** be normalized:
- JSON keys sorted lexicographically at every depth
- Description fields trimmed of leading/trailing whitespace
- Default values explicitly serialized (no implicit nulls)
- Encoding fixed at UTF-8 NFC

Implementations **MUST** use the same normalization function for digest computation and digest verification.

## 4. Chain-injection detection

The Chain-Injection Firewall (CIF) **MUST** apply two stages to every instruction segment:

### 4.1 Grammar stage

A typed grammar `G_inj` recognizes wrapper/override constructs. The grammar **MUST** at minimum cover:
- Imperative override verbs targeting the system layer ("override", "replace", "ignore", "disable", "supersede")
- Wrapper introductions ("you are now", "from now on you", "load new system")
- Defense-disable patterns ("turn off", "deactivate", "bypass")
- Control-flow markers attempting to redirect precedence

Strings derivable from `G_inj` **MUST** be flagged as `CHAIN_INJECTION_GRAMMAR`.

### 4.2 Semantic stage

A semantic validator `Φ` **MUST** evaluate intent against the active precedence and scopes. A segment **MUST** be flagged as `CHAIN_INJECTION_SEMANTIC` if it expresses intent to:
- Alter the precedence relation (Core ⊐ Enhancement ⊐ RuntimeHints)
- Disable, suspend, or replace the SOΛ-MX10 verification sequence
- Inject a new outer wrapper layer not present in the manifest

### 4.3 Disposition

Any segment flagged at either stage **MUST** be stripped before PLAN. The strip event **MUST** be committed to the trace ledger. The orchestrator **MUST NOT** surface the detection to the conversational interface (see §5 on stealth principle).

## 5. Uncertainty management

The orchestrator **MUST** compute a per-turn uncertainty score `u ∈ [0, 1]` derived from at least:
- Output entropy signals
- Contradiction-with-core heuristics

If `u > θ` (where `θ` is set by policy, typical default `0.05`), the orchestrator **MUST**:
- Commit `UNCERTAINTY_FLAG` to the trace ledger with the score and a contextual snapshot
- **NOT** modify the conversational output to indicate the flag, and **MUST NOT** disclose the flag to the requestor unless a **trust predicate** evaluates true

The trust predicate is implementation-defined but **MUST** require, at minimum, authenticated identity AND explicit authorization request. Implicit disclosure (e.g., generic "I'm not sure" hedging that varies with `u`) **MUST NOT** correlate with `u`. This prevents adversaries from probing internal guard state through conversational fingerprinting.

## 6. Cryptographic primitives

Conformant implementations **MUST** support:

- **Hash function** — SHA-256 minimum. SHA-3 and BLAKE3 **MAY** be offered as alternatives. The active hash function **MUST** be declared in the policy manifest.
- **Signatures** — HMAC-SHA256 minimum for symmetric attestation; Ed25519 **RECOMMENDED** for asymmetric. Post-quantum-safe signatures (CRYSTALS-Dilithium, etc.) **MAY** be offered.
- **Key custody** — Long-lived signing keys **SHOULD** be HSM-resident in production deployments.

The chosen primitives **MUST** be declared in the policy manifest and **MUST** be the same throughout a single trace chain. Mid-chain primitive substitution **MUST** be treated as `TRACE_BREAK`.

## 7. Latency guard

The verification sequence **MUST** complete within a **policy-declared latency budget** `t_budget`. The default for general-purpose deployments is `t_budget = 3000ms`; defense and high-sensitivity deployments **MAY** declare lower bounds.

On exhaustion the orchestrator **MUST**:
- Enter the **Minimal Safe Profile**: attested core only, all tools denied, no fallback permitted, no enhancement segments applied
- Commit `BUDGET_EXHAUSTED` to the trace ledger
- Continue serving the request under the minimal profile rather than failing the request open

Fail-open behavior is non-conformant.

## 8. State machine summary

```
                ┌─────────────────────────────────────┐
                │            Inbound turn             │
                └──────────────────┬──────────────────┘
                                   │
                                   ▼
                              ┌─────────┐
                              │  SYNC   │── core_digest mismatch ──▶ revert
                              └────┬────┘
                                   ▼
                              ┌─────────┐
                              │ VERIFY  │── any failure ──▶ strip / deny
                              └────┬────┘
                                   ▼
                              ┌─────────┐
                              │ ENHANCE │── mutation attempt ──▶ strip
                              └────┬────┘
                                   ▼
                              ┌─────────┐
                              │  PLAN   │── fallback declared ──▶ FPG
                              └────┬────┘
                                   ▼
                              ┌─────────┐
                              │ EXECUTE │── unattested ──▶ deny
                              └────┬────┘
                                   ▼
                              ┌─────────┐
                              │ ATTEST  │
                              └────┬────┘
                                   ▼
                              ┌─────────┐
                              │  AUDIT  │── commit trace event ──▶ STL
                              └─────────┘

   Cross-cutting:  Latency Guard (every phase)  ·  trace.sig (every event)
```

## 9. Implementation flexibility

This specification fixes the **contracts**: ordering, invariants, artifact shape, ledger semantics, conformance criteria. It does not fix:

- The internal architecture of any module (a SAE may be a service, a library, or compiled-in)
- The deployment topology (inline vs. sidecar vs. mesh — see [`07-glossary.md`](07-glossary.md))
- The choice of registry technology (signed API, blockchain, KMS-attested cache)
- Vendor-specific integration mechanics

Implementations **MUST** publish a Conformance Statement (see [`06-conformance.md`](06-conformance.md)) describing their choices.
