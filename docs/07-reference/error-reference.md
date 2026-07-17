# Error Reference

> **Document Level:** B — Reference
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §7.3
> - Implementation: `gleipnir-ipc/pkg/validation/validation.go:72`, `pkg/rest/server.go:31`
> - See also: [divergence D9](../04-divergence/errors.md)

---

## Purpose

Map error codes across the spec registry and the Gleipnir implementations.

## Spec Registry (§7.3)

`ZERO_HASH`, `EMPTY_SUBMITTER`, `EMPTY_SIGNATURE`, `INVALID_SIGNATURE`,
`SUBMITTER_UNKNOWN`, `LABEL_TOO_LONG`, `RATE_LIMITED`, `ENGINE_STOPPED`.

## Gleipnir gRPC (`pkg/validation`)

| Code | Meaning |
|------|---------|
| `INVALID_HASH` | Hash malformed / zero (spec: `ZERO_HASH`) |
| `INVALID_SUBMITTER` | Empty/unknown submitter (spec: `EMPTY_SUBMITTER`) |
| `INVALID_SIGNATURE` | Bad signature (spec: `EMPTY_SIGNATURE`/`INVALID_SIGNATURE`) |
| `SUBMITTER_MISMATCH` | Signature key ≠ submitter (extra) |
| `UNKNOWN_SUBMITTER` | Not in store (spec: `SUBMITTER_UNKNOWN`) |
| `LABEL_TOO_LONG` | Matches spec |
| `RATE_LIMITED` | Matches spec |
| `ENGINE_STOPPED` | Extra (shutdown) |

## Gleipnir REST (`pkg/rest`)

`INVALID_JSON`, `INVALID_HASH`, `EMPTY_SUBMITTER`, `LABEL_TOO_LONG`,
`EXTRA_FIELDS`, `STALE_TIMESTAMP`, `UNKNOWN_ROOT`, `INVALID_SIGNATURE`,
`NOT_FOUND`, `RATE_LIMITED`, `QUEUE_FULL`, `INTERNAL_ERROR`.

## Mapping Table

| Spec | gRPC | REST |
|------|------|------|
| `ZERO_HASH` | `INVALID_HASH` | `INVALID_HASH` |
| `EMPTY_SUBMITTER` | `INVALID_SUBMITTER` | `EMPTY_SUBMITTER` |
| `EMPTY_SIGNATURE` | `INVALID_SIGNATURE` | `INVALID_SIGNATURE` |
| `SUBMITTER_UNKNOWN` | `UNKNOWN_SUBMITTER` | `UNKNOWN_ROOT` |
| `LABEL_TOO_LONG` | `LABEL_TOO_LONG` | `LABEL_TOO_LONG` |
| `RATE_LIMITED` | `RATE_LIMITED` | `RATE_LIMITED` |
| — | `SUBMITTER_MISMATCH` | `UNKNOWN_ROOT` |
| — | — | `STALE_TIMESTAMP`, `EXTRA_FIELDS`, `QUEUE_FULL`, `INTERNAL_ERROR` |

## Guidance

Code clients against the **gRPC** set for the canonical path; the REST set is a
convenience surface with its own names. See
[divergence D9](../04-divergence/errors.md) for remediation.

## References

- [Section 03 — grpc-api](../03-gleipnir/grpc-api.md),
  [rest-api](../03-gleipnir/rest-api.md)
- [divergence D9](../04-divergence/errors.md)
