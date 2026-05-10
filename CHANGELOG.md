# Changelog

All notable changes to the SOΛ-MX10 specification are documented in this file.

This project follows [Semantic Versioning](https://semver.org/) at the protocol level:
- **MAJOR** — breaking changes to envelope schema, ledger event format, or the verification sequence
- **MINOR** — additive changes preserving existing contracts
- **PATCH** — clarifications and editorial fixes

## [3.0.0] — 2026
- Zenodo DOI: https://doi.org/10.5281/zenodo.20103197
- First public stable release of the specification.

### Added
- Public specification of the seven-phase verification sequence (SYNC → VERIFY → ENHANCE → PLAN → EXECUTE → ATTEST → AUDIT)
- Guarded Envelope artifact schema (`docs/02-guarded-envelope.md`)
- Fallback Preflight Gate five-step sequence (`docs/03-fallback-preflight.md`)
- Symbolic Trace Ledger event format and chain semantics (`docs/04-trace-integrity.md`)
- Public threat model (`docs/05-threat-model.md`)
- Conformance levels A and B with normative test points (`docs/06-conformance.md`)
- Reference pseudocode and a non-production Python SDK sketch (`code/`)
- Security policy and disclosure process (`SECURITY.md`)

### Changed from internal v2.x
- Scope restricted to the publicly disclosable subset of the v2.0 Fleet-Adaptive Security Layer
- Defense-specific embodiments removed from public scope and reserved for restricted continuations:
  - Sovereign-tier fallback configurations
  - Stealth-audit correlation algorithms
  - Specific tuning parameters for uncertainty thresholds and latency budgets in high-sensitivity deployments
- Wire formats and ledger semantics frozen for v3.x

### Notes
- v3.x is the first interoperable wire-level specification. Internal v1 and v2 are not interoperable with v3.x and are not supported.

---

## How to read future entries

Entries below this section follow this template:

```
## [x.y.z] — YYYY-MM-DD

### Added
- new optional fields, new event types, new conformance test points

### Changed
- changes that preserve backward compatibility

### Deprecated
- features marked for removal in a future major version

### Removed
- features removed in this major version (only in MAJOR releases)

### Fixed
- corrections to specification text that do not alter normative behavior

### Security
- security-relevant changes, with cross-references to advisories
```

Reporters credited in `Security` entries are acknowledged here unless they have requested anonymity.
