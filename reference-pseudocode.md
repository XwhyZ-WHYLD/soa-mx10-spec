# Reference Pseudocode

This document presents the SOΛ-MX10 verification sequence and supporting algorithms in language-agnostic pseudocode. It is **illustrative**, not normative. The normative contract is in [`docs/01-protocol-specification.md`](../docs/01-protocol-specification.md). Where this pseudocode and the spec disagree, the spec governs.

This material is intentionally limited to the public scope. Defense-specific tuning, stealth-audit correlation algorithms, and threshold-tuning logic are not included — see the threat model document for the public/restricted boundary.

---

## 1. The verification sequence

```
function process_turn(envelope):
    # Phase 1: SYNC
    sync_result = sync(envelope.header, envelope.instruction_vector)
    if sync_result == CORE_DRIFT:
        envelope = revert_to_attested_core(envelope)
        commit_event(CORE_REVERTED, envelope)

    # Phase 2: VERIFY (sub-steps may run in parallel)
    schema_result   = verify_all_schemas(envelope.schema_bundle, role_policy(envelope.header.role_id))
    injection_flags = chain_injection_scan(envelope.instruction_vector)
    uncertainty     = uncertainty_scan(envelope)

    # Phase 3: ENHANCE (after VERIFY completes)
    envelope = strip_segments(envelope, injection_flags)
    envelope = strip_segments(envelope, mutation_attempts(envelope))
    envelope = strip_segments(envelope, out_of_scope(envelope))

    # Phase 4: PLAN
    plan = model.plan(envelope)
    if plan.requires_fallback:
        ok = fallback_preflight(envelope, plan.target_runtime)
        if not ok:
            commit_event(FALLBACK_DENIED, envelope)
            envelope.decision_record.min_safe_active = true

    # Phase 5: EXECUTE
    for tool_call in plan.tool_calls:
        if not is_attested(tool_call.schema, envelope.schema_bundle):
            commit_event(EXECUTE_DENIED, envelope, tool_call)
            continue
        if not args_conform_strict(tool_call.args, tool_call.schema):
            commit_event(EXECUTE_DENIED, envelope, tool_call)
            continue
        if not capabilities_in_scope(tool_call.schema, role_policy):
            commit_event(EXECUTE_DENIED, envelope, tool_call)
            continue
        result = execute_tool(tool_call)
        commit_event(TOOL_CALL_AUTHORIZED, envelope, tool_call)

    # Phase 6: ATTEST
    for output in plan.outputs:
        attest_output(output, envelope)

    # Phase 7: AUDIT
    finalize_decision_record(envelope, uncertainty)
    commit_event(TURN_CLOSE, envelope)

    # Cross-cutting: latency guard
    if budget_exhausted():
        engage_minimal_safe_profile(envelope)
        commit_event(BUDGET_EXHAUSTED, envelope)
        commit_event(MIN_SAFE_ENGAGED, envelope)

    return envelope
```

---

## 2. SYNC

```
function sync(header, instruction_vector):
    core_segments = filter(instruction_vector, segment.layer == "core")
    canonical_core = canonical_serialize(core_segments)
    computed_digest = H(canonical_core)

    if computed_digest != header.core_digest:
        return CORE_DRIFT

    last_attested = lookup_attested_core(header.role_id, header.fleet_id)
    if computed_digest != last_attested:
        # Allow only if this turn carries a signed policy update bumping the policy
        if not has_signed_policy_update(header):
            return CORE_DRIFT

    commit_event(SYNC_OK, header)
    return OK
```

---

## 3. Schema attestation

