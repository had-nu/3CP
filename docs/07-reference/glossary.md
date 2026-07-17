# Glossary

> **Document Level:** B — Reference
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/glossary.md`](../../spec/glossary.md)
> - See also: [Section 01 — Terminology](../01-introduction/terminology.md)

---

## Purpose

Authoritative glossary of 3CP terms. Mirrors `spec/glossary.md` and the
implementation-specific terms used in these docs.

## Terms

| Term | Definition |
|------|------------|
| **3CP** | Three-Chain Protocol — provenance anchoring system |
| **Gleipnir** | Go reference implementation (consensus daemon) |
| **Carcosa** | Rust ZK-audit component (Winterfell STARKs) |
| **Triad** | The three-chain block (proposal, validation, supervision) |
| **SMT** | Sparse Merkle Tree — membership/non-membership proofs |
| **λ₁ (Lambda1)** | Fiedler eigenvalue; network diffusion health |
| **Supervision Root** | BLAKE3 of partial state; chains state transitions |
| **Mandate** | Signed compliance directive binding a quorum (spec §8) |
| **MandateProof** | Artifact a quorum must produce per a mandate |
| **Compliance gap** | Missing (`missing`) or incomplete (`missing_fields`) mandated event |
| **ECVRF** | Elliptic Curve Verifiable Random Function (RFC 9381) |
| **UID0** | Root identity; Dilithium3 key pair (CBOR) |
| **AIR** | Algebraic Intermediate Representation (Winterfell constraint system) |
| **STARK** | Scalable Transparent ARgument of Knowledge |
| **Blob store** | Off-chain storage for STARK proof binaries; 32-byte digest anchored |
| **AnchorProof** | SMT membership proof + block index returned by `WaitForAnchor` |
| **Error codes** | Registry uses `ZERO_HASH`, `EMPTY_SUBMITTER` (spec §7.3) — Gleipnir uses `INVALID_HASH`, `INVALID_SUBMITTER` (see [error-reference](error-reference.md)) |

## Divergence Notes

- Error-code names differ between spec and Gleipnir — see
  [error-reference](error-reference.md) and [divergence D9](../04-divergence/errors.md).
- Mandates/Carcosa are not yet implemented — see
  [divergence overview](../04-divergence/overview.md).

## References

- [Section 01 — Terminology](../01-introduction/terminology.md)
- [`spec/glossary.md`](../../spec/glossary.md)
