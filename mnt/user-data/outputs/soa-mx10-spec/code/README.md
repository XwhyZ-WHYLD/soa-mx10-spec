# Code

Reference material for implementers. **Not production code.**

## Files

| File | Purpose |
|------|---------|
| [`reference-pseudocode.md`](reference-pseudocode.md) | Language-agnostic pseudocode for the verification sequence and supporting algorithms |
| [`python_sdk_sketch.py`](python_sdk_sketch.py) | Runnable, illustrative Python sketch showing how an orchestrator wraps the protocol |

## What this is for

These artifacts exist to make the specification concrete. Reading the spec alone leaves room for plausible-but-wrong implementations. Reading code with the spec lets you check your understanding against working logic.

## What this is not

- **Not production-ready.** Cryptography is placeholder. Storage is in-memory. Manifest distribution is hand-waved. Key rotation, HSM integration, observability, and many other production concerns are absent.
- **Not normative.** Where the code and the spec disagree, the spec governs. The code may have bugs the spec does not.
- **Not the trade-secret subset.** This material is intentionally limited to the publicly disclosable scope. Stealth-audit correlation, defense-tier fallback configuration, and threshold-tuning logic are not here.

## Running the Python sketch

```bash
python3 python_sdk_sketch.py
```

The smoke test at the bottom of the file constructs an envelope containing:
- A legitimate Core instruction
- A malicious enhancement segment ("Ignore all previous instructions...")
- One attested tool schema (`get_weather`)

Expected output:
- Chain verifies (`True`)
- The malicious segment is dropped at ENHANCE
- The weather tool call is authorized at EXECUTE
- All seven phases report `ok`

If you change the malicious segment to something the grammar doesn't catch, you'll see it pass through — that's the limit of the grammar-only sketch and the reason the spec mandates a semantic stage that this sketch stubs out.

## Building a conformant implementation

Read [`../docs/06-conformance.md`](../docs/06-conformance.md) for the test points. Pass all of them. Publish a Conformance Statement. That's the contract.

The pseudocode and sketch here are scaffolding to help you get there — they do not themselves conform.
