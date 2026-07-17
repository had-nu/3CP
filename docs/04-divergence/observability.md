# Divergence — Observability / Metrics (D5)

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §11.3 (metrics)
> - Implementation: `gleipnir-ipc/cmd/provenanced/main.go`, `pkg/consensus/engine.go`

---

## Spec Requirement

The daemon SHOULD expose Prometheus metrics: `3cp_block_height`,
`3cp_pending_entries`, `3cp_anchor_latency_seconds`,
`3cp_supervision_lambda1`, `3cp_active_peers`, `3cp_submission_errors_total`,
etc. (spec §11.3).

## As-Built

`/metrics` is mounted (`promhttp.Handler()`), but **no application metric is
defined or emitted** — a full-source review found no `prometheus`/`promauto`
instrumentation. Scrape returns only Go runtime collectors.

The only observable node signals are `GetHealth` values (`block_height`,
`pending_hashes`, `lambda1`, `active_peers`, `avg_tps`, `status`) via gRPC/REST.

## Impact

| Dimension | Consequence |
|-----------|-------------|
| Dashboards | Cannot build spec-native Grafana panels |
| Alerting | No latency/error counters to alert on |
| Spec conformance | Fails §11.3 |

**Severity: Major.** Operators must poll `GetHealth` instead of scraping
Prometheus. See [Section 03 — metrics](../03-gleipnir/metrics.md).

## Remediation

Add `promauto` counters/gauges/histograms in `pkg/consensus` and `pkg/server`,
mirroring the §11.3 registry. Until then, deploy a sidecar that polls
`GetHealth` and exports the values.

## References

- [Section 03 — metrics](../03-gleipnir/metrics.md)
- [Section 05 — Observability](../05-operations/observability.md)
- [overview](overview.md) (D5)
