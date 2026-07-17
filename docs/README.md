# 3CP Ecosystem Documentation

This documentation set covers the **3CP protocol**, its Go reference
implementation **Gleipnir**, the Rust ZK-audit framework **Carcosa**, and the
operational/developer/reference material needed to run and extend them.

All documents follow the rules in `DOCUMENTATION_SPEC.md`: every file carries a
classification/status header, divergence callouts where the implementation
departs from the specification, Mermaid diagrams, and classified log guidance.

## How to Read

Start with the introduction, then the normative specification. Reference
implementations are documented as-built; where they diverge from the spec, the
gap is recorded explicitly in **Section 04 — Divergence**.

## Sections

| Section | Topic | Docs |
|---------|-------|------|
| [01-introduction](../01-introduction/) | Vision, architecture overview, terminology, threat model | 4 |
| [02-3cp-specification](../02-3cp-specification/) | Normative protocol: overview, crypto, consensus, SMT, VRF, anchoring, networking, protobuf, state machine | 9 |
| [03-gleipnir](../03-gleipnir/) | Gleipnir (Go) architecture, engine, state machine, scheduler, storage, gRPC/REST API, metrics, deployment, production, troubleshooting | 11 |
| [04-divergence](../04-divergence/) | Consolidated spec↔implementation gap analysis (overview + per-topic) | 9 |
| [04-carcosa](../04-carcosa/) | Carcosa (Rust) as-built: architecture, AIR, CLI, proving, verification, witness, trace generation, audit, 3CP integration | 9 |
| [05-operations](../05-operations/) | Security, observability, backup, restore, disaster recovery, cluster bootstrap, rolling upgrades, incident response, performance, capacity planning | 10 |
| [06-developer-guide](../06-developer-guide/) | Repository layout, implementing a client, extending the engine | 7 |
| [07-reference](../07-reference/) | Glossary, error reference, metrics reference, API reference, configuration reference, protobuf reference | 7 |

## Key Implementation Gaps (summary)

Critical gaps operators must know (full detail in Section 04):

- **Mandates** — absent in Gleipnir (spec §8). [divergence](../04-divergence/mandates.md)
- **Multi-node BFT** — libp2p gossip not wired; single-node consensus. [divergence](../04-divergence/consensus.md)
- **Carcosa** — only the Winterfell AIR + CLI scaffold exist; no prover/verifier/blob-store/gRPC. [divergence](../04-divergence/carcosa.md)
- **Metrics** — `/metrics` mounted but no series emitted. [divergence](../04-divergence/observability.md)
- **ECVRF** — custom hash-to-curve diverges from RFC 9381. [divergence](../04-divergence/vrf.md)

## Sources

- Specification: [`spec/3CP.md`](../../spec/3CP.md)
- Gleipnir: `gleipnir-ipc/` (Go)
- Carcosa: `carcosa/` (Rust)
- Operational playbook: [`3CP_Operational_Playbook.md`](../../3CP_Operational_Playbook.md)
