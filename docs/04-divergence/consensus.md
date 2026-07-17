# Divergence — Consensus & Networking (D3, D4, D10)

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §6 (consensus), §7 (networking)
> - Implementation: `gleipnir-ipc/pkg/consensus/engine.go`, `pkg/identity/vrf.go`, `pkg/smt/`

---

## D3 — Multi-node BFT gossip (Critical)

**Spec:** validators exchange proposals/votes over libp2p gossip (spec §6.1),
reaching BFT consensus with a quorum of N−M.

**As-built:** `provenanced` runs a single-node `RunCycle` — it proposes and
commits blocks locally. libp2p is imported but **not wired** into the engine;
the 5-node compose topology runs 5 independent single-node chains.

**Impact:** No fault tolerance, no replication, no BFT safety across validators.
Treat as a single source of truth.

## D4 — Per-entry signature verification (Major)

**Spec:** submitted entries carry a `Signature` verified against the submitter's
Dilithium3 key before anchoring (spec §7.2.1).

**As-built:** `authenticateSubmit` exists and runs on the gRPC path
(`pkg/server/server.go`), but the per-entry `Signature` field inside an anchored
`Entry` is accepted into the SMT **without verification** during `RunCycle`.
The REST path verifies its own `X-Signature` transport signature, not the
on-chain entry signature.

**Impact:** A malformed/mismatched on-chain entry signature could be anchored;
external verifiers may reject. Mitigated by transport auth but not protocol-level.

## D10 — Block hash algorithm (Minor)

**Spec:** block hash SHOULD use BLAKE3 for consistency with the SMT/supervision
digest (spec §4).

**As-built:** `computeBlockHash` (`engine.go`) uses **SHA-256**, while the SMT
and supervision root use BLAKE3. Mixed digests.

**Impact:** Cosmetic cross-check friction; no safety break, but inconsistent with
the spec's digest policy.

## Related

- D8 (VRF hash-to-curve) → [vrf](vrf.md)
- Single-node posture in production → [Section 03 — production](../03-gleipnir/production.md)

## Remediation

1. Integrate libp2p gossip into `RunCycle`; implement vote aggregation + commit
   threshold.
2. Verify per-entry `Signature` inside the proposal step, rejecting bad entries.
3. Align `computeBlockHash` to BLAKE3 (or document the dual-digest rationale).

## References

- [spec §6 — Consensus](../02-3cp-specification/consensus.md)
- [spec §7 — Networking](../02-3cp-specification/networking.md)
- [overview](overview.md) (D3, D4, D10)
