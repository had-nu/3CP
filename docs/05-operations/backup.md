# Backup

> **Document Level:** E — Operational
>
> **Classification:** Operational
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/pkg/storage/bolt.go` (BoltDB file)
> - See also: [Section 03 — storage](../03-gleipnir/storage.md)

---

## Purpose

Procedure to back up a Gleipnir node's durable state.

## Scope

The BoltDB data directory. This is the only durable state — the SMT and chain
are inside it.

## What to Back Up

The daemon's `data/` directory (the BoltDB file). It contains state, blocks,
pending, anchored, smt, subchains, meta buckets. Stop the daemon or snapshot via
filesystem freeze for a consistent copy.

## Procedure

1. Stop `provenanced` (or pause writes) for a clean snapshot.
2. Copy the data directory:
   ```bash
   # illustrative
   cp -a /var/lib/provenanced/data /backup/provenanced-data-$(date +%F)
   ```
3. Verify the copy opens (`bbolt` CLI or a temporary mount).
4. Rotate backups; retain per retention policy.

## Notes

- BoltDB is a single file; filesystem-level snapshotting (LVM/ZFS) is ideal.
- UID0 CBORs are **separate** credentials — back them up independently and
  securely (see [security](security.md)).

## Restore

See [restore](restore.md).

## References

- [Section 03 — storage](../03-gleipnir/storage.md)
- [disaster-recovery](disaster-recovery.md)
