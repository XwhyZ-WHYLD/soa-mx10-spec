# 04 — Trace Integrity

The Symbolic Trace Ledger (STL) is the append-only, hash-chained record of every decisive event a SOΛ-MX10-conformant orchestrator emits. Trace integrity is not a logging convenience — it is the protocol's evidence layer. Without an unbroken trace chain, fallback preflight cannot succeed, post-incident audit cannot proceed, and conformance cannot be demonstrated.

## 1. Event format

Every ledger event **MUST** carry:

```json
{
  "event_id":       "<UUIDv7>",
  "seq":            <strictly increasing uint64>,
  "ts":             "<RFC 3339 UTC>",
  "event_type":     "<see §2>",
  "subject":        {
    "role_id":     "<UUIDv4>",
    "fleet_id":    "<UUIDv4>",
    "envelope_id": "<UUIDv4 from header>"
  },
  "payload_digest": "<hex SHA-256 of the canonical event payload>",
  "prev_hash":      "<hex SHA-256 of the previous event's full record>",
  "issuer":         "<DID or key id>",
  "sig":            "<base64url signature over (seq | ts | event_type | subject | payload_digest | prev_hash)>"
}
```

The `prev_hash` field is what makes the ledger a chain. Each event's `prev_hash` **MUST** equal the SHA-256 of the canonical serialization of the immediately preceding record. The genesis event of a chain uses `prev_hash = "0" * 64`.

UUIDv7 (time-ordered) is **RECOMMENDED** for `event_id` to support efficient time-range queries; UUIDv4 is **ACCEPTABLE**.

## 2. Event types

Conformant implementations **MUST** emit at least these event types:

### 2.1 Lifecycle

| Type | When |
|------|------|
| `TURN_OPEN` | At start of a new turn (envelope received) |
| `TURN_CLOSE` | At AUDIT |
| `SYNC_OK` | SYNC completed successfully |
| `CORE_DRIFT` | SYNC detected a core_digest mismatch |
| `CORE_REVERTED` | SYNC reverted to last attested core |

### 2.2 Verification

| Type | When |
|------|------|
| `SCHEMA_ATTESTED` | A schema verified successfully (one event per schema) |
| `SCHEMA_UNATTESTED` | A schema failed attestation |
| `CAPABILITY_DELTA` | Schema requests capabilities beyond previous attestation |
| `CHAIN_INJECTION_GRAMMAR` | CIF grammar stage flag |
| `CHAIN_INJECTION_SEMANTIC` | CIF semantic stage flag |
| `MUTATION_ATTEMPT` | Enhancement segment attempted to override Core |

### 2.3 Execution

| Type | When |
|------|------|
| `TOOL_CALL_AUTHORIZED` | EXECUTE permitted a tool call |
| `EXECUTE_DENIED` | EXECUTE blocked a tool call |
| `OUTPUT_ATTESTED` | ATTEST labeled an output |

### 2.4 Fallback

| Type | When |
|------|------|
| `FALLBACK_TRIGGERED` | FPG entered |
| `CORE_DRIFT_AT_FALLBACK` | FPG step 1 failed |
| `SCHEMA_REVERIFICATION_FAILED` | FPG step 2 failed |
| `TRACE_BREAK` | FPG step 3 failed |
| `FALLBACK_DENIED` | FPG aborted |
| `FALLBACK_PROCEEDED` | FPG passed; secondary invoked |

### 2.5 Operational

| Type | When |
|------|------|
| `UNCERTAINTY_FLAG` | u > θ at VERIFY |
| `BUDGET_EXHAUSTED` | Latency guard fired |
| `MIN_SAFE_ENGAGED` | Minimal Safe Profile activated |
| `STL_ANCHOR` | Periodic anchor commit (see §5) |

Implementations **MAY** define additional event types prefixed with their organization namespace (e.g., `acme.MY_EVENT`).

## 3. The trace.sig running value

