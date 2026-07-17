# Divergence — gRPC API (D1, D6)

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §7
> - Implementation: `gleipnir-ipc/pkg/server/api.proto`, `pkg/server/server.go`

---

## D1 — StreamBlocks unimplemented (Major)

**Spec:** `StreamBlocks` streams committed blocks to subscribers (spec §7).

**As-built:** the handler returns `status.Error(codes.Unimplemented, ...)`. The
`conformance-test` asserts this explicitly. No streaming subscription exists.

**Impact:** Clients cannot subscribe to the block stream; they must poll
`GetBlock`. Confirms a test (`conformance D1`).

## D6 — SubmitResponse fields zeroed (Minor)

**Spec:** `SubmitResponse.block_index` / `block_time` SHOULD reflect the
anchored placement upon acceptance.

**As-built:** both are hardcoded to `0` at submit time (entry not yet anchored).

**Impact:** Clients must call `WaitForAnchor` → `GetBlock` to learn placement.
No safety break; just incomplete response.

## Related

- D2 (mandates RPCs absent) → [mandates](mandates.md)
- Full surface → [Section 03 — grpc-api](../03-gleipnir/grpc-api.md)

## Remediation

1. Implement `StreamBlocks` with engine block-commit subscription.
2. Populate `block_index`/`block_time` (or document the deferred-fill contract).

## References

- [spec §7 — Networking](../02-3cp-specification/networking.md)
- [overview](overview.md) (D1, D6)
