# Performance

> **Document Level:** E — Operational
>
> **Classification:** Operational
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §11 (performance considerations)
> - Implementation: `gleipnir-ipc/pkg/consensus/engine.go` (cycleInterval 3s)

---

## Purpose

Document performance characteristics and the absence of measured benchmarks.

## Scope

Throughput, latency, and resource posture. **No benchmarks are published** in
the reference implementation; figures below are architectural bounds, not
measurements.

## Latency

| Bound | Value | Notes |
|-------|-------|-------|
| Cycle floor | 3s (`cycleInterval`, hardcoded) | Max time between proposal attempts |
| Anchor latency (typical) | ~1 cycle + consensus | `WaitForAnchor` polls at 100ms |
| `WaitForAnchor` timeout | ≥ 3 × cycleInterval (9s) | Spec §11.2 guidance |

> All latency values are **design bounds**, not measured SLAs. No fabricated
> throughput numbers are stated.

## Throughput

`GetHealth` exposes `avg_tps`. Under load, throughput is bounded by:

- SMT insert cost per entry (BLAKE3, depth 256).
- `cycleInterval` gating proposal frequency.
- λ₁ power iteration every `LambdaInterval` cycles.

No published TPS figure exists; populate via load testing in your environment.

## Resource Posture

- Single goroutine cycle loop — CPU cost per cycle is modest; λ₁ recompute is
  the largest periodic cost.
- BoltDB grows with chain length; plan storage accordingly ([capacity-planning](capacity-planning.md)).

## Tuning Levers

| Lever | Effect |
|-------|--------|
| Lower `cycleInterval` | Faster anchoring (currently hardcoded) |
| `LambdaInterval` | λ₁ recompute frequency (default 10) |
| `DecayRate` / `MinLambda1` | Supervision sensitivity |

## References

- [Section 03 — scheduler](../03-gleipnir/scheduler.md)
- [Section 03 — metrics](../03-gleipnir/metrics.md)
- [divergence D7](../04-divergence/state-machine.md)
