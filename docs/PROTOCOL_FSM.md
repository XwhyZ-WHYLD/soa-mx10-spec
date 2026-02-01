# SOA-MX10 v3.0  
## Formal Execution State Machine Specification (FSM)

---

## 1. Purpose

This document defines the **formal finite state machine (FSM)** governing execution eligibility within the SOA-MX10 v3.0 protocol.

The FSM specifies:
- valid system states
- permitted state transitions
- execution authorization boundaries
- mandatory halt conditions

The FSM is **enforced by the Execution Gate (EG)** and **Verification Kernel (VK)** and is independent of AI behavior.

---

## 2. Design Principles

The FSM is designed to satisfy the following principles:

1. **No State Skipping**  
   All transitions must follow explicitly permitted paths.

2. **Fail-Closed Semantics**  
   Any invalid or ambiguous transition results in a terminal halt state.

3. **Authorization-Bound Execution**  
   Execution is permitted only in explicitly authorized states.

4. **Deterministic Enforcement**  
   State transitions are deterministic and non-AI.

---

## 3. State Definitions

| State ID | State Name | Description |
|--------|-----------|-------------|
| S0 | INIT | Session initialization; no execution permitted |
| S1 | CONTEXT_LOCKED | Policy, role, and session context locked |
| S2 | PROPOSAL_RECEIVED | Operation proposal received from untrusted source |
| S3 | STT_SUBMITTED | State Transition Token submitted for verification |
| S4 | STT_VERIFIED | STT verified successfully by Verification Kernel |
| S5 | EXECUTION_AUTHORIZED | Execution Receipt issued |
| S6 | OPERATION_EXECUTED | Operation executed through Execution Gate |
| S7 | COMMIT | Execution committed to append-only ledger |
| SH | HALT | Terminal halt state (no execution permitted) |

---

## 4. Permitted State Transitions

| From State | To State | Condition |
|-----------|---------|-----------|
| S0 | S1 | Context successfully initialized and locked |
| S1 | S2 | Operation proposal received |
| S2 | S3 | STT constructed and submitted |
| S3 | S4 | STT verified successfully |
| S3 | SH | STT invalid or verification failed |
| S4 | S5 | Execution Receipt issued |
| S5 | S6 | Execution Gate validates receipt |
| S6 | S7 | Execution committed to ledger |
| Any | SH | Invalid transition or missing authorization |

---

## 5. Execution Authorization Rule

Execution of any operation is permitted **only** when the system is in state **S5 (EXECUTION_AUTHORIZED)** and a valid **Execution Receipt** has been issued by the Verification Kernel.

Any attempt to execute an operation outside this state SHALL result in a transition to **SH (HALT)**.

---

## 6. Halt State (SH)

### Definition
The HALT state is a terminal state in which:
- no further execution is permitted
- no state transitions are allowed
- recovery requires explicit external reinitialization

### Conditions Triggering HALT
- missing State Transition Token
- invalid or malformed STT
- signature verification failure
- monotonic counter violation
- state hash mismatch
- missing or invalid Execution Receipt
- unauthorized execution attempt
- undefined state transition

---

## 7. Fallback Execution Path Handling

Fallback execution is modeled as a **separate execution proposal**, not a continuation.

Fallback handling requirements:
- requires a new STT
- bound to the same policy hash and session lineage
- subject to the same FSM transitions
- cannot bypass authorization states

Fallback attempts that do not satisfy these conditions SHALL transition to **SH (HALT)**.

---

## 8. FSM Enforcement Responsibility

| Component | Responsibility |
|---------|----------------|
| Execution Gate (EG) | Enforces execution state and blocks unauthorized execution |
| Verification Kernel (VK) | Validates STTs and issues Execution Receipts |
| Append-Only Ledger (AL) | Records committed execution events |
| AI Components | Proposal generation only (no state authority) |

---

## 9. FSM Determinism Guarantee

Given identical:
- initial state
- authorization artifacts
- verification results

The FSM SHALL transition deterministically.

AI output variability does not influence FSM behavior.

---

## 10. Specification Notes

- The FSM defines **authorization logic**, not implementation detail.
- Timing, concurrency, and scheduling concerns are outside the scope of this document.
- The FSM is compatible with both single-node and distributed deployments.

---

## 11. Versioning

- Protocol Version: **SOA-MX10 v3.0**
- FSM Status: **Final (Specification)**
