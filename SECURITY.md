# Security Policy

## Scope

This security policy covers the **SOΛ-MX10 v3.0 specification itself**. It applies to:

- Authorization bypasses, replay vulnerabilities, or state-rollback risks in the protocol as specified
- Ambiguities or omissions in the specification that could lead to insecure but conformant implementations
- Cryptographic primitive choices that are or become inadequate
- Defects in the *Conformance* test points

This policy does **not** cover:

- Defects in third-party implementations of the specification (report those to the implementation maintainer)
- Misuse of conforming implementations by deployers
- Issues unrelated to the SOΛ-MX10 protocol layer

For a fuller treatment of in-scope versus out-of-scope concerns, see [`docs/05-threat-model.md`](docs/05-threat-model.md).

## Severity classification

We classify defects against this specification as Critical, High, Medium, or Low. The criteria are defined in [`docs/05-threat-model.md` §4](docs/05-threat-model.md#4-severity-classification-for-disclosed-defects).

## Reporting

To report a security defect against the SOΛ-MX10 specification:

1. **Do not** open a public GitHub issue for security-relevant defects.
2. Use GitHub's [private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability) on this repository.
3. Provide:
   - A clear description of the defect
   - The specific section(s) of the specification it affects
   - A worked example or proof-of-concept showing how the defect could be exploited
   - Any suggested mitigation or specification clarification

If GitHub private vulnerability reporting is unavailable to you, a public placeholder issue requesting a private channel is acceptable. Do not include exploit details in the placeholder.

## Acknowledgment

We acknowledge receipt within **72 hours** of submission via the same channel used to report. We do not commit to a fix timeline at acknowledgment; that depends on severity and the nature of the change required.

## Coordinated disclosure

For Critical and High severity defects, we follow coordinated disclosure:

1. We work with the reporter to confirm and characterize the defect.
2. We prepare a specification update that addresses the defect.
3. We notify known conforming implementers privately, with a window to update before public disclosure.
4. We publish the specification update and a public advisory naming the reporter (with their consent).

The notification window is typically 30–90 days depending on severity and complexity. Reporters who prefer immediate public disclosure should say so at submission; we will respect that preference.

For Medium and Low severity defects, we may handle them as ordinary specification updates without coordinated notification.

## Recognition

Reporters are credited in the relevant `CHANGELOG.md` entry and in the published advisory unless they request anonymity.

We do not currently offer monetary rewards. Specification-level defects are typically of academic and operational interest rather than direct exploitability of a specific deployment.

## Out of scope for this policy

If you have discovered a vulnerability in:

- An **implementation** of SOΛ-MX10 — report to that implementation's maintainer
- A **deployment** that claims SOΛ-MX10 conformance — report to that deployment's operator
- A different **XWHYZ Research / WHYLD** project — see that project's `SECURITY.md`

We are happy to forward reports to the appropriate maintainer if you are unsure where to send a disclosure.

## Contact

Primary channel: GitHub private vulnerability reporting on this repository.

Backup channel: Listed in the `Maintainers` section of [`README.md`](README.md). For Critical defects where GitHub reporting is unavailable, use any contact route published by XWHYZ Research / WHYLD.
