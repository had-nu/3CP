# Gleipnir gRPC API

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §7
> - Implementation: `gleipnir-ipc/pkg/server/api.proto`, `pkg/server/server.go`, `pkg/server/pb/`
> - Tests: `pkg/server/api_test.go`, `cmd/conformance-test/`

---

Per `DOCUMENTATION_SPEC.md` §12, each endpoint documents purpose, request,
response, validation, error conditions, authentication, and examples.

## Purpose

Document the gRPC surface Gleipnir exposes: the `ProvenanceAnchor` service, its
methods, authentication, and error codes.

## Scope

The gRPC service in `pkg/server`. Message schemas are in
[Section 02 — Protobuf](../02-3cp-specification/protobuf.md) and
[Section 07 — Protobuf Reference](../07-reference/protobuf-reference.md).

## Service

`api.proto` package `provenance`, service `ProvenanceAnchor`. Wired in
`cmd/provenanced/main.go` (`RegisterProvenanceAnchorServer`); gRPC reflection is
enabled. Default port `50051` (`--grpc-port` / `IPC_GRPC_PORT`).

## Methods

| Method | Status | Handler |
|--------|--------|---------|
| `SubmitHash` | Implemented | `server.go:64` |
| `WaitForAnchor` | Implemented | `server.go:142` |
| `VerifyHash` | Implemented | `server.go:162` |
| `GetCurrentStateRoot` | Implemented | `server.go:182` |
| `GetHealth` | Implemented | `server.go:193` |
| `GetBlock` | Implemented | `server.go:210` (returns `Triad` + `Sigs`) |
| `StreamBlocks` | **Unimplemented** | Returns `Unimplemented` (conformance D1 asserts this) |
| `SubmitMandate` / `GetMandate` / `GetActiveMandates` | **Absent** | Not in `api.proto` |

## SubmitHash

**Purpose:** submit a provenance hash for anchoring.

**Request (`SubmitRequest`):** `hash`, `submitter`, `signature`, `timestamp`,
`label`, `approver`, `reference`.

**Authentication (spec §7.2.1):** the server MUST authenticate the caller.
`authenticateSubmit` verifies a Dilithium3 signature over
`hash || submitter || LE64(timestamp) || label` against the Dilithium3 public
key of the claimed `submitter`.

**Validation:** delegated to `pkg/validation` — rejects invalid hash, empty
submitter, over-long label, invalid signature, unknown submitter, rate-limited.

**Response (`SubmitResponse`):** `accepted`, `status`, `block_index`,
`block_time`, `error_code`.

> `block_index` and `block_time` are always 0 in the response — they are not
> populated at submit time (the entry is not yet anchored).

**Example (illustrative log):**
```
# Illustrative — grpcurl-style
grpcurl -d '{"hash":"...","submitter":"...","signature":"...","timestamp":...,"label":"release:gate"}' \
  localhost:50051 provenance.ProvenanceAnchor/SubmitHash
```

## WaitForAnchor / VerifyHash

- `WaitForAnchor(hash)` blocks until the hash is anchored, returning the
  `AnchorProof` (SMT proof + block index). Client polls at 100ms internally.
- `VerifyHash(hash)` checks whether a hash is anchored (non-blocking).

Recommended gate sequence (spec §11.1): `SubmitHash` → `WaitForAnchor` →
`GetBlock` → verify SMT proof client-side.

## GetCurrentStateRoot / GetHealth / GetBlock

| Method | Returns |
|--------|---------|
| `GetCurrentStateRoot` | Current SMT root bytes |
| `GetHealth` | `lambda1`, `block_height`, `active_peers`, `total_peers`, `pending_hashes`, `avg_tps` |
| `GetBlock(index)` | Full block including `Triad` and `Sigs` |

## Error Conditions

`SubmitResponse.error_code` carries codes from `pkg/validation/validation.go:72`:
`INVALID_HASH`, `INVALID_SUBMITTER`, `INVALID_SIGNATURE`, `SUBMITTER_MISMATCH`,
`LABEL_TOO_LONG`, `RATE_LIMITED`, `ENGINE_STOPPED`. These **differ from the
spec §7.3 registry** (e.g. `ZERO_HASH`, `EMPTY_SUBMITTER`). See
[Section 07 — Error Reference](../07-reference/error-reference.md).

## Timeouts, Retry, Idempotency

- `SubmitHash` is effectively idempotent by hash — duplicate hashes dedup at
  proposal time (spec §6.3.2).
- Clients SHOULD set a `WaitForAnchor` timeout ≥ `3 × cycleInterval` (spec §11.2)
  and fail closed on timeout.

## Implementation Divergence

> - `StreamBlocks` unimplemented; mandate RPCs absent.
> - Per-entry `Signature` accepted but not verified server-side.
> - Error codes differ from the spec registry.

## References

- [spec §7 — gRPC API](../02-3cp-specification/networking.md)
- [Section 02 — Protobuf](../02-3cp-specification/protobuf.md)
- [Section 07 — Protobuf Reference](../07-reference/protobuf-reference.md),
  [API Reference](../07-reference/api-reference.md),
  [Error Reference](../07-reference/error-reference.md)
- [Section 06 — Implementing a Client](../06-developer-guide/implementing-a-client.md)
