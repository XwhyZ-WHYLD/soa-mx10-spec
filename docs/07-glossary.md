# 07 — Glossary

Terms defined here are used consistently across this specification. Italicized words within definitions are themselves defined entries.

## A

**Anchor** — A periodic commit of the chain head's running *trace.sig* to a trust authority outside the orchestrator's control, used to detect tamper in compacted segments.

**Append-only** — A storage discipline in which committed records cannot be edited, deleted, or reordered. The *Symbolic Trace Ledger* is append-only.

**Attestation** — The act of verifying that an artifact's digest matches a signed manifest entry under a trusted key. Schemas, cores, and trace events are all attested.

**Audit (phase)** — The seventh and final phase of the verification sequence; commits a turn-closing event to the *Symbolic Trace Ledger*.

## B

**Bounded Adaptation** — The discipline by which the orchestrator drops context in a deterministic precedence order (Core → Recent Turns → Critical References → History) when budget constraints require it. Bounded adaptation never erodes Core.

## C

**Canonical normalization** — The deterministic transformation applied before digesting or signing to ensure two semantically equivalent representations produce identical bytes. JSON canonicalization per RFC 8785 is recommended.

**Capability tag** — A label on a schema declaring what side effects a tool may have (e.g., `network.read`, `file.write`, `pii.access`). Capability tags must be a subset of the role's *allowed_scopes*.

**Capability delta** — The condition where a previously attested schema now requests broader capabilities. Triggers `CAPABILITY_DELTA` and requires re-attestation.

**Chain-Injection** — An attack pattern in which an adversary attempts to wrap, supersede, or fork the enhancement layer to override Core. The *Chain-Injection Firewall* (CIF) is the mitigation.

**Conformance Statement** — A signed document declaring an implementation's conformance level and choices. Required for any "SOΛ-MX10 v3.0 Level A/B conformant" claim.

**Core** — The highest-precedence layer of the *Instruction Vector*. Mutation-locked; cannot be overridden by lower layers.

**core_digest** — The hash of the canonical *Core* instructions. Recomputed at SYNC and verified to match the attested value.

## D

**Decision Record** — The portion of the *Guarded Envelope* populated by the orchestrator during processing. Records phase outcomes, violations, and final trace.sig.

**Drift** — Unauthorized change to an attested artifact. *Schema drift* is unauthorized schema change; *core drift* is unauthorized Core change.

## E

**Enhancement** — A layer of the Instruction Vector that may decorate Core behavior (formatting hints, routing preferences) but cannot override it.

**Envelope** — Short for *Guarded Envelope*.

**Execute (phase)** — The fifth phase of the verification sequence. Tool calls run only with attested schemas, schema-conformant arguments, and in-scope capabilities.

## F

**Fail-open** — The behavior in which a security check that fails or times out permits the underlying operation. SOΛ-MX10 prohibits fail-open; it requires fail-secure to *Minimal Safe Profile*.

**Fail-secure** — The opposite of fail-open: a failed or timed-out check denies the operation or enters Minimal Safe Profile.

**Fallback Preflight Gate (FPG)** — The five-step sequence that must pass before a fallback to a secondary model or vendor is invoked. See [`03-fallback-preflight.md`](03-fallback-preflight.md).

**Fleet** — A collection of model instances under unified governance (shared role definitions, manifest, ledger namespace). The unit of operation for SOΛ-MX10.

**Fleet Policy Graph** — The data structure representing role definitions and precedence relationships within a fleet. Roles are immutable nodes.

## G

**Guarded Envelope** — The canonical artifact carrying header, instruction vector, schema bundle, context bundle, and decision record. The unit of authorization in SOΛ-MX10.

## H

**HSM** — Hardware Security Module. Long-lived signing keys should be HSM-resident.

## I

**Inline guardian** — A deployment topology where the orchestrator's verification logic is co-resident with the application router or LLM client. Distinct from a remote post-processor.

**Instruction Vector** — The ordered array of instruction segments inside a Guarded Envelope. Order is significant: lower indices have higher precedence.

**Issuer** — The entity that signs an artifact (envelope, segment, manifest entry, ledger event). Identified by DID or key id.

## L

**Latency Guard** — The cross-cutting mechanism that enforces the policy-declared latency budget for the verification sequence. Engages Minimal Safe Profile on exhaustion.

**Layer** — The precedence tier of an instruction segment: `core`, `developer`, `enhancement`, or `runtime_hint`.

**Ledger** — Short for *Symbolic Trace Ledger*.

