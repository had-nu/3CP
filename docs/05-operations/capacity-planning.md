# Capacity Planning

> **Document Level:** E — Operational
>
> **Classification:** Operational
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/pkg/storage/bolt.go`, `pkg/smt/`
> - See also: [performance](performance.md)

---

## Purpose

Sizing guidance for Gleipnir storage and compute.

## Scope

Disk, memory, and node-count planning. No measured utilization is published.

## Storage

- BoltDB holds the full chain (blocks), the SMT, pending, anchored, and state.
- SMT depth is 256 (BLAKE3); on-disk size scales with distinct anchored keys
  plus internal nodes. Plan for steady growth with chain length.
- UID0 CBORs are small and stored outside the data dir.

## Compute

- Single goroutine cycle loop; modest CPU.
- λ₁ power iteration (every `LambdaInterval` cycles) is the periodic CPU peak.
- Memory scales with `NetworkState.Nodes` and pending queue depth.

## Node Count

- Today: independent single-node instances (no quorum). Plan per-instance
  capacity; a future multi-node release changes this ([divergence D3](../04-divergence/consensus.md)).
- Quorum math (N−M tolerance) applies once gossip lands; size N for the
  required fault tolerance.

## Backup Capacity

- Each [backup](backup.md) is a full copy of the data dir. Retention policy must
  account for growth.

## Recommendations

1. Provision disk with headroom for chain growth; alert at 70% util.
2. Monitor `pending_hashes`; sustained growth signals a throughput/cycle issue.
3. Schedule backups per [backup](backup.md).

## References

- [performance](performance.md), [backup](backup.md)
- [Section 03 — storage](../03-gleipnir/storage.md)
