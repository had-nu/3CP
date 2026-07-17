# Divergence — Carcosa (D11, D12)

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Design: [`CARCOSA.md`](../../CARCOSA.md)
> - Implementation: `carcosa/air/src/approval_air.rs`, `carcosa/cli/src/main.rs`
> - Reference: `carcosa/docs/BLUEPRINT.md`

---

## Spec / Design Intent

Carcosa is the ZK-audit component: it generates Winterfell STARK proofs
attesting that an anchoring quorum satisfied its active mandates, stores the
proof blobs off-chain (32-byte digest anchored on-chain), and serves them via
gRPC (spec §8, §13.5; `CARCOSA.md`).

## As-Built (what exists)

| Component | Status |
|-----------|--------|
| Winterfell AIR (`air/src/approval_air.rs`) | **Implemented** — 3-column trace, 2 transition constraints (HASH_INIT, HASH_STEP) |
| CLI arg-parsing (`cli/src/main.rs`) | **Implemented** — `--mode`, `--in`, `--out`, `--field` |
| Prover | Absent (no `prover/` crate) |
| Verifier | Absent (no `verifier/` crate) |
| gRPC server | Absent (`proto/`, `grpc/` only stubs) |
| Blob store | Absent (`store/` print-only) |
| Audit logic | Absent (`audit/` print-only) |
| Test data | Absent (`test-data/` placeholder) |
| Docker | Absent (`docker/` placeholder) |

## D11 — Prover/Verifier planned (Planned)

The AIR defines the constraint shape but **no prover or verifier executes it**.
Public-input digests are declared but **not enforced** by the AIR constraints
(`main.rs` help text notes this). Carcosa cannot yet produce or check a proof.

## D12 — Blob-store / gRPC planned (Planned)

The digest-anchoring flow (compute 32-byte BLAKE3 of proof blob, submit to
Gleipnir) is designed but the `store/` and `grpc/` crates only print intent.

## Impact

| Dimension | Consequence |
|-----------|-------------|
| ZK audit | Not available end-to-end |
| Mandate proofs | Gleipnir cannot emit them (see [mandates](mandates.md)) |
| Spec conformance | Carcosa is pre-alpha relative to spec §8 |

**Severity: Planned** — documented as future work, clearly separated from
as-built.

## Remediation

1. Implement the Winterfell `Prover`/`Verifier` over `ApprovalAir`.
2. Enforce public-input digests as boundary constraints (close the noted gap).
3. Add `store/` (off-chain blob persistence + BLAKE3 digest) and `grpc/`.
4. Wire `audit/` to consume mandate proofs from Gleipnir.

## References

- [Section 04 — Carcosa](../04-carcosa/architecture.md)
- [overview](overview.md) (D11, D12)
- [mandates](mandates.md)
