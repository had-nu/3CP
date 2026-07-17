# Gleipnir REST API

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §7 (gRPC is canonical; REST is Gleipnir-specific)
> - Implementation: `gleipnir-ipc/pkg/rest/server.go`
> - Tests: `pkg/rest/server_test.go`
> - Reference: `gleipnir-ipc/docs/internal/SPEC-REST-API.md`

---

## Purpose

Document Gleipnir's REST surface, a convenience HTTP layer over the same engine
the gRPC service uses.

## Scope

The `rest` package. gRPC (the canonical transport) is in [grpc-api](grpc-api.md).

> **Classification note:** The specification designates gRPC as the canonical
> transport (spec §2). REST is a **reference-implementation-specific** surface,
> not a normative protocol interface.

## Endpoints

Go 1.22+ `method /path` routing (`pkg/rest/server.go`):

| Method + Path | Handler | Auth |
|---------------|---------|------|
| `POST /v1/submit` | `server.go:196`/`350` | Required |
| `GET /v1/verify/{hash}` | `server.go:197`/`437` | Required |
| `GET /v1/block/{index}` | `server.go:198`/`468` | Required |
| `GET /v1/health` | `server.go:199`/`529` | Optional |
| `GET /v1/state-root` | `server.go:200`/`545` | Optional |

Default listen `:8080` (`--rest-listen` / `IPC_REST_LISTEN`).

## POST /v1/submit

**Request body:** `{ "hash": "...", "submitter": "...", "label": "..." }`.

**Behavior:** `withAuth` → enqueue → wait for anchor → return `block_index`,
`state_root`, `smt_proof`.

**Response (illustrative):**
```json
{
  "block_index": 249,
  "state_root": "…hex…",
  "smt_proof": "…hex…"
}
```

## Authentication

Headers: `X-UID0-RootID`, `X-Signature`, `X-Timestamp`. The server verifies a
Dilithium3 signature over `body || METHOD || PATH || timestamp` with a **30s
skew window** (`server.go:255`). CORS handled by `withCORS`.

Options: `WithKeysDir` (load UID0 CBORs), `WithAllowedRoots` (whitelist),
`WithRateLimit`.

## TLS

Optional `--rest-tls-cert` / `--rest-tls-key`
(`IPC_REST_TLS_CERT` / `IPC_REST_TLS_KEY`); the server warns on plain HTTP
(`server.go:214`). Production deployments SHOULD enable TLS — see
[Section 05 — Security](../05-operations/security.md).

## Error Codes

REST uses its own constant set (`server.go:31`): `INVALID_JSON`, `INVALID_HASH`,
`EMPTY_SUBMITTER`, `LABEL_TOO_LONG`, `EXTRA_FIELDS`, `STALE_TIMESTAMP`,
`UNKNOWN_ROOT`, `INVALID_SIGNATURE`, `NOT_FOUND`, `RATE_LIMITED`, `QUEUE_FULL`,
`INTERNAL_ERROR`. See [Section 07 — Error Reference](../07-reference/error-reference.md).

## Failure Modes

| Failure | Response |
|---------|----------|
| Malformed JSON / extra fields | `INVALID_JSON` / `EXTRA_FIELDS` |
| Stale timestamp (>30s) | `STALE_TIMESTAMP` |
| Unknown root / bad signature | `UNKNOWN_ROOT` / `INVALID_SIGNATURE` |
| Queue saturated | `QUEUE_FULL` |

## Implementation Divergence

> - The REST label limit and error set differ from both the gRPC path and the
>   spec registry; treat REST as a Gleipnir convenience surface.
> - No mandate endpoints.

## References

- [grpc-api](grpc-api.md), [spec §7](../02-3cp-specification/networking.md)
- [Section 05 — Security](../05-operations/security.md)
- [Section 07 — Error Reference](../07-reference/error-reference.md)
