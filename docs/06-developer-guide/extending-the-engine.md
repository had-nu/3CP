# Extending the Engine

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/pkg/consensus/engine.go`, `pkg/state/apply.go`
> - See also: [Section 03 — engine](../03-gleipnir/engine.md)

---

## Purpose

Guide for contributors extending `Engine` (new cycle phases, mandates, gossip).

## Scope

Hook points inside `RunCycle` and `state.Apply`. Not a full rewrite guide.

## Hook Points

| Extension | Where |
|-----------|-------|
| New pre-proposal logic | `RunCycle` before `proposeBlock` (`engine.go`) |
| Mandate enforcement | Inside proposal step — see [divergence D2](../04-divergence/mandates.md) |
| Per-entry signature check | `proposeBlock` entry loop (currently missing — D4) |
| Custom supervision metric | `state.Apply` after λ₁ recompute |
| Persistence side-effect | `persist()` after `AppendBlock` |

## Adding a Cycle Phase

1. Implement the phase as a method on `Engine`.
2. Call it from `RunCycle` at the correct ordering point (spec §6.3):
   `collectHeartbeats → applyState → proposeBlock → commitBlock → persist`.
3. Add a unit test mirroring `pkg/consensus/engine_test.go`.

## Adding Mandates (planned)

See [divergence D2](../04-divergence/mandates.md) for the full gap and the
required `pkg/mandate` + RPCs. Wire `GetActiveMandates` into `proposeBlock` to
emit `MandateProof`.

## Wiring libp2p (planned)

See [divergence D3](../04-divergence/consensus.md). Replace the single-node
commit with proposal broadcast → vote gather → threshold commit.

## Testing

- Unit: `pkg/consensus/*_test.go`, `pkg/state/*_test.go`.
- Conformance: `cmd/conformance-test/` asserts unimplemented `StreamBlocks` and
  the auth path.
- Race: `make test-race`.

## References

- [Section 03 — engine](../03-gleipnir/engine.md)
- [divergence D2](../04-divergence/mandates.md),
  [D3](../04-divergence/consensus.md), [D4](../04-divergence/consensus.md)
