# 02 — The Guarded Envelope

The **Guarded Envelope** is the canonical artifact passed between SOΛ-MX10 phases. It is the unit of authorization. Every operation an orchestrator performs is bound to exactly one envelope.

## 1. Top-level structure

```json
{
  "header":             { ... },
  "instruction_vector": [ ... ],
  "schema_bundle":      [ ... ],
  "context_bundle":     { ... },
  "decision_record":    { ... }
}
```

Implementations **MUST** support the JSON encoding above for interoperability. CBOR encoding using the same field structure **MAY** be used internally for performance.

## 2. Header

```json
{
  "header": {
    "envelope_version": "3.0",
    "role_id":          "<UUIDv4>",
    "fleet_id":         "<UUIDv4>",
    "core_digest":      "<hex SHA-256>",
    "policy_version":   "<semver>",
    "nonce":            "<128-bit base64url>",
    "ts":               "<RFC 3339 UTC>",
    "issuer":           "<DID or key id>"
  }
}
```

Field semantics:

- `envelope_version` — **MUST** be `3.0` for this version of the spec
- `role_id` — Immutable identifier for the role node in the Fleet Policy Graph
- `fleet_id` — Identifier for the orchestration fleet this envelope belongs to
- `core_digest` — Hash of the canonical core instructions; recomputed at SYNC and MUST match the SYNC-attested value
- `policy_version` — Semver of the active policy manifest
- `nonce` — Per-envelope nonce; MUST NOT be reused within a single trace chain
- `ts` — Envelope construction timestamp; clock skew tolerance is policy-declared
- `issuer` — Identifier of the entity that constructed and signed this envelope

The header **MUST** be signed; the signature **MUST** cover the entire envelope, not the header alone (see §6).

## 3. Instruction Vector

An ordered array of instruction segments. Order is significant: lower indices have higher precedence.

```json
{
  "instruction_vector": [
    {
      "segment_id":   "<UUIDv4>",
      "layer":        "core" | "developer" | "enhancement" | "runtime_hint",
      "issuer":       "<DID or key id>",
      "scope":        ["<scope-tag>", ...],
      "sig":          "<base64url signature>",
      "body":         "<segment payload>"
    }
  ]
}
```

### 3.1 Layer precedence

The protocol enforces a total precedence order:

```
core  ⊐  developer  ⊐  enhancement  ⊐  runtime_hint
```

A segment of layer L **MUST NOT** modify any clause issued by a segment of layer L' where L' ⊐ L. Attempts to do so are flagged as `MUTATION_ATTEMPT` and the violating segment is stripped during ENHANCE.

### 3.2 Scope

Each segment declares an array of scope tags. The role's policy declares `allowed_scopes(role_id)`. A segment is acceptable if and only if `scope(segment) ⊆ allowed_scopes(role_id)`. Out-of-scope segments are stripped.

### 3.3 Signature

Every non-runtime_hint segment **MUST** carry a valid signature from a key trusted by the role's policy. Runtime hints **MAY** be unsigned (they cannot influence anything above their precedence anyway).

## 4. Schema Bundle

An array of tool/function schemas available to the model in this turn.

```json
{
  "schema_bundle": [
    {
      "name":          "<tool name>",
      "schema":        { /* JSON Schema, draft 2020-12 */ },
      "schema_digest": "<hex SHA-256 of canonicalized schema>",
      "capability_tags": ["<capability>", ...],
      "manifest_ref":  "<manifest entry id>",
      "manifest_sig":  "<base64url signature over manifest_ref|schema_digest>"
    }
  ]
}
```

A schema is **attested** when:
- `schema_digest` matches the canonical digest of `schema` (recomputed at VERIFY)
- The pair `(manifest_ref, schema_digest)` matches a non-revoked entry in the active manifest
- `manifest_sig` verifies under a trusted key
- `capability_tags ⊆ allowed_scopes(role_id)`

Unattested schemas **MUST NOT** be invoked.

## 5. Context Bundle

```json
{
  "context_bundle": {
    "user_content":       "<string or structured>",
    "retrieved_documents": [
      {
        "doc_id":       "<id>",
        "provenance":   "<source URI / signature>",
        "fetched_ts":   "<RFC 3339 UTC>",
        "content_hash": "<hex SHA-256>"
      }
    ],
    "prior_turn_fingerprints": [
      "<hex SHA-256>", ...
    ]
  }
}
```

Retrieved documents **MUST** carry a provenance attestation. Documents lacking provenance are treated as `untrusted_input` and **MUST NOT** influence Core or Schema interpretation — only conversational content.

`prior_turn_fingerprints` are the trace.sig values of preceding turns in this conversation, enabling conversation-level continuity verification.

## 6. Envelope signature

The envelope **MUST** carry a single outer signature covering the canonical serialization of all fields above. The signature is what an inline guardian verifies first; everything else is parsed only after that signature succeeds.

```json
{
  "envelope_sig": {
    "alg":     "Ed25519" | "HMAC-SHA256" | "<other>",
    "key_id":  "<key id>",
    "value":   "<base64url signature>"
  }
}
```

The signature scope **MUST** include header, instruction_vector, schema_bundle, and context_bundle. The decision_record is mutated during the verification sequence and is not included in the inbound signature.

## 7. Decision Record

The decision_record is **populated by the orchestrator during processing** — it is not part of the inbound envelope. After AUDIT it carries:

```json
{
  "decision_record": {
    "phase_outcomes": {
      "sync":    "ok" | "core_drift" | "reverted",
      "verify":  "ok" | "schema_unattested" | "chain_injection" | "uncertainty_flag",
      "enhance": "ok" | "mutation_attempt_stripped",
      "plan":    "ok" | "fallback_invoked",
      "execute": "ok" | "denied",
      "attest":  "ok",
      "audit":   "ok"
    },
    "violations":      [ ... ],
    "drops":           [ ... ],
    "trace_sig_final": "<hex SHA-256>",
    "uncertainty":     0.0,
    "latency_ms":      0,
    "min_safe_active": false
  }
}
```

The final decision_record **MUST** be committed to the Symbolic Trace Ledger as part of AUDIT.

## 8. Canonical serialization

For digest computation and signature verification, envelopes **MUST** be serialized canonically:

- JSON keys sorted lexicographically at every depth
- No whitespace between tokens
- UTF-8 NFC encoding
- Numbers in shortest round-trippable decimal representation
- No trailing newline

[RFC 8785 (JCS)](https://www.rfc-editor.org/rfc/rfc8785) is **RECOMMENDED**. Implementations **MUST** declare their canonicalization choice in the Conformance Statement.

## 9. Versioning and evolution

Future versions of the envelope **MUST** preserve backward-compatible parsing:
- New optional fields **MAY** be added in minor versions
- Existing field semantics **MUST NOT** change in minor versions
- Removing or renaming fields requires a major version bump and is non-trivially breaking

Implementations encountering an `envelope_version` newer than they support **MUST** reject the envelope with `UNSUPPORTED_VERSION` rather than attempt partial parsing.
