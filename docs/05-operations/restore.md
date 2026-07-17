# Restore

> **Document Level:** E — Operational
>
> **Classification:** Operational
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/pkg/consensus/engine.go` (`SetStorage`/`loadPersisted`)
> - See also: [backup](backup.md)

---

## Purpose

Restore a Gleipnir node from a backup.

## Scope

Replacing the data directory and resuming the engine.

## Procedure

1. Stop `provenanced`.
2. Replace the data directory with the backup copy.
3. Confirm UID0 CBORs referenced by `--uid-file` are present and match.
4. Start `provenanced`. The engine's `loadPersisted` reads state, blocks,
   pending, anchored, and the SMT, resuming at the last `Cycle` with the correct
   SMT root.
5. Verify `GetCurrentStateRoot` matches expectations and `block_height`
   continues incrementing.

## Failure Modes

| Symptom | Cause | Action |
|---------|-------|--------|
| `ErrChainBroken` on boot | `SupervisionRoot` mismatch (corrupt/partial backup) | Use an earlier backup |
| SMT deserialization error | Incompatible schema | Confirm same `schemaVersion` (v1) |
| Block height regresses | Wrong backup selected | Re-restore correct snapshot |

## References

- [Section 03 — storage](../03-gleipnir/storage.md)
- [backup](backup.md), [disaster-recovery](disaster-recovery.md)
