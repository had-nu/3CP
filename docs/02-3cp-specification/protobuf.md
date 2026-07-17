# 3CP Wire Format and Protocol Buffers

> **Document Level:** C — Protocol
>
> **Classification:** Normative (CBOR) + Reference Implementation (protobuf)
>
> **Status:** Draft
>
> **Protocol Version:** 1
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §3.6, §4, §7
> - Implementation: `gleipnir-ipc/pkg/server/api.proto`, `pkg/server/pb/`
> - Schemas: [`spec/schemas/`](../../spec/schemas/)

---

## Purpose

Define the two wire encodings 3CP uses: **canonical CBOR** for on-chain
structures (normative) and **Protocol Buffers** for the gRPC transport
(reference implementation). Clarify which is the protocol boundary.

## Scope

CBOR encoding rules and CDDL schemas (normative), and the Gleipnir `api.proto`
service and messages (reference implementation).

## Two Encodings, One Boundary

| Encoding | Used for | Normativity |
|----------|----------|-------------|
| Canonical CBOR | On-chain: blocks, entries, mandates, UID0 | **Normative** (spec §3.6, §4, §8) |
| Protocol Buffers | gRPC request/response messages | Transport realisation (spec §2) |

The **normative protocol boundary** is the CBOR structures and consensus rules,
not the protobuf messages. A conformant implementation MUST produce identical
CBOR bytes for identical content; the protobuf layer is the canonical transport
that carries requests to and from the node.

## Canonical CBOR Rules (spec §3.6)

- Integer map keys in ascending order.
- Smallest integer representation (`uint` ≤ 23 in one byte).
- Definite-length strings and byte strings.
- 64-bit IEEE 754 floats.

Implementation: `pkg/identity/cbor.go` (`CanonicalEncOptions().EncMode()`).

## CDDL Schemas

The normative wire structures are defined in CDDL
([RFC 8610](https://tools.ietf.org/html/rfc8610)):

| Schema | Structure | Document |
|--------|-----------|----------|
| [`block.cddl`](../../spec/schemas/block.cddl) | Block, ProvenanceEntry, QuorumConfig | [anchoring](anchoring.md) |
| [`mandate.cddl`](../../spec/schemas/mandate.cddl) | MandateEntry, Rule | [anchoring](anchoring.md) |
| [`cross-chain.cddl`](../../spec/schemas/cross-chain.cddl) | CrossChainProof | [anchoring](anchoring.md) |
| [`smt.cddl`](../../spec/schemas/smt.cddl) | SMT proof | [sparse-merkle-tree](sparse-merkle-tree.md) |
| [`vrf.cddl`](../../spec/schemas/vrf.cddl) | VRFProof | [vrf](vrf.md) |

## Protocol Buffers (Reference Implementation)

`pkg/server/api.proto` — package `provenance`, service `ProvenanceAnchor`.

### RPCs (as implemented)

| RPC | Status |
|-----|--------|
| `SubmitHash` | Implemented |
| `WaitForAnchor` | Implemented |
| `VerifyHash` | Implemented |
| `GetCurrentStateRoot` | Implemented |
| `GetHealth` | Implemented |
| `GetBlock` | Implemented |
| `StreamBlocks` | Declared, returns `Unimplemented` |

### Messages

`SubmitRequest` (hash, submitter, signature, timestamp, label, approver,
reference), `SubmitResponse` (accepted, status, block_index, block_time,
error_code), `AnchorProof`, `StateRootResponse`, `HealthResponse`, `Block`,
`BlockRange`, `ProvenanceEntry`, `Empty`.

Generated code: `pkg/server/pb/api.pb.go`, `pkg/server/pb/api_grpc.pb.go`
(regenerated via `make proto`). Full reference:
[Section 07 — Protobuf Reference](../07-reference/protobuf-reference.md).

## Implementation Divergence

> | Spec (§7) | `api.proto` |
> |-----------|-------------|
> | `SubmitMandate`, `GetMandate`, `GetActiveMandates` | **Absent** — no mandate RPCs or messages |
> | `StreamBlocks` | Declared but `Unimplemented` |
> | `SubmitResponse.block_index` / `block_time` | Present but always 0 (populated only after anchoring, not at submit) |
>
> The protobuf surface is therefore a subset of the spec §7 service.

## References

- [`spec/3CP.md`](../../spec/3CP.md) §3.6, §4, §7
- [RFC 8610](https://tools.ietf.org/html/rfc8610) — CDDL
- [RFC 8949](https://www.rfc-editor.org/info/rfc8949) — CBOR
- [Section 07 — Protobuf Reference](../07-reference/protobuf-reference.md)
- [networking](networking.md), [anchoring](anchoring.md)
