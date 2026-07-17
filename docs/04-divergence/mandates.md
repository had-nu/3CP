# Divergence — Mandates (D2)

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §8, `spec/notes/mandatory-anchoring.md`
> - Implementation: `gleipnir-ipc/` — **no mandate package or RPC**

---

## Spec Requirement

Mandates (spec §8, §13.5) are signed compliance directives that bind an
anchoring quorum to produce a specific proof artifact on a schedule. They
comprise `Mandate` (target chain, rule-set, cadence, threshold, TTL),
`MandateProof` (the artifact), `MandateComplianceGap` (missing / missing_fields
events), and the `SubmitMandate` / `GetMandate` / `GetActiveMandates` RPCs.

## As-Built

Gleipnir contains **no mandate code**:

- No `pkg/mandate` package.
- No mandate messages in `api.proto`.
- The engine never loads, activates, or enforces mandates.
- Carcosa's audit logic (planned) is the intended consumer of mandate proofs.

## Impact

| Affected area | Consequence |
|---------------|-------------|
| Compliance attestation | Not available — no mandate proofs produced |
| Audit trail | Carcosa has nothing to verify against mandates |
| `MandateComplianceGap` | Cannot be emitted |

This is a **Critical** gap: a core normative feature is entirely absent.

## Remediation

1. Implement `pkg/mandate` (parse, activate, TTL, threshold).
2. Add `SubmitMandate`/`GetMandate`/`GetActiveMandates` to `api.proto`.
3. Wire mandate enforcement into `RunCycle` (produce `MandateProof`).
4. Define `MandateComplianceGap` emission on schedule miss.

Until then, operators MUST NOT represent Gleipnir as mandate-capable.

## References

- [spec §8 — Mandates](../02-3cp-specification/anchoring.md)
- [overview](overview.md) (D2)
- [carcosa gap](carcosa.md)
