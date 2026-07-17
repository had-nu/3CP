# Threat Model

> **Document Level:** B — Architectural
>
> **Classification:** Informative (derived from Normative spec §12)
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §12
> - Implementation: `gleipnir-ipc/pkg/consensus`, `pkg/identity`, `pkg/validation`
> - Reference: [`spec/notes/mandatory-anchoring.md`](../../spec/notes/mandatory-anchoring.md)

---

## Purpose

State the adversarial model 3CP defends against, the guarantees it provides
under that model, and the guarantees it explicitly does **not** provide. This
document answers the security questions mandated by `DOCUMENTATION_SPEC.md` §24:
what is protected, against whom, under which assumptions, and what happens if
those assumptions fail.

## Scope

The protocol-level threat model (spec §12) plus the reference implementation's
position against each threat. Operational hardening is covered in
[Section 05 — Security](../05-operations/security.md).

## Assets Protected

| Asset | Protection | Mechanism |
|-------|------------|-----------|
| Integrity of anchored evidence | Tamper-evidence | SMT proofs over BLAKE3-256 (spec §5) |
| Attribution of submissions | Non-repudiation | Dilithium3 signatures (spec §3.3) |
| Ordering / existence in time | Chain linking + finality | Block hash chain, instant finality (spec §3.2, §6.1) |
| The obligation to record | Detectable omission | Mandates (spec §13) — *not yet in Gleipnir* |
| Proposer fairness | Grinding resistance | ECVRF leader election (spec §3.5) |

## Adversary Capabilities

The specification (spec §12.1) defines four adversaries:

| ID | Adversary | Capabilities | Cannot do |
|----|-----------|--------------|-----------|
| **A1** | Network adversary | Observe, delay, reorder, drop messages between peers | Forge Dilithium3 signatures or break ECVRF proofs |
| **A2** | Compromised operator | Controls one or more validators, their signing keys; can deviate from the protocol | Control the majority of the quorum (M-of-N protects against this) |
| **A3** | Compromised submitter | Holds a valid signing identity but is not the intended submitter | This is a custody problem, not a protocol failure (spec §11.4) |
| **A4** | External adversary | No key material; can attempt DoS, timing, or replay | Produce valid entries or blocks |

## Guarantees

| Guarantee | Condition | Where enforced |
|-----------|-----------|----------------|
| Non-repudiation | Dilithium3 unforgeable under NIST Level 3 | `pkg/identity/dilithium.go`; block co-signing in `pkg/consensus/engine.go` |
| Integrity | SMT collision-resistant under BLAKE3-256 | `pkg/smt/smt.go` |
| Proposer fairness | ECVRF grinding-resistant (RFC 9381) | `pkg/identity/vrf.go`, `pkg/consensus` VRF selection |
| Finality | No forks, no rollbacks | Ticker-driven single-block-per-cycle in `pkg/consensus/engine.go` |
| Third-party verifiability | Public chain suffices; no server access | `pkg/smt` verify + block hash recomputation (spec §5.4, §3.2) |

## Limitations (spec §12.3)

| Limitation | Mitigation |
|------------|------------|
| No access control | Access control is the consuming application's responsibility |
| No entry encryption | Hash confidential preimages before submission |
| No built-in key rotation | Rotation requires out-of-band validator consensus |
| Custody dependency | Non-repudiation is void if signing keys are co-located with the credentials they should constrain (spec §11.4) |
| λ₁ supervision | λ₁ below `MinLambda1` may indicate fragmentation; implementations SHOULD fail closed |
| Mandate ≠ enforcement | A mandate makes omission *detectable*, not *impossible* (spec §12.5) |

## Enforcement Points

Mandate enforcement operates at two points (spec §12.5):

1. **Submission time** — when an entry references a mandate via `MandateRef`,
   the server validates required fields, active period, and authority match.
2. **Verification time** — a verifier compares the chain against the active
   mandate set to detect missing events. This is the cryptographic novelty:
   any third party can perform the comparison deterministically.

## Implementation Divergence

> **Behavior not yet verified in the reference implementation.**
>
> The following spec-defined defenses are **not present** in Gleipnir as
> reviewed:
>
> | Spec feature | Status in Gleipnir | Impact on threat model |
> |--------------|--------------------|------------------------|
> | Mandates (§13) | Not implemented (no mandate code in `pkg/`) | Detectable-omission guarantee against A2 is **not realised**; omission is currently indistinguishable from "not required" |
> | Per-entry signature verification | `ProvenanceEntry.Signature` accepted but **not verified** server-side | Entry-level non-repudiation (spec §7.2.1) is not enforced by the server; block-level co-signing still applies |
> | Multi-node quorum in the shipped daemon | libp2p gossip **not wired** into `provenanced`; the daemon runs single-node | A2 protection (M-of-N) is exercised in tests, not in the default daemon deployment |
> | `RATE_LIMITED` full enforcement | Rate limiter exists (`pkg/consensus/ratelimit.go`); confirm window semantics against spec §7.4 before relying on it |

These gaps are load-bearing for operators: **do not assume mandate-based
omission detection or multi-node Byzantine tolerance in a default Gleipnir
deployment.** See [Section 03](../03-gleipnir/architecture.md) and
[Section 05 — Security](../05-operations/security.md).

## Residual Attacks

| Attack | Status | Notes |
|--------|--------|-------|
| Denial of service (A4) | Partially mitigated | Rate limiting per submitter (spec §7.4); network-level DoS out of scope |
| Replay (A4) | Mitigated | Timestamp + signed payload in auth (spec §7.2.1); REST enforces a 30s skew window |
| Grinding on proposer (A2) | Mitigated | ECVRF (spec §3.5); note the RFC 9381 hash-to-curve divergence in [vrf](../02-3cp-specification/vrf.md) |
| Silent omission (A2) | **Not mitigated in Gleipnir** | Requires mandates (unimplemented) |

## References

- [`spec/3CP.md`](../../spec/3CP.md) §11, §12, §13 — security & mandates
- [`spec/notes/mandatory-anchoring.md`](../../spec/notes/mandatory-anchoring.md) — mandate design rationale
- [Section 05 — Security](../05-operations/security.md) — operational hardening
- [Section 07 — Error Reference](../07-reference/error-reference.md) — error codes
