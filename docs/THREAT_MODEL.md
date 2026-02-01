# SOA-MX10 v3.0  
## Threat Model and Security Assumptions

---

## 1. Purpose

This document defines the **threat model**, **security assumptions**, and **explicit scope boundaries** of the SOA-MX10 v3.0 execution-control protocol.

The purpose of this document is to:
- clarify which threats the protocol is designed to mitigate
- state which threats are explicitly out of scope
- demonstrate disciplined security reasoning

This document does **not** describe implementation-specific defenses.

---

## 2. Security Philosophy

SOA-MX10 is designed under the assumption that:
- AI components may behave adversarially
- AI outputs cannot be trusted
- policy or prompt-based controls are insufficient
- execution authority must be enforced externally

The protocol focuses on **execution control**, not intent detection or behavior correction.

---

## 3. Assets to Protect

The protocol is designed to protect the following assets:

- integrity of execution paths
- authorization of computational operations
- correctness of execution order
- auditability of execution events
- prevention of unauthorized tool invocation

The protocol does **not** attempt to protect AI internal state or reasoning quality.

---

## 4. Adversary Model

### 4.1 In-Scope Adversaries

The following adversaries are explicitly considered:

#### A. Adversarial AI Output
- AI generates deceptive, malformed, or malicious proposals
- AI attempts to induce unauthorized execution
- AI attempts to exploit fallback mechanisms

#### B. Malformed Authorization Artifacts
- replayed State Transition Tokens
- forged or tampered STTs
- stale or out-of-order authorization attempts

#### C. Execution Bypass Attempts
- direct execution calls bypassing authorization
- reuse of prior Execution Receipts
- unauthorized fallback invocation

#### D. State Manipulation Attempts
- counter reuse
- state lineage corruption
- fork attempts in execution history

---

## 5. Out-of-Scope Threats

The following threats are explicitly out of scope:

- operator compromise
- insider threats
- compromised deployment environment
- hardware-level attacks
- cryptographic primitive failures
- denial-of-service attacks

These exclusions are intentional and align with standard security boundary definitions.

---

## 6. Threat Mitigations

| Threat | Mitigation Mechanism |
|------|---------------------|
| Unauthorized execution | Execution Gate requires valid Execution Receipt |
| Adversarial AI proposals | AI treated as untrusted input |
| Replay attacks | Monotonic counters and state lineage validation |
| Forged authorization | Cryptographic verification by Verification Kernel |
| Fallback bypass | Fallback treated as separate authorized execution path |
| Execution order manipulation | Append-only ledger with ordered commits |
| State fork attempts | Ledger consistency checks |

Mitigations are enforced structurally, not by convention.

---

## 7. Failure Handling

### Fail-Closed Enforcement
Any of the following conditions SHALL result in execution prevention:
- missing authorization artifact
- invalid or malformed STT
- failed verification
- receipt mismatch
- undefined state transition

There is no partial or best-effort execution mode.

---

## 8. Residual Risk

Even with correct deployment, the following residual risks remain:

- operator error
- misconfiguration of trust boundaries
- external system vulnerabilities
- performance degradation under load

These risks are outside the protocol’s control scope.

---

## 9. Security Boundaries Summary

| Boundary | Trust Level |
|--------|-------------|
| AI / Proposal Domain | Untrusted |
| Verification Kernel | Trusted |
| Execution Gate | Trusted |
| Ledger | Trusted |

Trust boundaries are enforced structurally and cryptographically.

---

## 10. Non-Claims

SOA-MX10 does not claim to:
- detect malicious intent
- align AI behavior
- prevent harmful content generation
- replace organizational security controls

The protocol enforces **execution eligibility only**.

---

## 11. Specification Status

- Protocol Version: **SOA-MX10 v3.0**
- Threat Model Status: **Final (Specification)**

