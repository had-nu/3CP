# BFT Consensus in 3CP v2.0

## Why Two Phases?

3CP v1.0 defined leader election via VRF and M-of-N co-signatures, but did not
specify the exact moment a block becomes final. This allowed implementations to
append blocks locally before quorum verification was complete — a safety
violation under Byzantine faults.

v2.0 introduces an atomic two-phase consensus:

1. **PREPARE:** Validators verify the proposed block and sign its hash.
2. **COMMIT:** The leader collects `Q = ceil(2N/3)` PREPARE signatures,
   constructs the final block, and collects COMMIT signatures.

A block is **final** only after both phases complete with valid quorum.

## Tolerance

With `Q = ceil(2N/3)`, the protocol tolerates `f < N/3` Byzantine validators.
This is the classic BFT bound. No fork is possible without `f >= N/3`
colluding validators, and even then safety holds (they cannot forge a valid
quorum for conflicting blocks).

## State Machine

```
CYCLE_START
    |
    v
PREPARING  (VRF election + proposal + PREPARE sig collection)
    |
    +---> Timeout? ---> CYCLE_ABORTED (entries retained)
    |
    v
PREPARED   (Q PREPARE sigs collected)
    |
    v
COMMITTING (B_final broadcast + COMMIT sig collection)
    |
    v
COMMITTED  (block appended, state transitioned)
    |
    v
CYCLE_END
```

## Slashing Evidence

A Byzantine leader that double-proposes for the same cycle produces
cryptographic evidence: two distinct blocks with valid VRF proofs and valid
leader signatures. Any validator can submit this evidence as a
`ProvenanceEntry` with label `3cp:slash-evidence:v1`.