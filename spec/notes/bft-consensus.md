# BFT Consensus in 3CP v3.0

> **Protocol Version:** 3
> **Source:** [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §6

## Why Two Phases?

3CP v1.0 defined leader election via VRF and M-of-N co-signatures, but not the
exact moment a block becomes final — a safety violation under Byzantine faults.
v3.0 uses an atomic two-phase consensus:

1. **PREPARE:** Validators verify the proposed block and sign its hash `H(B)`.
2. **COMMIT:** The leader collects `Q = ceil(2N/3)` PREPARE signatures,
   constructs the final block, and signs it.

A block is **final** only after both phases complete with valid quorum.

> **Correction from v2.0.** v2.0 defined `CommitSig = Sign(H(B_final))` where
> `B_final` contained the `CommitSig` — a self-referential definition. v3.0
> removes the circularity: PREPARE covers `H(B)` (keys 0–12), `CommitSig` covers
> `B_final` excluding key 15 (spec §5.6).

## Reserved Key 8 and Optional Members

Key 8 is reserved and MUST be omitted (or `null`); PREPARE signatures live
exclusively in keys 13/14. `Triad` (key 4) and key 8 are **optional** members.
`Metadata` (key 19) is the declared home of mode flags such as
`3cp:degraded-block`.

## Quorum bitmap bit order

`PrepareSigsBitmap` is **MSB-first within each byte**; bit `i` at byte
`floor(i/8)`, position `7 - (i mod 8)`. Bits with index `>= N` MUST be zero.
`PrepareSigsPayload` length MUST equal `popcount(bitmap)`.

## Tolerance

With `Q = ceil(2N/3)`, the protocol tolerates `f < N/3` Byzantine validators.
No fork is possible without `f >= N/3` colluding validators.

## Degraded mode (non-BFT)

When `N < 4`, `Q = 1`. This is a single-signature trust configuration; it does
**not** provide Byzantine fault tolerance and MUST NOT be relied on for
non-repudiation against a single compromised key (spec §6.5).

## State Machine

```
CYCLE_START → PREPARING → PREPARED → COMMITTING → COMMITTED → CYCLE_END
                  |
                  +-- Timeout → CYCLE_ABORTED (entries retained)
```

## Slashing Evidence

A leader that double-proposes for a cycle produces evidence: two distinct blocks
for the same `Index`, each with a valid leader `CommitSig`. Any validator may
submit this pair with `Label = "3cp:slash-evidence:v1"`.
