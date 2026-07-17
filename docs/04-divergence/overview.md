# 3CP Implementation Divergence — Overview

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md), `spec/schemas/*.cddl`
> - Implementation: `gleipnir-ipc/` (Go), `carcosa/` (Rust)
> - Operational: [`3CP_Operational_Playbook.md`](../../3CP_Operational_Playbook.md)

---

## Purpose

This section is the consolidated, authoritative record of where the reference
implementations (Gleipnir, Carcosa) diverge from the 3CP specification. It is
the index into the per-topic gap documents.

## Scope

All spec↔implementation gaps. Each topic has its own gap doc; this page
summarizes severity and links to detail. Cryptographic divergences are also
noted inline in Section 02.

## Divergence Severity Model

| Severity | Meaning |
|----------|---------|
| **Critical** | Breaks a normative protocol guarantee (e.g. consensus safety, verifiability) |
| **Major** | Feature absent or non-conformant; operator impact |
| **Minor** | Cosmetic / non-normative deviation |
| **Planned** | Intended, documented as future work |

## Summary Table

| # | Area | Impl | Severity | Detail |
|---|------|------|----------|--------|
| D1 | StreamBlocks RPC | Gleipnir | Major | Unimplemented (`Unimplemented` returned) — [grpc-api](grpc-api.md) |
| D2 | Mandates | Gleipnir | Critical | Entire mandate subsystem absent — [mandates](mandates.md) |
| D3 | Multi-node BFT gossip | Gleipnir | Critical | libp2p not wired; single-node consensus — [consensus](consensus.md) |
| D4 | Per-entry signature verification | Gleipnir | Major | Accepted, not verified — [consensus](consensus.md) |
| D5 | Prometheus metrics | Gleipnir | Major | `/metrics` mounted, no series — [observability](observability.md) |
| D6 | `block_index`/`block_time` | Gleipnir | Minor | Always 0 in `SubmitResponse` — [grpc-api](grpc-api.md) |
| D7 | `LambdaInterval` default | Gleipnir | Minor | 10 vs spec 0 — [state-machine](state-machine.md) |
| D8 | ECVRF hash-to-curve | Gleipnir | Major | Diverges from RFC 9381 — [vrf](vrf.md) |
| D9 | Error-code registry | Gleipnir | Minor | `INVALID_HASH` vs `ZERO_HASH` etc — [errors](errors.md) |
| D10 | Block hash algorithm | Gleipnir | Minor | SHA-256 vs BLAKE3 spec — [consensus](consensus.md) |
| D11 | Carcosa prover/verifier | Carcosa | Planned | Only Winterfell AIR present — [carcosa](carcosa.md) |
| D12 | Carcosa blob-store/gRPC | Carcosa | Planned | Absent — [carcosa](carcosa.md) |

## How to Read the Gap Docs

Each gap document follows the same shape: **Spec requirement → As-built →
Impact → Remediation**. Implementations are documented as-built; where the spec
is the target, the gap is stated without apology.

## Cross-References

- Section 02 normative docs carry inline `> -` divergence callouts.
- Section 03 (Gleipnir) and Section 04 (Carcosa) each have their own
  "Implementation Divergence" sections.
- Section 07 reference docs (errors, metrics) mirror the registry mismatches.

## References

- [Section 02 — Specification](../02-3cp-specification/protocol-overview.md)
- [Section 03 — Gleipnir](../03-gleipnir/architecture.md)
- [Section 04 — Carcosa](../04-carcosa/architecture.md)
