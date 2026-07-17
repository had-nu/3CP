# Gleipnir Troubleshooting

> **Document Level:** E — Operational
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/pkg/consensus/engine.go`, `pkg/state/apply.go`, `pkg/storage/bolt.go`, `pkg/server/server.go`
> - Operational: [`3CP_Operational_Playbook.md`](../../3CP_Operational_Playbook.md) §Disaster Recovery

---

## Purpose

Reference for diagnosing common Gleipnir failures, their symptoms, root causes,
and remediations. Operational runbooks are in
[Section 05 — Operations](../05-operations/).

## Scope

Engine, state, storage, and API failure modes observable in the reference
implementation.

## Symptom Index

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| Daemon fails to start, "database locked" | Second process on data dir | Ensure single process per `data/` volume |
| `ErrChainBroken` on boot | `SupervisionRoot` mismatch | Restore from backup; see [restore](../05-operations/restore.md) |
| `ErrNetworkFragmented` | λ₁ < `MinLambda1` with ≥2 nodes | Check peer/reputation inputs; reduce decay |
| `SubmitHash` returns `INVALID_SIGNATURE` | Bad Dilithium3 sig | Regenerate signature over exact canonical bytes |
| `SubmitHash` returns `UNKNOWN_SUBMITTER` | UID not in store | Register identity; check `provectl` output |
| `WaitForAnchor` times out | Entry not anchored | Confirm engine cycling; increase timeout |
| `GetBlock` returns no `Sigs` | Single-node commit | Expected; multi-sig pending |
| `/metrics` empty of 3CP series | Instrumentation absent | Use `GetHealth`; see [metrics](metrics.md) |
| `--uid-file`/`--node-id` missing | Required flag | Provide both; see [deployment](deployment.md) |
| REST `STALE_TIMESTAMP` | Clock skew >30s | Sync clocks on caller |

## Diagnostics

- **Health:** `grpcurl ... GetHealth` or `GET /v1/health` → inspect `lambda1`,
  `pending_hashes`, `block_height`.
- **State root:** `GetCurrentStateRoot` to confirm SMT continuity across restarts.
- **BoltDB:** inspect the data dir; schema version in `schema` bucket (v1).
- **Logs:** daemon logs commit cycle indices — a stall (no new blocks) points at
  `RunCycle` errors recovered in `cycleLoop`.

## Known Limitations

- No multi-node replication — a single validator is the only source of truth.
- Per-entry signature verification is not enforced; a bad signature may pass the
  store but fail external verification.
- `block_index`/`block_time` in `SubmitResponse` are always 0.

## Disaster Recovery

If the data dir is corrupted or `ErrChainBroken` cannot be resolved, restore the
most recent BoltDB backup and resume. Procedure:
[Section 05 — Disaster Recovery](../05-operations/disaster-recovery.md).

## References

- [Section 05 — Operations](../05-operations/) (backup, restore, DR)
- [engine](engine.md), [state-machine](state-machine.md), [storage](storage.md)
- [Section 04 — Divergence](../04-divergence/overview.md)
