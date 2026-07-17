# Divergence — Error Code Registry (D9)

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §7.3 (error registry)
> - Implementation: `gleipnir-ipc/pkg/validation/validation.go:72`

---

## Spec Requirement

The spec defines a submission error registry (§7.3) including `ZERO_HASH`,
`EMPTY_SUBMITTER`, `EMPTY_SIGNATURE`, `INVALID_SIGNATURE`,
`SUBMITTER_UNKNOWN`, `LABEL_TOO_LONG`, `RATE_LIMITED`, etc.

## As-Built

`pkg/validation/validation.go:72` defines a different set:

| Spec code | Gleipnir code |
|-----------|---------------|
| `ZERO_HASH` | `INVALID_HASH` |
| `EMPTY_SUBMITTER` | `INVALID_SUBMITTER` |
| `EMPTY_SIGNATURE` | `INVALID_SIGNATURE` |
| `SUBMITTER_UNKNOWN` | `UNKNOWN_SUBMITTER` |
| `LABEL_TOO_LONG` | `LABEL_TOO_LONG` (match) |
| — | `SUBMITTER_MISMATCH` (extra) |
| `RATE_LIMITED` | `RATE_LIMITED` (match) |

The REST layer has yet another set (`INVALID_JSON`, `EXTRA_FIELDS`,
`STALE_TIMESTAMP`, `UNKNOWN_ROOT`, `QUEUE_FULL`, `INTERNAL_ERROR`).

## Impact

| Dimension | Consequence |
|-----------|-------------|
| Interop | Clients coded against the spec registry mis-map errors |
| Spec conformance | Fails §7.3 |
| Severity | Minor — semantics overlap, names differ |

## Remediation

Either adopt the spec registry names in `pkg/validation` (add aliases for
backward compat) or extend the spec to ratify the implementation set. Document
the chosen mapping in [Section 07 — Error Reference](../07-reference/error-reference.md).

## References

- [Section 03 — grpc-api](../03-gleipnir/grpc-api.md)
- [Section 07 — Error Reference](../07-reference/error-reference.md)
- [overview](overview.md) (D9)
