# Divergence — State Machine & Config (D7)

> **Document Level:** B — Gap Analysis
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §6.4 (cycle params)
> - Implementation: `gleipnir-ipc/pkg/state/config.go`

---

## D7 — LambdaInterval default (Minor)

**Spec:** `LambdaInterval` (λ₁ recompute cadence) default is **0** — recompute
every cycle for tightest supervision accuracy.

**As-built:** `pkg/state/config.go` sets `LambdaInterval = 10`. λ₁ is recomputed
every 10 cycles (or when `λ₁ == 0`).

**Impact:** Fragmentation detection lags up to 10 cycles; minor supervision
fidelity loss. No safety break, but diverges from the spec's "recompute
frequently" guidance.

## Other Config Notes

- `MinLambda1 = 0.10`, `DecayRate = 0.05`, `Eta = 0.28`, `SMTDepth = 256` — all
  match the spec's stated defaults.
- `cycleInterval = 3s` is hardcoded in `server.NewServer` (not a config field).

## Remediation

Lower `LambdaInterval` to 0 (or make it configurable) to match spec §6.4; expose
`cycleInterval` as a flag/env.

## References

- [Section 03 — state-machine](../03-gleipnir/state-machine.md)
- [overview](overview.md) (D7)
