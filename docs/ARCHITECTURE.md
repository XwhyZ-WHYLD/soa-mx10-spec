# SOA-MX10 v3.0  
## System Architecture and Trust Boundaries

---

## 1. Purpose

This document defines the **system architecture** and **trust boundaries** of the SOA-MX10 v3.0 execution-control protocol.

The architecture is designed to:
- separate proposal, verification, and execution roles
- prevent unauthorized execution paths
- enforce deterministic control independent of AI behavior

This document specifies **structural boundaries**, not implementation details.

---

## 2. Architectural Overview

SOA-MX10 is structured as a **control-plane overlay** on AI-orchestrated systems.

The system is divided into three primary domains:

1. **Untrusted Proposal Domain**
2. **Trusted Verification Domain**
3. **Controlled Execution Domain**

No single component may span more than one domain.

---

## 3. Untrusted Proposal Domain

### Components
- AI models (LLMs or other AI systems)
- user inputs
- external data sources
- orchestration logic that generates operation proposals

### Characteristics
- outputs are treated as **untrusted**
- may be adversarial, malformed, or deceptive
- have **no execution authority**

### Permitted Actions
- propose operations
- construct State Transition Tokens (STTs)
- submit authorization requests

### Explicit Prohibitions
- direct tool invocation
- direct external calls
- execution of fallback paths
- modification of execution state

The Untrusted Proposal Domain cannot cause execution directly.

---

## 4. Trusted Verification Domain

### Components
- Verification Kernel (VK)
- cryptographic verification mechanisms
- state transition validation logic

### Role
The Trusted Verification Domain determines whether a proposed operation is **eligible** for execution.

### Responsibilities
- validate State Transition Tokens
- verify cryptographic signatures
- enforce monotonic counter progression
- validate state lineage and transition legality
- issue Execution Receipts upon successful verification

### Constraints
- deterministic behavior only
- no AI inference
- no policy interpretation beyond explicit rules
- no execution capability

This domain authorizes execution but cannot execute operations itself.

---

## 5. Controlled Execution Domain

### Components
- Execution Gate (EG)
- tool invocation interfaces
- external system connectors
- fallback execution mechanisms

### Role
The Controlled Execution Domain performs operations **only** when explicitly authorized.

### Responsibilities
- intercept all execution attempts
- require a valid Execution Receipt
- block execution absent authorization
- enforce fail-closed behavior

### Properties
- non-bypassable
- isolated from proposal generation
- isolated from verification logic

Execution without authorization is **technically impossible** within this domain.

---

## 6. Append-Only Ledger

### Placement
The Append-Only Ledger operates alongside the Trusted Verification Domain.

### Function
- records verified execution events
- provides tamper-evident auditability
- supports detection of replay or fork attempts

### Properties
- write-only
- ordered
- immutable once appended

The ledger does not participate in authorization decisions.

---

## 7. Data Flow Summary

1. An untrusted component proposes an operation.
2. A State Transition Token is submitted to the Verification Kernel.
3. The Verification Kernel validates the token.
4. If valid, an Execution Receipt is issued.
5. The Execution Gate permits execution only upon receipt validation.
6. The execution event is committed to the Append-Only Ledger.

At no point does the AI component cross into execution authority.

---

## 8. Trust Boundary Enforcement

| Boundary | Enforcement Mechanism |
|--------|----------------------|
| Proposal → Verification | Cryptographic token validation |
| Verification → Execution | Execution Receipt requirement |
| Execution → Proposal | No reverse data path |
| Fallback → Execution | Same authorization requirements |

Trust boundaries are enforced structurally, not by convention.

---

## 9. Fallback Architecture

Fallback execution is modeled as a **distinct execution path** within the Controlled Execution Domain.

### Requirements
- fallback requires a new State Transition Token
- fallback is subject to the same verification process
- fallback cannot reuse prior authorization artifacts
- fallback cannot bypass the Execution Gate

Fallback is not a retry mechanism; it is a separate authorized execution.

---

## 10. Failure Containment

Failure in any domain is contained to that domain.

Examples:
- invalid STT → verification failure, no execution
- execution attempt without receipt → blocked at Execution Gate
- ledger failure → execution halted

No domain failure propagates into unauthorized execution.

---

## 11. Non-Goals

The architecture explicitly does not attempt to:
- ensure AI alignment
- prevent malicious intent generation
- guarantee correctness of AI reasoning
- control AI internal behavior

The architecture controls **execution eligibility only**.

---

## 12. Architectural Guarantees

Given correct deployment of SOA-MX10:

- unauthorized operations cannot execute
- AI behavior variability does not affect enforcement
- fallback cannot bypass authorization
- execution paths are deterministic and auditable

---

## 13. Specification Status

- Protocol Version: **SOA-MX10 v3.0**
- Architecture Status: **Final (Specification)**

