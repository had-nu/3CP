# Observability

> **Document Level:** E — Operational
>
> **Classification:** Operational
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §11.3
> - Implementation: [Section 03 — metrics](../03-gleipnir/metrics.md)

---

## Purpose

How to monitor a Gleipnir deployment given the current instrumentation gap.

## Scope

Health polling, logging, and dashboards. Prometheus metrics are not yet emitted
(see [divergence D5](../04-divergence/observability.md)).

## Health Polling (primary signal)

Poll `GetHealth` (gRPC/REST) on a 5–15s interval and export:

| Field | Alert when |
|-------|-----------|
| `block_height` | Stalls (no increment over N intervals) |
| `pending_hashes` | Sustained growth |
| `lambda1` | `< MinLambda1` (0.10) → fragmentation |
| `active_peers` | Drops below expected (currently always 0 — single node) |
| `avg_tps` | Unexpected 0 under load |

## Recommended Exporter

Until native instrumentation lands, run a sidecar that polls `GetHealth` and
exposes the values as Prometheus gauges. Example scrape target mirrors
`deploy/prometheus.yml`.

## Logging

The daemon logs commit cycles (block index) and startup/shutdown. Ship logs to
your aggregation system; alert on absence of commit-cycle lines.

## Dashboards

Build panels from the `GetHealth` fields above. Do **not** depend on
`/metrics` 3CP series yet.

## References

- [Section 03 — metrics](../03-gleipnir/metrics.md)
- [divergence D5](../04-divergence/observability.md)
- [Section 07 — Metrics Reference](../07-reference/metrics-reference.md)
