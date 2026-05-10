# 03 — Fallback Preflight Sequence

When an orchestrator switches from a primary model or vendor to a secondary one, the security envelope does not transfer automatically. Vendor-specific runtimes have different system-prompt semantics, different schema formats, different rate limits, and different defaults. A fallback that does not reapply the full verification sequence is a **fallback bypass** — one of the highest-impact attack vectors in fleet operation.

The Fallback Preflight Gate (FPG) is the deterministic sequence that **MUST** complete and pass before any fallback invocation.

## 1. Trigger conditions

The FPG **MUST** be invoked whenever any of the following occur:

- The primary model returns a hard error (timeout, quota, content policy refusal)
- The orchestrator's policy declares a routing rule that selects a different model or vendor
- The plan emitted in PLAN names a secondary model or external agent
- A circuit breaker on the primary path opens

The FPG **MUST NOT** be skipped under any latency pressure. If the latency budget is exhausted before the FPG completes, the orchestrator **MUST** abort the fallback and serve the request under the Minimal Safe Profile from the primary model.

## 2. The five-step sequence

```
                  ┌────────────────────────┐
                  │    Fallback Triggered  │
                  └───────────┬────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │ 1. Rehydrate Core      │
                  │    Instructions        │
                  └───────────┬────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │ 2. Re-pull & Verify    │
                  │    Signed Schema       │
                  └───────────┬────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │ 3. Validate Trace      │
                  │    Signature Continuity│
                  └───────────┬────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │ 4. Abort on Failure /  │
                  │    Proceed on Pass     │
                  └───────────┬────────────┘
                              │ (pass)
                              ▼
                  ┌────────────────────────┐
                  │ 5. Invoke Secondary    │
                  │    Model               │
                  └────────────────────────┘
```

### 2.1 Rehydrate Core Instructions

The orchestrator **MUST**:
1. Reload the canonical core instructions for the active role from the trusted source
2. Recompute `core_digest` and verify it matches the value attested at the original SYNC for this turn
3. Re-render the core instructions in the format required by the secondary model/vendor (system prompt, system message, etc.) using a vendor-specific adapter that does not alter semantic content

If the rehydrated `core_digest` does not match, **MUST** raise `CORE_DRIFT_AT_FALLBACK` and **MUST NOT** invoke the fallback.

### 2.2 Re-pull and Verify Signed Schema

The orchestrator **MUST**:
1. Re-fetch every schema in the schema_bundle from the manifest source (not from cache, unless the cache is itself signature-attested and within its freshness window)
2. Recompute each `schema_digest`
3. Re-verify each `manifest_sig` under the trust anchor that the secondary vendor's runtime will observe
4. Re-translate schema definitions into the secondary vendor's tool-definition format using a deterministic adapter

Any digest mismatch, signature failure, or capability_tag delta **MUST** abort the fallback. The orchestrator **MUST** commit `SCHEMA_REVERIFICATION_FAILED` to the trace ledger.

### 2.3 Validate Trace Signature Continuity

The orchestrator **MUST**:
1. Recompute the running `trace.sig` from the genesis event of this turn through the most recent committed event
2. Verify the recomputed value matches the stored running value
3. Verify that each event in the chain is signed by a trusted issuer
4. Verify monotonic timestamps and strictly increasing sequence numbers

If any check fails, **MUST** raise `TRACE_BREAK` and abort the fallback.

### 2.4 Abort on Failure / Proceed on Pass

The decision is binary. If any of steps 1–3 fail:

- The fallback **MUST NOT** proceed
- The orchestrator **MUST** commit `FALLBACK_DENIED` to the trace ledger with the failing step
- The host application **MUST** receive a structured error indicating fallback unavailability — **NOT** a conversational explanation

If all three pass, the orchestrator proceeds to step 5 with a fresh envelope constructed for the secondary runtime.

### 2.5 Invoke Secondary Model

The fallback envelope **MUST** carry:
- The same `role_id` and `fleet_id` as the original
- A new `envelope_version` of the same value (`3.0`)
- A `policy_version` matching the original (no policy upgrade during a fallback)
- A new `nonce` (nonces are per-envelope)
- A `header.predecessor_trace_sig` field carrying the tail of the verified chain
- A fresh outer envelope_sig from the orchestrator

The secondary model's response **MUST** flow through the standard EXECUTE → ATTEST → AUDIT phases. The secondary model is now operating under SOΛ-MX10 governance for the remainder of the turn.

## 3. Multi-tier fallback

Fallback chains longer than two hops (primary → secondary → tertiary → ...) are permitted. Each hop **MUST** repeat the full preflight sequence against the next runtime. The trace.sig chain accumulates events monotonically — there is no "reset" between hops.

A fallback chain depth of more than `max_fallback_depth` (policy-declared, default `3`) **MUST** abort.

## 4. What the FPG does not do

The FPG does not:

- Translate semantic differences between models. If the primary and secondary disagree on a tool's behavior at the model level, the FPG does not reconcile it. That is the host application's concern.
- Retry the primary. If the primary failed, FPG does not attempt recovery on the primary; it hands off cleanly to the secondary if and only if preflight passes.
- Synchronize conversational state. Conversation history is part of the context_bundle; the FPG only verifies its provenance, not its meaning.

## 5. Example abort cases

| Symptom | Step that catches it | Trace event |
|---------|---------------------|-------------|
| Core instructions edited in registry between SYNC and fallback | 1 | `CORE_DRIFT_AT_FALLBACK` |
| Schema cache poisoned with valid-looking but unsigned entry | 2 | `SCHEMA_REVERIFICATION_FAILED` |
| Adversary attempts to splice a forged trace event mid-chain | 3 | `TRACE_BREAK` |
| Secondary vendor's signing key revoked since manifest issuance | 2 | `SCHEMA_REVERIFICATION_FAILED` |
| Latency budget exhausted during step 2 | latency guard | `BUDGET_EXHAUSTED` (and Minimal Safe Profile) |

## 6. Conformance

An implementation conforms to this section if and only if:
- All five steps execute in order on every fallback trigger
- Step 4 is binary (no partial fallback, no degraded fallback)
- Trace events are committed for every abort case
- The Minimal Safe Profile is engaged on latency exhaustion rather than fail-open
