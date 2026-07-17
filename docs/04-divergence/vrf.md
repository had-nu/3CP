# Divergence — ECVRF Hash-to-Curve (D8)

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §3.5 (VRF)
> - Reference: RFC 9381 §5.4.1 (`HashToCurveTryAndIncrement`)
> - Implementation: `gleipnir-ipc/pkg/identity/vrf.go`

---

## Spec Requirement

The ECVRF MUST follow RFC 9381. The hash-to-curve step (RFC 9381 §5.4.1,
`HashToCurveTryAndIncrement`) produces a deterministic point `H` from the
message and the public key. A single-point `H` is then used in the proof.

## As-Built

`pkg/identity/vrf.go` implements a **custom** hash-to-curve: it computes two
Elligator2 points `h1`, `h2` from two independent hashes and sums them
(`h1 + h2`) to obtain `H`. This is a two-point Elligator2 construction, not the
RFC 9381 single-point try-and-increment.

## Impact

| Dimension | Consequence |
|-----------|-------------|
| Interop | Proofs are NOT verifiable by an RFC 9381-compliant verifier |
| Soundness | Custom construction is unaudited; security margin unknown |
| Spec conformance | Fails §3.5 conformity |

**Severity: Major.** The VRF is usable internally but non-conformant with the
cited standard.

## Evidence

- `vrf.go`: `H = h1 + h2` (two `HashToCurve` calls with distinct prefixes).
- RFC 9381 §5.4.1: a single `H = arbitrary_string_to_point(Y || ...)` via
  try-and-increment.

## Remediation

Replace the two-point sum with RFC 9381 `HashToCurveTryAndIncrement`. Re-run the
`pkg/identity` VRF tests after the change and add a known-answer RFC 9381 vector.

## References

- [spec §3.5 — VRF](../02-3cp-specification/vrf.md)
- [overview](overview.md) (D8)