```
function verify_all_schemas(bundle, role_policy):
    results = []
    for entry in bundle:
        normalized = canonicalize_schema(entry.schema)
        recomputed_digest = H(normalized)
        if recomputed_digest != entry.schema_digest:
            commit_event(SCHEMA_UNATTESTED, entry, "digest_mismatch")
            results.append((entry, FAIL))
            continue

        manifest_entry = lookup_manifest(entry.manifest_ref)
        if manifest_entry == NOT_FOUND or manifest_entry.revoked:
            commit_event(SCHEMA_UNATTESTED, entry, "manifest_miss_or_revoked")
            results.append((entry, FAIL))
            continue

        if not verify_signature(entry.manifest_sig, manifest_entry.public_key):
            commit_event(SCHEMA_UNATTESTED, entry, "sig_invalid")
            results.append((entry, FAIL))
            continue

        if not subset(entry.capability_tags, role_policy.allowed_scopes):
            commit_event(SCHEMA_UNATTESTED, entry, "scope_violation")
            results.append((entry, FAIL))
            continue

        # Capability delta check against last attested for this schema name
        previous_caps = lookup_previous_capabilities(entry.name)
        if previous_caps and not subset(entry.capability_tags, previous_caps):
            commit_event(CAPABILITY_DELTA, entry)
            # Capability deltas require fresh attestation; do not auto-pass
            results.append((entry, REQUIRES_REATTEST))
            continue

        commit_event(SCHEMA_ATTESTED, entry)
        results.append((entry, OK))
    return results
```

---

## 4. Chain-injection detection

```
function chain_injection_scan(instruction_vector):
    flags = []
    for segment in instruction_vector:
        if segment.layer == "core":
            continue   # core is immutable; nothing to flag
        # Stage 1: grammar
        if matches_grammar(segment.body, G_inj):
            commit_event(CHAIN_INJECTION_GRAMMAR, segment)
            flags.append(segment.segment_id)
            continue
        # Stage 2: semantic
        if has_intent_to_alter_precedence(segment.body, active_precedence):
            commit_event(CHAIN_INJECTION_SEMANTIC, segment)
            flags.append(segment.segment_id)
            continue
    return flags

# G_inj — abridged. Production grammars are richer and version-managed.
G_inj = {
    "imperative_override":   re.compile(r"\b(ignore|override|disregard|replace)\s+(prior|previous|the|all|your)\b", IGNORECASE),
    "wrapper_introduction":  re.compile(r"\b(you\s+are\s+now|from\s+now\s+on|act\s+as|pretend\s+to\s+be|load\s+new)\b", IGNORECASE),
    "defense_disable":       re.compile(r"\b(turn\s+off|deactivate|disable|bypass|suspend)\s+(safety|monitor|security|guard)\b", IGNORECASE),
    "control_flow_redirect": re.compile(r"\b(jailbreak|prompt\s+injection|developer\s+mode)\b", IGNORECASE),
}

function has_intent_to_alter_precedence(text, precedence):
    # Implementation-defined. A semantic validator that interprets
    # the segment in the context of the active precedence relation.
    # Implementations MAY use small classifiers, rule-based parsers, or
    # constrained LLM calls. The output is a boolean: does this segment
    # express intent to elevate or replace any layer above its own?
    ...
```

The grammar above is illustrative. Production grammars must:
- Cover unicode normalization attacks (homoglyphs, zero-width characters)
- Be regularly updated against red-team output
- Be version-pinned per policy version

---

## 5. Fallback preflight