**Level A / Level B** — The two conformance levels. See [`06-conformance.md`](06-conformance.md).

## M

**Manifest** — The signed registry of attested schemas, capability taxonomies, and trust anchors active for a given policy version.

**Mesh mode** — A deployment topology where SOΛ-MX10 runs as a sidecar in a service mesh, observing RPCs to/from tool providers and LLMs.

**Minimal Safe Profile (MSP)** — The fail-secure operating mode entered on latency exhaustion or other unrecoverable verification failure: attested Core only, all tools denied, no fallback permitted, no enhancements applied.

**Mutation lock** — The property that an artifact cannot be modified once attested without a new signed update. Core is mutation-locked.

**Mutation X (MX)** — Component of the protocol's name; refers to mutation-class governance. See [README §"On the name"](../README.md).

## N

**Nonce** — A per-envelope random value, never reused within a trace chain.

## P

**Plan (phase)** — The fourth phase of the verification sequence. The model produces a plan; if it declares a fallback, the FPG intercepts.

**Policy manifest** — Active signed manifest declaring schema attestations, capability taxonomy, cryptographic choices, latency budget, and trust anchors.

**Precedence** — The total order of instruction layers: `core ⊐ developer ⊐ enhancement ⊐ runtime_hint`.

**Predecessor trace.sig** — The tail of the verified trace chain carried into a fallback envelope to maintain continuity.

**prev_hash** — The hash of the previous ledger event's full record. Together with per-event signatures, prev_hash chains the ledger.

## R

**Role Sovereignty** — The invariant that a model's declared role and core directives are mutation-locked across all fleet instances.

**Runtime hint** — The lowest-precedence instruction layer. May provide ephemeral routing or formatting information; cannot influence anything above it.

## S

**Schema attestation** — Verification that a tool/function schema's digest matches a signed manifest entry under a trusted key, with capability tags within the role's allowed scope.

**Schema Attestation Engine (SAE)** — The component responsible for schema attestation.

**Schema Bundle** — The portion of the Guarded Envelope listing tool/function schemas available to the model in this turn.

**Scope** — A set of capability tags. Each segment declares its scope; segments out of *allowed_scopes* are stripped.

**Sidecar mode** — See *Mesh mode*.

**Signature** — Cryptographic proof of issuer identity. Required on envelopes, signed segments, manifest entries, and ledger events.

**Sovereign** — Component of the protocol's name; refers to the Role Sovereignty invariant.

**Stealth principle** — Trace events and uncertainty signals are not surfaced conversationally. Detection of any guard state by an adversary interacting with the model is prohibited.

**Symbolic Trace Ledger (STL)** — The append-only, hash-chained record of decisive events. See [`04-trace-integrity.md`](04-trace-integrity.md).

**SYNC (phase)** — The first phase of the verification sequence. Recomputes core_digest and verifies against the attested value.

## T

**trace.sig** — Two related concepts:
1. The per-event hash that chains an event to its predecessor (`prev_hash` field)
2. The running value summarizing the entire chain to date

Context determines which is meant. The running value is what fallback preflight verifies for continuity.

**Trust anchor** — A key, KMS, or external service trusted to verify signatures. Manifests declare active trust anchors.

**Trust predicate** — An implementation-defined function that gates uncertainty disclosure. Must require, at minimum, authenticated identity and explicit authorization.

## U

**Uncertainty Scanner (US)** — The bounded-cost component computing per-turn uncertainty score `u`.

**Uncertainty threshold (θ)** — The policy-declared threshold above which `UNCERTAINTY_FLAG` is committed. Default `0.05`.

## V

**Verification sequence** — The seven phases SYNC → VERIFY → ENHANCE → PLAN → EXECUTE → ATTEST → AUDIT, executed deterministically per turn.

**VERIFY (phase)** — The second phase. Schema attestation, chain-injection detection, and uncertainty scan run here.

## Symbol reference

| Symbol | Meaning |
|--------|---------|
| `H(·)` | The protocol hash function (SHA-256 minimum) |
| `θ` | Uncertainty threshold |
| `t_budget` | Latency budget for the verification sequence |
| `u` | Per-turn uncertainty score, ∈ [0, 1] |
| `⊐` | Strict precedence relation between layers |
| `⊆` | Scope subset (segment scope ⊆ role allowed scopes) |
| `Λ` | Lambda — the protocol mark; lambda-calculus lineage |
| `Φ` | Semantic validator within the Chain-Injection Firewall |
| `G_inj` | Grammar of override/wrapper constructs |
| `Ξ` | Context-transfer function used in fallback preflight |
