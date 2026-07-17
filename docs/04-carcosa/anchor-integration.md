# 3CP Integration

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `carcosa/cli/src/commands/anchor.rs`
> - Design: `carcosa/docs/ARCHITECTURE.md` §2
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §8, §13.5

---

## Purpose

Document how Carcosa anchors a proof digest into Gleipnir and the current
integration status.

## Scope

The 32-byte BLAKE3 digest of a STARK proof blob, submitted as a `ProvenanceEntry`
with a `MandateRef`. The gRPC surface is in
[Section 03 — grpc-api](../03-gleipnir/grpc-api.md).

## Intended Integration

```mermaid
sequenceDiagram
    participant C as carcosa anchor
    participant G as Gleipnir (SubmitHash)
    C->>G: hash=BLAKE3(proof.bin), label="zk:<t>of<n>", MandateRef
    G-->>C: AnchorProof (SMT proof)
    Note over C,G: only the 32B digest is on-chain; proof stays off-chain
```

- `ProvenanceEntry.Hash` = BLAKE3(proof) (32B)
- `Submitter` = e.g. `zk-poc-1`
- `Label` = `zk:2of3`
- `MandateRef` = `BLAKE3(mandate)` (32B)

This keeps the chain small and private: the proof binary lives off-chain in the
blob store; only its digest is verifiable on-chain.

## As-Built

- **`carcosa anchor`** (`anchor.rs`): parses `--hash --label --mandate-ref
  --gleipnir-addr` and **prints** `[OK] entry anchored`. It does **not** open a
  gRPC connection to Gleipnir and does **not** call `SubmitHash`. The
  integration is a **stub** (D12).
- Carcosa has **no gRPC client crate** (`proto/` and `grpc/` are empty/absent).

## Dependencies

- Requires Gleipnir `SubmitHash` + mandates ([divergence D2](../04-divergence/mandates.md)).
- Requires adding a `tonic` client in Carcosa.

## Known Gaps

- No real submission; the digest-anchoring loop is unimplemented.
- `MandateRef` is never fetched from Gleipnir (mandates absent).

## References

- [Section 03 — grpc-api](../03-gleipnir/grpc-api.md),
  [rest-api](../03-gleipnir/rest-api.md)
- [audit](audit.md), [proving](proving.md)
- [divergence D2](../04-divergence/mandates.md),
  [D12](../04-divergence/carcosa.md)
