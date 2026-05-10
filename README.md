[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20103197.svg)](https://doi.org/10.5281/zenodo.20103197)
# SOΛ-MX10 v3.0

**Deterministic Execution-Control Protocol for AI-Orchestrated Systems**

SOΛ-MX10 is a specification for a fleet-scale governance layer that gates execution of computational operations in AI-orchestrated environments. No tool call, no enhancement, no fallback runs unless it is bound to a valid, signed state-transition artifact and survives a deterministic verification sequence.

This repository is **specification-only**. It defines the protocol, artifact formats, conformance criteria, and a reference algorithm. Production implementations and defense-specific embodiments are out of scope.

---

## What it is

Conventional AI safety operates per session, per instance, per prompt. SOΛ-MX10 operates at the orchestration layer — across an entire fleet of model instances, tool surfaces, and vendor fallbacks — and treats every decisive instruction as an artifact that must be attested before it influences execution.

The protocol enforces six invariants:

1. **Role Sovereignty** — A model's declared role and core directives are mutation-locked. Enhancements may decorate; they cannot override.
2. **Bounded Adaptation** — Context is dropped in a strict precedence order (Core → Recent Turns → Critical References → History) — never in a way that erodes core identity.
3. **Schema Attestation** — Every tool/function schema is digested, signed, and matched against a manifest. Drift triggers quarantine.
4. **Chain-Injection Resistance** — Wrapper or override constructs (grammar-level and semantic) are detected and stripped before planning.
5. **Fallback Invariance** — Cross-vendor fallback is permitted only after a preflight sequence reapplies the full security envelope to the new runtime.
6. **Trace Continuity** — Every decisive step commits to an append-only, hash-chained ledger that survives compaction.

A latency guard enforces deterministic ordering within a bounded budget. On exhaustion the system fails to a **minimal safe profile** — never fail-open.

---

## Why it exists

LLM fleets are now mission-critical infrastructure. The attack surface expands faster than per-instance defenses can cover:

- **Role drift** — multi-turn prompts that gradually shift an agent's identity or scope.
- **Schema spoofing** — runtime-injected or silently mutated tool definitions that expand capabilities.
- **Chain injection** — wrapper prompts that attempt to supersede or replace the security layer itself.
- **Fallback bypass** — coercing the orchestrator into switching to an unguarded vendor or model.
- **Trace loss** — handoffs across tools or models where the audit chain breaks.
- **Uncertainty exploits** — probing the model's exposed uncertainty signals to fingerprint defenses.

No public framework before v2.0 combined fleet role governance, signed-schema verification, chain-injection blocking, fallback preflight, and cryptographic trace continuity into a single deterministic, latency-bounded sequence. SOΛ-MX10 is that combination, specified as a portable, model-agnostic protocol.

---

## Conceptual model

```
┌──────────────────────────────────────────────────────────────┐
│                      Guarded Envelope                        │
│  ┌─────────┐  ┌──────────────┐  ┌─────────┐  ┌────────────┐ │
│  │ Header  │  │ Instruction  │  │ Schema  │  │  Context   │ │
│  │ role_id │  │   Vector     │  │ Bundle  │  │  Bundle    │ │
│  │ digest  │  │ (ordered)    │  │ (digested)│ (provenanced) │
│  └─────────┘  └──────────────┘  └─────────┘  └────────────┘ │
└──────────────┬───────────────────────────────────────────────┘
               │
               ▼
   SYNC → VERIFY → ENHANCE → PLAN → EXECUTE → ATTEST → AUDIT
               │                                       │
               └────── trace.sig hash chain ───────────┘
```

The **Guarded Envelope** is the unit of authorization. Each turn flows through seven deterministic phases. At every decisive step a `trace.sig` event is committed to an append-only **Symbolic Trace Ledger** — forming a verifiable continuity chain across tools, vendors, and fallback boundaries.

An operation without a verifying envelope is not "denied" — it is *not yet authorized to exist*. There is no fallback path.

---

## Repository structure

| Path | Contents |
|------|----------|
| [`docs/`](docs/) | Specification — overview, protocol, envelope schema, preflight, trace, threat model, conformance, glossary |
| [`code/`](code/) | Reference pseudocode and a non-production Python SDK sketch |
| [`legal/`](legal/) | License (`LICENSE.md`) and naming/trademark notices |
| [`CHANGELOG.md`](CHANGELOG.md) | Version history and breaking-change record |
| [`SECURITY.md`](SECURITY.md) | Disclosure process and threat-model summary |

The `docs/` files are the authoritative specification. Code under `code/` is illustrative — it shows shape, not bytes-on-the-wire.

---

## Scope and non-scope

**In scope (v3.0):**
- Protocol-level invariants and verification sequence
- Guarded Envelope artifact schema
- Trace ledger event format and chain semantics
- Fallback preflight contract
- Conformance criteria for implementations

**Out of scope (intentionally):**
- Defense-specific or sovereign-tier fallback configurations
- Stealth-audit correlation algorithms
- Threshold tuning parameters for uncertainty scoring
- Production-grade reference implementations
- Vendor-specific integration code

The omitted material is held as trade secret or reserved for restricted continuations. See [`docs/05-threat-model.md`](docs/05-threat-model.md) for the public threat model.

---

## Status

- **Version:** 3.0 (first public stable spec)
- **Stage:** Public specification — open for review, citation, and independent reference-implementation work
- **Stability:** v3.x freezes the wire-level envelope and ledger event formats. Earlier internal versions (v1, v2) are not interoperable
- **Patent posture:** A USPTO provisional application covering the underlying invention has been prepared by the inventor. This specification documents the publicly disclosable subset

---

## Topics

`distributed-systems` · `state-machine` · `access-control` · `execution-control` · `ai-orchestration` · `authorization-protocol` · `cryptographic-authorization` · `execution-gating` · `systems-architecture` · `computer-security` · `specification`

---

## Citation

> Roshan George, Thomas. *SOΛ-MX10: Deterministic Execution-Control Protocol 
> for AI-Orchestrated Systems.* XWHYZ Research / WHYLD, 2026.  
> DOI: [10.5281/zenodo.20103197](https://doi.org/10.5281/zenodo.20103197)  
> GitHub: https://github.com/XwhyZ-WHYLD/soa-mx10-spec

---

## Security

See [`SECURITY.md`](SECURITY.md) for the disclosure process and threat-model summary. Design-level issues — authorization bypass, replay, state rollback, signature substitution, chain-injection escapes — are in scope. Defects in third-party implementations are not.

---

## License

See [`legal/LICENSE.md`](legal/LICENSE.md). Trademark and naming rights to "SOΛ-MX10" and the SOΛ-MX family are reserved by XWHYZ.

---

## Maintainers

**XWHYZ Research / WHYLD** — independent research on AI–human coexistence infrastructure, sovereign-grade protocol design, and agentic governance primitives.

---

## On the name

**SOΛ-MX10** — *Sovereign · Logic-locked · Mutation-resistant · Tier 10*. The Λ marks lambda-calculus lineage; the MX prefix denotes mutation-class governance; the numeric tier reflects the integrity tier the protocol targets. The name is functional, not decorative — it encodes the design intent.

