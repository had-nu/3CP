# Terminology

> **Document Level:** A — Conceptual
>
> **Classification:** Informative
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/glossary.md`](../../spec/glossary.md), [`spec/3CP.md`](../../spec/3CP.md)
> - Implementation: `gleipnir-ipc/pkg/`

---

## Purpose

Define the core vocabulary of the 3CP ecosystem. This is the introductory,
navigable subset. The authoritative, exhaustive glossary is
[Section 07 — Glossary](../07-reference/glossary.md), which mirrors
[`spec/glossary.md`](../../spec/glossary.md).

## Scope

Terms used across protocol, implementation, and framework documents. Where a
term has an implementation-specific meaning that differs from the specification,
the divergence is noted here and detailed in the referenced document.

## Core Terms

| Term | Definition | Source |
|------|------------|--------|
| **Anchor** | The act of recording a hash into the 3CP chain, making it part of an immutable block | spec glossary |
| **AnchorProof** | SMT proof + block index + block timestamp for an entry; produced by `WaitForAnchor` / `VerifyHash` | spec §9, `pkg/chain` |
| **Block** | Atomic unit of the chain; one block per consensus cycle; contains entries, prev-hash, state root, signatures, metadata | spec §4.1 |
| **BlockHash** | SHA-256 identifier of a block over a canonical preimage | spec §3.2 |
| **Canonical CBOR** | Deterministic CBOR: ascending integer keys, smallest representation, definite-length | spec §3.6 |
| **Chain-of-custody** | Chronological documentation of custody, control, transfer, analysis, and disposition of digital evidence | spec glossary |
| **Compliance gap** | An event that should exist per an active mandate but does not (`missing`), or exists without required fields (`missing_fields`) | spec §13.5 |
| **Consensus cycle** | One tick-driven round producing one block: VRF selection → proposal → SMT verify → M-of-N co-sign → state transition | spec §6 |
| **Contestability** | Evidence can be challenged, but over the intact chain — not one reconstructed after the incident | spec glossary |
| **Cross-chain proof** | Dual-Merkle proof: an entry is in a sub-chain AND the sub-chain root is anchored in the parent | spec §9.2 |
| **Dilithium3** | ML-DSA-65 post-quantum signature; NIST Level 3; 2700-byte signature, 1952-byte public key | spec §3.3 |
| **ECVRF** | Elliptic Curve Verifiable Random Function (RFC 9381) over Ristretto255; used for leader election | spec §3.5 |
| **Finality** | Instant: one cycle = one final block; no forks, rollbacks, or reorganization | spec §6.1 |
| **Kyber1024** | ML-KEM-1024 post-quantum key encapsulation; NIST Level 5; used for peer-to-peer channels | spec §3.4 |
| **Laplacian λ₁** | Fiedler eigenvalue of the network Laplacian; supervises diffusion health; below `MinLambda1` signals fragmentation | spec §6.4, `pkg/state` |
| **Mandate** | Signed, versioned, anchored declaration of which event classes require anchoring | spec §13 |
| **MandateRef** | `ProvenanceEntry` field (CBOR key 7) referencing the mandate an entry satisfies | spec §4.2 |
| **M-of-N quorum** | Configurable threshold of validators required to finalize a block | spec §4.3 |
| **Non-repudiation** | Cryptographic guarantee that a party cannot deny submitting an entry (Dilithium3) | spec §12.2 |
| **Proposer** | Validator selected by ECVRF (lowest Gamma) to build the next block | spec §6.3.1 |
| **ProvenanceEntry** | A single anchored record: hash, submitter, timestamp, optional label/approver/reference/signature/mandate-ref | spec §4.2 |
| **QuorumConfig** | Total validators + minimum required signatures to finalize a block | spec §4.3 |
| **RootID** | 16-byte unique validator identifier derived from UID0 | spec §8.2 |
| **SMT** | Sparse Merkle Tree; depth 256; BLAKE3-256; commits to the set of all anchored entries | spec §5 |
| **Sub-chain** | Isolated per-service SMT, periodically checkpointed into the parent via cross-chain proofs | spec §9 |
| **Third-party verifiability** | Any holder of the public chain can verify proofs/signatures/hashes without the producing network | spec §1 |
| **UID0** | Soulbound identity token binding a validator to a deterministic, verifiable identity (canonical CBOR) | spec §8 |
| **Zero hash** | 32 all-zero bytes; rejected by entry validation | spec §7.2 |

## Ecosystem-Specific Terms

| Term | Definition | Source |
|------|------------|--------|
| **Gleipnir** | The Go reference implementation of 3CP | [`README.md`](../../README.md) |
| **Carcosa** | Rust ZK audit framework that couples to 3CP (does not implement it) | [`CARCOSA.md`](../../CARCOSA.md) |
| **AIR** | Algebraic Intermediate Representation — the constraint system for a Winterfell STARK | [Carcosa AIR](../04-carcosa/air.md) |
| **STARK** | Scalable Transparent ARgument of Knowledge; used by Carcosa (Winterfell) | [Carcosa proving](../04-carcosa/proving.md) |
| **Blob store** | Off-chain storage for STARK proof binaries; only the 32-byte digest is anchored | [Carcosa](../04-carcosa/architecture.md) |

## Divergence Notes

| Term | Specification | Reference implementation (Gleipnir) |
|------|---------------|-------------------------------------|
| **Mandate** | Normative type (spec §13) | **Not implemented** (`pkg/` contains no mandate code) |
| **Error codes** | Registry uses `ZERO_HASH`, `EMPTY_SUBMITTER`, etc. (spec §7.3) | Uses `INVALID_HASH`, `INVALID_SUBMITTER`, etc. (`pkg/validation/validation.go:72`) — see [error-reference](../07-reference/error-reference.md) |
| **StreamBlocks** | Defined RPC (spec §7.1) | Declared in proto, returns `Unimplemented` |

## References

- [Section 07 — Glossary](../07-reference/glossary.md) — full glossary
- [`spec/glossary.md`](../../spec/glossary.md) — source glossary
- [`spec/3CP.md`](../../spec/3CP.md) — normative specification
