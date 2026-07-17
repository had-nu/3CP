# Incident Response

> **Document Level:** E — Operational
>
> **Classification:** Operational
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Threat model: [Section 01 — Threat Model](../01-introduction/threat-model.md)
> - Operational: [`3CP_Operational_Playbook.md`](../../3CP_Operational_Playbook.md)

---

## Purpose

Runbook-style guidance for responding to common incidents.

## Scope

Operational incidents. Cryptographic attacks are in the threat model.

## Incident Catalog

| Incident | Detection | Response |
|----------|-----------|----------|
| Chain stall | `block_height` not advancing | Check `RunCycle` logs; restart; restore if corrupt |
| Fragmentation | `lambda1 < 0.10` | Inspect reputation inputs; reduce `DecayRate` |
| Data corruption | `ErrChainBroken` on boot | [Restore](restore.md) earlier backup |
| Auth failures spike | `INVALID_SIGNATURE`/`UNKNOWN_SUBMITTER` | Verify UID0s; clock skew |
| UID0 compromise | Suspected key leak | Revoke at app layer; rotate credentials ([security](security.md)) |
| Storage full | Disk alert | Rotate/prune; expand volume |

## Classification of Logs

Per `DOCUMENTATION_SPEC.md` §20, incident records use:

| Class | Use |
|-------|-----|
| `INFO` | Routine recovery actions |
| `WARN` | Degraded but operating |
| `ERROR` | Service-impacting |
| `SECURITY` | Suspected compromise / auth abuse |

> No fabricated incident logs are recorded here. Operators SHOULD append
> classified entries to their own incident ledger.

## Escalation

1. Contain (stop affected daemon).
2. Preserve evidence (backup current state before restore).
3. Restore from known-good ([disaster-recovery](disaster-recovery.md)).
4. Post-incident: update [divergence](../04-divergence/overview.md) if a gap
   caused it.

## References

- [Section 01 — Threat Model](../01-introduction/threat-model.md)
- [disaster-recovery](disaster-recovery.md), [security](security.md)
