# Vision

> **Document Level:** A — Conceptual
>
> **Classification:** Informative
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §1, §12
> - Implementation: n/a (conceptual)
> - Reference: [`README.md`](../../README.md)

---

## Purpose

This document states the goals, scope, and non-goals of the 3CP ecosystem. It is
the entry point for engineers, operators, auditors, implementers, and
researchers who need to understand *why* the ecosystem exists before studying
*how* it works.

## Scope

Applies to the ecosystem as a whole: the **3CP protocol**, its reference
implementation **Gleipnir**, and the coupled zero-knowledge audit framework
**Carcosa**. This document is conceptual (Level A) and non-normative. Normative
protocol behavior is defined in [Section 02](../02-3cp-specification/protocol-overview.md).

## Background

Existing compliance frameworks (ISO 27001, SOC 2, NIST CSF, DORA, CRA, NIS2)
assume that organisations produce evidence honestly and that auditors can
evaluate it. When legal liability and economic interest conflict, there is an
objective incentive to alter, suppress, fragment, or reinterpret that evidence.

The **forensic indistinction problem** is the inability to distinguish, after
the fact, a conscious decision that was never recorded from an event that never
occurred. Traditional logs cannot resolve this because they are mutable by the
party with the strongest incentive to alter them.

## Core Thesis

> Accountability cannot depend on the good faith of the audited entity.

3CP inverts the traditional trust model. Instead of trusting the operator to
record everything, it makes the **integrity of chain-of-custody evidence
verifiable by independent third parties, regardless of the operator's
cooperation**. It extends this with **Mandatory Event Anchoring**
([spec §13](../../spec/3CP.md)): signed, versioned, protocol-native declarations
that define which events MUST be recorded. Compliance becomes independently
verifiable by comparing the chain against the active mandate set — omission
becomes detectable.

## Goals

| Goal | Mechanism |
|------|-----------|
| Third-party verifiability | Any holder of the public chain verifies SMT proofs, signatures, and block hashes without contacting the producing network (spec §1) |
| Non-repudiation | Dilithium3 signatures bind each entry to a submitting identity (spec §3.3) |
| Instant finality | One consensus cycle produces one final block — no forks, no rollbacks (spec §6.1) |
| Detectable omission | Mandates make the *obligation* to record independently verifiable (spec §13) |
| Post-quantum security | Dilithium3, Kyber1024, ECVRF (Ristretto255), BLAKE3-256 throughout (spec §3) |
| No tokens, no mining | Consensus is VRF + M-of-N quorum; there is no economic layer (spec §1) |

## Non-Goals

3CP is **not** a blockchain, token, smart-contract platform, or general-purpose
ledger. It explicitly does **not** provide:

| Non-goal | Rationale |
|----------|-----------|
| Access control | 3CP records *who did what*; deciding *who may do what* is the consuming application's responsibility (spec §12.3) |
| Entry confidentiality | Anchored hashes are public; confidential preimages must be hashed before submission (spec §12.3) |
| Built-in key rotation | Identity rotation requires out-of-band validator consensus (spec §12.3) |
| Enforcement of anchoring | A mandate declares what *should* be anchored; it cannot physically prevent omission. The protocol makes omission *detectable*, not *impossible* (spec §12.5) |

## Ecosystem Roles

| Artifact | Role | Language | Relationship to 3CP |
|----------|------|----------|---------------------|
| **3CP** | The protocol | — | Defines wire formats, consensus rules, verification algorithms |
| **Gleipnir** | Reference implementation | Go | Realises 3CP: accepts submissions, forms consensus, produces blocks |
| **Carcosa** | Coupled ZK audit framework | Rust | *Consumes* the 3CP anchoring layer (via Gleipnir gRPC) to seal STARK proofs on-chain; does **not** implement 3CP |

```
3CP (protocol) ← Gleipnir (implements) ← Carcosa (consumes)
```

## Implementation Status

- The protocol specification is **Draft** and validated by a 33-test
  conformance suite against Gleipnir at the gRPC boundary (spec §10).
- No independent second implementation exists yet; untested edge cases may
  remain (spec §10, non-normative note).
- Gleipnir implements the consensus substrate but **does not yet implement
  Mandates** — see [threat-model.md](threat-model.md) and
  [Section 03](../03-gleipnir/architecture.md) for the full gap analysis.
- Carcosa is a **design-complete PoC skeleton**: only the Winterfell AIR and CLI
  argument parsing are implemented. See [Section 04](../04-carcosa/architecture.md).

## References

- [`spec/3CP.md`](../../spec/3CP.md) — normative protocol specification
- [architecture-overview.md](architecture-overview.md) — how the layers fit together
- [threat-model.md](threat-model.md) — adversary model and guarantees
- [terminology.md](terminology.md) — core vocabulary
