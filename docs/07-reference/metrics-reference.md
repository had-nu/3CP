# Metrics Reference

> **Document Level:** B — Reference
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §11.3
> - Implementation: see [Section 03 — metrics](../03-gleipnir/metrics.md) and
>   [divergence D5](../04-divergence/observability.md)

---

## Purpose

Define the intended Prometheus metric set (spec §11.3) and its current status.

## Intended Metrics (spec §11.3)

| Metric | Type | Meaning |
|--------|------|---------|
| `3cp_block_height` | Gauge | Current chain height |
| `3cp_pending_entries` | Gauge | Entries awaiting anchoring |
| `3cp_anchor_latency_seconds` | Histogram | Time from submit to anchor |
| `3cp_supervision_lambda1` | Gauge | Network λ₁ |
| `3cp_active_peers` | Gauge | Connected peers |
| `3cp_submission_errors_total` | Counter | Failed submissions by code |

## Current Status

**Not emitted.** `/metrics` is mounted but no series are defined (see
[divergence D5](../04-divergence/observability.md)). Until instrumentation is
added, these names are **planned/illustrative**, not observed.

## Interim Signals

The `GetHealth` fields (`block_height`, `pending_hashes`, `lambda1`,
`active_peers`, `avg_tps`) are the available substitutes — see
[Section 05 — Observability](../05-operations/observability.md).

## References

- [Section 03 — metrics](../03-gleipnir/metrics.md)
- [divergence D5](../04-divergence/observability.md)
- [Section 05 — Observability](../05-operations/observability.md)
