# API Reference

> **Document Level:** B — Reference
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/pkg/server/api.proto`, `pkg/server/server.go`, `pkg/rest/server.go`
> - See also: [Section 03 — grpc-api](../03-gleipnir/grpc-api.md),
>   [rest-api](../03-gleipnir/rest-api.md)

---

## Purpose

Concise index of every RPC and REST endpoint with status.

## gRPC — `ProvenanceAnchor`

| Method | Status | Notes |
|--------|--------|-------|
| `SubmitHash` | ✅ | `block_index`/`block_time` = 0 (D6) |
| `WaitForAnchor` | ✅ | Polls at 100ms |
| `VerifyHash` | ✅ | Non-blocking |
| `GetCurrentStateRoot` | ✅ | |
| `GetHealth` | ✅ | `lambda1`, `block_height`, peers, `avg_tps` |
| `GetBlock` | ✅ | Returns `Triad` + `Sigs` |
| `StreamBlocks` | ❌ | `Unimplemented` (D1) |
| `SubmitMandate` / `GetMandate` / `GetActiveMandates` | ❌ | Absent (D2) |

## REST — `:8080`

| Method + Path | Status | Auth |
|---------------|--------|------|
| `POST /v1/submit` | ✅ | Required |
| `GET /v1/verify/{hash}` | ✅ | Required |
| `GET /v1/block/{index}` | ✅ | Required |
| `GET /v1/health` | ✅ | Optional |
| `GET /v1/state-root` | ✅ | Optional |

Auth: `X-UID0-RootID` + `X-Signature` + `X-Timestamp` (30s skew). Optional TLS.

## Message Schemas

Full `.proto`/CDDL schemas: [protobuf-reference](protobuf-reference.md) and
[Section 02 — Protobuf](../02-3cp-specification/protobuf.md).

## Error Codes

[error-reference](error-reference.md).

## References

- [Section 03 — grpc-api](../03-gleipnir/grpc-api.md),
  [rest-api](../03-gleipnir/rest-api.md)
- [divergence overview](../04-divergence/overview.md)
