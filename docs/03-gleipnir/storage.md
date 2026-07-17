# Gleipnir Storage

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §5 (SMT), §4 (blocks)
> - Implementation: `gleipnir-ipc/pkg/storage/bolt.go`, `pkg/consensus/storage.go`, `pkg/smt/smt_serialization.go`
> - Tests: `pkg/storage/persisttest/persistence_test.go`

---

## Purpose

Document how Gleipnir persists and recovers chain state: the storage interface,
the BoltDB backend, and boot-time recovery (chain height + SMT rebuild).

## Scope

The `EngineStorage` interface, the `BoltStorage` backend, and serialization of
the SMT and state. Backup/restore procedures are in
[Section 05 — Backup](../05-operations/backup.md) and
[Restore](../05-operations/restore.md).

## Storage Interface

`EngineStorage` at `pkg/consensus/storage.go:12`:

| Method group | Methods |
|--------------|---------|
| State | `SaveState`, `LoadState` |
| Blocks | `SaveBlock`, `LoadBlocks`, `GetBlock` |
| Pending | `SavePending`, `LoadPending` |
| Anchored | `SaveAnchored`, `LoadAnchored` |
| SMT | `SaveSMT`, `LoadSMT` |
| Sub-chains | `SaveSubChain`, `LoadSubChains` |
| Meta | `SaveMeta`, `LoadMeta` |
| Lifecycle | `Close` |

## BoltDB Backend

`pkg/storage/bolt.go` wraps `go.etcd.io/bbolt`.

| Bucket | Contents |
|--------|----------|
| `state` | Serialized `NetworkState` |
| `blocks` | Chain blocks |
| `pending` | Pending entries |
| `anchored` | Anchor proof index |
| `smt` | Serialized SMT |
| `subchains` | Sub-chain state |
| `meta` | Metadata |
| `schema` | Schema version (`schemaVersion = 1`) |

`NewBoltStorage` runs `initBuckets` + `migrate` (`bolt.go:55`, `bolt.go:69`).
Serialization is JSON (`json.Marshal`) for state/blocks/etc.; the SMT is
serialized via `pkg/smt/smt_serialization.go`.

## Boot Recovery

```mermaid
sequenceDiagram
    participant D as provenanced
    participant E as Engine
    participant B as BoltStorage
    D->>E: SetStorage(bolt)
    E->>B: LoadState / LoadBlocks / LoadPending / LoadAnchored / LoadSMT
    B-->>E: persisted data
    E->>E: resume at last Cycle; SMT root restored
    Note over E: chain height = len(blocks); no rebuild-from-entries needed
```

`Engine.SetStorage` → `loadPersisted()` (`pkg/consensus/engine.go:153`) reads
state, pending, anchored, blocks, and the SMT so the engine resumes at the last
`Cycle` with the correct SMT root. This is the concrete realisation of the boot
sequence "recover chain height / rebuild SMT" in the operational playbook.

## Persistence Triggers

| Trigger | Location |
|---------|----------|
| Successful block append | `pkg/consensus/engine.go:529` |
| `Stop()` | `pkg/consensus/engine.go:119` |

## Data Ownership and Caches

- The engine owns in-memory `blocks`, `pending`, `anchored`, and `st` (SMT);
  BoltDB is the durable mirror.
- The `anchored` map is an index/cache rebuildable from blocks + SMT if needed.

## Failure Modes

| Failure | Symptom | Recovery |
|---------|---------|----------|
| BoltDB file locked | Startup error | Ensure single process per data dir |
| Missing key | `storage.ErrNotFound` | Treated as absent by callers |
| SMT deserialization failure | Boot fails | Restore from backup (see [restore](../05-operations/restore.md)) |
| Partial write before crash | Last block may be absent | Re-sync from peers on rejoin |

## Persistence Notes

- Storage format is JSON internally; the **normative on-chain format is
  canonical CBOR** (spec §4). The JSON serialization is an implementation
  detail of the local store, not the wire/verification format.

## References

- [spec §5 — SMT](../02-3cp-specification/sparse-merkle-tree.md)
- [engine](engine.md)
- [Section 05 — Backup](../05-operations/backup.md),
  [Restore](../05-operations/restore.md),
  [Disaster Recovery](../05-operations/disaster-recovery.md)
