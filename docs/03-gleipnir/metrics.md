# Gleipnir Metrics

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/cmd/provenanced/main.go`, `pkg/consensus/engine.go` (`GetHealth`)
> - Reference: `gleipnir-ipc/deploy/prometheus.yml`

---

## Purpose

Document the observability signals Gleipnir exposes and, critically, the gap
between the mounted `/metrics` endpoint and the metrics actually emitted.

## Scope

The `/metrics` endpoint, the health signals available via gRPC/REST, and the
Prometheus scrape configuration. Operational dashboards are covered in
[Section 05 — Observability](../05-operations/observability.md).

## `/metrics` Endpoint

`cmd/provenanced/main.go` mounts `promhttp.Handler()` at `/metrics` on the
metrics port (default `9090`, `--metrics-port` / `IPC_METRICS_PORT`).

> **Behavior not yet verified in the reference implementation.**
>
> The `/metrics` handler is mounted, but **no application metrics are defined or
> emitted**. A code review found no `prometheus`/`promauto`/`NewCounter`/
> `NewGauge`/`NewHistogram` definitions anywhere in the Go source. Scraping
> `/metrics` returns only the default Go runtime collectors registered by
> `promhttp`, not 3CP-specific counters.
>
> Any 3CP-specific metric named in
> [Section 07 — Metrics Reference](../07-reference/metrics-reference.md) is
> therefore **planned/illustrative**, not observed, until the instrumentation is
> added.

## Health Signals (available today)

The observable, node-specific signals come from `GetHealth`
(`pkg/consensus/engine.go:215`) via gRPC/REST, not Prometheus:

| Signal | Meaning |
|--------|---------|
| `block_height` | Current chain height |
| `pending_hashes` | Entries awaiting anchoring |
| `lambda1` | Network diffusion health (λ₁) |
| `active_peers` / `total_peers` | Peer counts |
| `avg_tps` | Average throughput |
| `status` | Node status (e.g. `running`) |

These are the values a dashboard or gate SHOULD poll (spec §11.3). See
[api-reference](../07-reference/api-reference.md).

## Prometheus Scrape Configuration

`deploy/prometheus.yml` targets `val-1..5:9090` at a 15s interval. This scrapes
the mounted endpoint; until instrumentation exists, dashboards should derive
health from the gRPC/REST `GetHealth` values instead.

## Recommendation

Until native metrics are implemented, operators SHOULD:

1. Poll `GetHealth` on an interval and export the values to their monitoring
   system via a sidecar/exporter, or
2. Contribute Prometheus instrumentation to `pkg/consensus` and `pkg/server`.

## References

- [Section 05 — Observability](../05-operations/observability.md)
- [Section 07 — Metrics Reference](../07-reference/metrics-reference.md)
- [grpc-api](grpc-api.md)
