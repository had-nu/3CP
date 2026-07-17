# Cluster Bootstrap

> **Document Level:** E — Operational
>
> **Classification:** Operational
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Operational: [`3CP_Operational_Playbook.md`](../../3CP_Operational_Playbook.md) §Boot Sequence
> - Implementation: `gleipnir-ipc/docker-compose.yml`, `cmd/provectl/`

---

## Purpose

Bootstrap a multi-validator Gleipnir topology from cold start.

## Scope

Identity generation and daemon launch. Note the current single-node limitation
(see [divergence D3](../04-divergence/consensus.md)) — the compose topology runs
5 independent single-node chains.

## Boot Sequence (per playbook)

```mermaid
sequenceDiagram
    participant Op as Operator
    participant B as bootstrap svc
    participant V as validators
    Op->>B: provectl init --validators 5 --out /uids
    B->>B: generate UID0 CBORs (Dilithium3)
    B->>V: mount /uids
    Op->>V: start provenanced (--uid-file, --node-id, --peers)
    V->>V: load state / rebuild SMT
    Note over V: resume at last Cycle; cycles advance
```

## Steps

1. **Generate identities:** `provectl init --validators 5 --out /uids` (the
   compose `bootstrap` service does this).
2. **Mount UID0s:** each validator mounts the `uids` volume and gets its
   `--uid-file` + `--node-id`.
3. **Start daemons:** `docker compose up -d validator-1 .. validator-5`.
4. **Verify:** `GetHealth` shows `block_height` advancing; `GetCurrentStateRoot`
   stable per node.

## Caveats

- The daemons do **not** gossip/consensus with each other today. Treat each as
  an independent instance.
- `--peers` is accepted but unused by the engine.

## References

- [Section 03 — deployment](../03-gleipnir/deployment.md)
- [divergence D3](../04-divergence/consensus.md)
- [rolling-upgrades](rolling-upgrades.md)