In addition to per-event hashes, every envelope carries a running `trace.sig` value that summarizes the entire chain to date. It is computed as:

```
trace.sig_0     = H(genesis_event_record)
trace.sig_n     = H(trace.sig_{n-1} || event_record_n)
```

where `||` is byte concatenation and `H` is the protocol hash function. The current `trace.sig` is the hash of the most recent committed event's full record concatenated with the previous running value. It serves as a constant-size fingerprint of the entire chain — useful for fallback preflight (§3.3 of [`03-fallback-preflight.md`](03-fallback-preflight.md)) and for audit assertions.

## 4. Append-only semantics

The STL **MUST** be append-only. Specifically:

- No event **MAY** be edited or deleted after commit
- Reordering events **MUST** be impossible without invalidating every subsequent `prev_hash`
- The storage layer **SHOULD** enforce write-once semantics at the storage primitive level (e.g., AWS QLDB, append-only log file with HMAC seal, immutable database with cryptographic verification)

If the storage layer does not natively support append-only semantics, the implementation **MUST** layer cryptographic seals — typically periodic HMAC anchors over a sliding window of events — and **MUST** detect tamper.

## 5. Compaction and anchoring

Long-running fleets accumulate ledgers faster than is practical to keep online. Compaction is permitted under these constraints:

- A **compacted segment** is a contiguous range `[seq_a, seq_b]` of events
- Compaction produces a **summary record** containing: the seq range, the start `prev_hash`, the end `payload_digest`, the count of events, and a Merkle root computed over all event records in the range
- The summary record is committed to the chain as a `STL_ANCHOR` event with `event_type` set accordingly
- Compacted events **MUST** remain individually retrievable for at least the policy-declared retention window before archive

Compaction **MUST NOT** alter the `trace.sig` running value continuity. Implementations **MUST** preserve the ability to recompute the running value from anchors plus retained events.

External anchoring is **RECOMMENDED**:
- Periodically commit the chain head's `trace.sig` to a trust anchor outside the orchestrator's control (e.g., a customer-controlled KMS, a notary service, an internal audit cluster)
- This prevents undetectable local tamper

## 6. Verification protocol

To verify a chain:

1. Walk events in `seq` order
2. For each event, verify:
   - `seq` is exactly one greater than the predecessor's
   - `ts` is non-decreasing (strict monotonicity may be relaxed within a clock-skew tolerance declared by policy)
   - `prev_hash` equals `H(canonical_serialize(previous_event_record))`
   - `sig` verifies under a key trusted by the active manifest at the time of `ts`
3. Verify the running `trace.sig` matches the stored value

Verification is what fallback preflight step 3 performs. Verification is also what an external auditor performs against an exported chain. The same algorithm serves both.

## 7. Privacy

Event records **MUST NOT** contain raw user content, retrieved document content, or any payload that would leak the conversation. Records carry **digests** of these payloads, not the payloads themselves. The mapping from digest to content (if retained at all) lives in a separate, access-controlled store with its own retention policy.

This separation:
- Makes the ledger safely shareable for audit
- Permits content deletion (e.g., for GDPR right-to-erasure) without breaking the chain — only the lookup table needs purging
- Prevents the ledger itself from becoming an exfiltration target

## 8. Stealth principle

Trace events are **stealth**. They are not surfaced to the conversational interface. An adversary interacting with the model **MUST NOT** be able to:
- Determine whether their input triggered a `MUTATION_ATTEMPT` event
- Infer the configured uncertainty threshold from response variation
- Probe whether the fleet is in Minimal Safe Profile from output style

Implementation choices that make trace events conversationally observable are non-conformant.

## 9. Cross-fleet portability

The ledger format above is intentionally vendor-neutral. A SOΛ-MX10-conformant ledger from any implementation **MUST** be verifiable by any other conformant verifier given the appropriate public keys. This portability is what makes the spec usable across organizations and makes external audit feasible without proprietary tooling.
