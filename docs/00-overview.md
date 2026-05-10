# 00 — Overview

## What SOΛ-MX10 is

SOΛ-MX10 is a protocol-layer specification for governing the execution of computational operations issued by AI-orchestrated systems — specifically, fleets of large language model (LLM) instances operating across multiple roles, tools, and vendors.

It defines a deterministic verification sequence that every decisive instruction, schema, and fallback transition must pass before it influences execution. The protocol is **model-agnostic** and **vendor-neutral**: it requires no access to model weights, internal logits, or vendor-private interfaces. It operates as an **inline guardian** between the orchestrator and downstream tools, APIs, and fallback models.

## What it is not

- **Not a content moderation system.** It does not classify outputs as safe or unsafe. Output filtering is the host application's responsibility.
- **Not a model evaluation framework.** It does not score model capability, alignment, or correctness.
- **Not a prompt firewall.** It governs orchestration-level artifacts (instructions, schemas, fallbacks, trace events) — not raw prompt strings in isolation.
- **Not a single-instance defense.** Every primitive in the protocol is fleet-aware. Per-session or per-prompt enforcement is a degenerate case.

## The core insight

Conventional access control authorizes an *actor*: a user, a service, a token. SOΛ-MX10 authorizes a *transition*: a bound `(current_state, next_state, operation)` triple, signed by an attested issuer, witnessed by a hash-chained ledger.

This shift matters because autonomous agents move faster than actors. Once an agent is authenticated, it can chain hundreds of operations in seconds. Token-bounded authorization cannot describe what should and should not happen across that chain. State-transition authorization can.

## The eight modules

| Module | Purpose |
|--------|---------|
| Role Governance | Mutation-locks declared role across fleet instances |
| Bounded Adaptation | Enforces deterministic precedence (Core → Recent → Critical → History) |
| Schema Verification | Validates signed tool/function schemas against a manifest |
| Chain-Injection Detection | Blocks wrapper/override constructs at grammar and semantic layers |
| Fallback Control | Reapplies the security envelope before invoking a secondary model |
| Uncertainty Management | Logs uncertainty above threshold internally; gates disclosure on a trust predicate |
| Trace Integrity | Maintains an append-only `trace.sig` chain across all calls |
| Latency Guard | Enforces a deterministic execution budget; fails to a minimal safe profile |

Each module is specified independently in the documents that follow. Together they form a single deterministic verification pipeline.

## Reading order

1. This overview
2. [`01-protocol-specification.md`](01-protocol-specification.md) — the protocol itself
3. [`02-guarded-envelope.md`](02-guarded-envelope.md) — artifact schema
4. [`03-fallback-preflight.md`](03-fallback-preflight.md) — the cross-vendor preflight sequence
5. [`04-trace-integrity.md`](04-trace-integrity.md) — the symbolic trace ledger
6. [`05-threat-model.md`](05-threat-model.md) — adversary model and mitigations
7. [`06-conformance.md`](06-conformance.md) — what an implementation must do to claim conformance
8. [`07-glossary.md`](07-glossary.md) — terms

## Versioning

This specification follows semantic versioning at the protocol level:

- **Major** — breaking changes to envelope schema, ledger event format, or the verification sequence
- **Minor** — additive changes (new optional fields, new module behaviors that preserve existing contracts)
- **Patch** — clarifications and editorial fixes

v3.0 is the first publicly released stable specification. Internal v1 and v2 are not interoperable with v3.x and are not supported.
