# Protobuf Reference

> **Document Level:** B — Reference
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/pkg/server/api.proto`
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §7, `spec/schemas/*.cddl`
> - See also: [Section 02 — Protobuf](../02-3cp-specification/protobuf.md)

---

## Purpose

Reference for the `.proto` messages used by Gleipnir's gRPC API.

## Package

`provenance` (service `ProvenanceAnchor`). Regenerated via `make proto`.

## Messages

| Message | Key fields |
|---------|-----------|
| `SubmitRequest` | `hash`, `submitter`, `signature`, `timestamp`, `label`, `approver`, `reference` |
| `SubmitResponse` | `accepted`, `status`, `block_index`, `block_time`, `error_code` |
| `AnchorRequest` | `hash` |
| `AnchorProof` | SMT proof + `block_index` |
| `VerifyRequest` / `VerifyResponse` | `hash` / `anchored` |
| `StateRootResponse` | `root` bytes |
| `HealthResponse` | `lambda1`, `block_height`, `active_peers`, `total_peers`, `pending_hashes`, `avg_tps`, `status` |
| `BlockRequest` / `BlockResponse` | `index` / `Triad` + `Sigs` |
| `Triad` | `proposal`, `validation`, `supervision` blocks |
| `Sigs` | Multi-signature set (empty in single-node) |

## Divergences vs Spec/CDDL

- No `Mandate*` messages (spec §8) — [D2](../04-divergence/mandates.md).
- No `StreamBlocks` response streaming — [D1](../04-divergence/grpc-api.md).
- `block_index`/`block_time` unused in `SubmitResponse` — [D6](../04-divergence/grpc-api.md).
- Canonical on-chain format is CBOR (spec §4); protobuf is the transport only.

## Generation

`make proto` runs `protoc` into `pkg/server/pb/`. The REST layer uses its own
JSON bodies (see [rest-api](../03-gleipnir/rest-api.md)).

## References

- [Section 02 — Protobuf](../02-3cp-specification/protobuf.md)
- [Section 03 — grpc-api](../03-gleipnir/grpc-api.md)
- [api-reference](api-reference.md)