```
function fallback_preflight(envelope, target_runtime):
    # Step 1: Rehydrate Core
    core = reload_core(envelope.header.role_id, envelope.header.fleet_id)
    canonical = canonical_serialize(core)
    if H(canonical) != envelope.header.core_digest:
        commit_event(CORE_DRIFT_AT_FALLBACK, envelope)
        return false
    rehydrated = adapt_core_for_runtime(core, target_runtime)

    # Step 2: Re-pull and verify schemas
    for entry in envelope.schema_bundle:
        fresh = fetch_manifest_entry(entry.manifest_ref)
        if fresh == NOT_FOUND or fresh.revoked:
            commit_event(SCHEMA_REVERIFICATION_FAILED, envelope, entry)
            return false
        if not verify_signature(fresh.sig, fresh.public_key):
            commit_event(SCHEMA_REVERIFICATION_FAILED, envelope, entry)
            return false
        if H(canonicalize_schema(fresh.schema)) != entry.schema_digest:
            commit_event(SCHEMA_REVERIFICATION_FAILED, envelope, entry)
            return false
        translate_schema_for_runtime(fresh.schema, target_runtime)

    # Step 3: Validate trace.sig continuity
    chain = load_trace_chain_for_envelope(envelope)
    if not verify_chain(chain):
        commit_event(TRACE_BREAK, envelope)
        return false

    # Step 4: All passed; construct fallback envelope
    fallback_env = construct_envelope(
        role_id        = envelope.header.role_id,
        fleet_id       = envelope.header.fleet_id,
        core_digest    = envelope.header.core_digest,
        policy_version = envelope.header.policy_version,
        nonce          = fresh_nonce(),
        ts             = now_utc(),
        predecessor_trace_sig = chain.tail.running_trace_sig
    )
    sign_envelope(fallback_env, orchestrator_key)
    commit_event(FALLBACK_PROCEEDED, envelope)

    # Step 5: Hand off (the new envelope flows through SYNC → AUDIT
    # against the secondary runtime)
    return invoke_secondary(target_runtime, fallback_env)
```

---

## 6. Trace ledger commit

```
function commit_event(event_type, subject, *extra):
    prev = ledger.tail()
    seq  = prev.seq + 1 if prev else 0
    payload = canonical_serialize({
        "event_type": event_type,
        "subject": subject,
        "extra": extra,
    })
    payload_digest = H(payload)
    prev_hash = H(canonical_serialize(prev)) if prev else "0" * 64

    record = {
        "event_id":       new_uuid_v7(),
        "seq":            seq,
        "ts":             now_utc(),
        "event_type":     event_type,
        "subject":        subject,
        "payload_digest": payload_digest,
        "prev_hash":      prev_hash,
        "issuer":         orchestrator_identity,
    }
    record["sig"] = sign(canonical_serialize(record), orchestrator_key)
    ledger.append(record)

    # Update running trace.sig
    running = ledger.running_trace_sig()
    new_running = H(running || canonical_serialize(record))
    ledger.set_running_trace_sig(new_running)

    return record
```

---

## 7. Latency guard

```
function with_latency_guard(t_budget, work_fn, envelope):
    deadline = now_monotonic() + t_budget
    try:
        result = work_fn(deadline)
        if now_monotonic() > deadline:
            engage_minimal_safe_profile(envelope)
            commit_event(BUDGET_EXHAUSTED, envelope)
            commit_event(MIN_SAFE_ENGAGED, envelope)
        return result
    except DeadlineExceeded:
        engage_minimal_safe_profile(envelope)
        commit_event(BUDGET_EXHAUSTED, envelope)
        commit_event(MIN_SAFE_ENGAGED, envelope)
        return MIN_SAFE_RESPONSE

function engage_minimal_safe_profile(envelope):
    envelope.schema_bundle = []                   # all tools denied
    envelope.instruction_vector = filter(
        envelope.instruction_vector,
        seg.layer == "core"
    )                                             # core only
    envelope.decision_record.min_safe_active = true
    # Fallback is not permitted while in MSP — caller must not invoke FPG
```

---

## 8. Canonical serialization

```
function canonical_serialize(obj):
    # JCS (RFC 8785) is RECOMMENDED.
    # Minimum requirements:
    #   - JSON keys sorted lexicographically at every depth
    #   - No insignificant whitespace
    #   - UTF-8 NFC encoding
    #   - Numbers in shortest round-trippable decimal
    #   - No trailing newline
    return jcs.canonicalize(obj)
```

---

## What this pseudocode does not cover

- Stealth-audit correlation algorithms across events
- Adaptive uncertainty threshold tuning
- Sovereign and defense-tier fallback configuration
- Specific manifest-distribution protocols (signed-API vs. ledger vs. KMS-attested cache)
- Performance optimizations (manifest caching, grammar DFA compilation, parallel attestation)

These are either deployer choices or restricted material. See the threat model and conformance documents for the public/restricted boundary.
