# 3CP Consensus

> **Document Level:** C — Protocol
>
> **Classification:** Normative
>
> **Status:** Draft
>
> **Protocol Version:** 1
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §6
> - Implementation: `gleipnir-ipc/pkg/consensus/engine.go` (`RunCycle`), `pkg/consensus/quorum.go`, `pkg/consensus/triad.go`
> - Tests: `pkg/consensus/consensus_test.go`, `consensus_multinode_test.go`, `consensus_liveness_test.go`, `consensus_adversarial_test.go`

---

Per `DOCUMENTATION_SPEC.md` §14, this document covers node lifecycle, election,
proposal, voting, validation, commit, recovery, timeouts, partition handling,
replay protection, and safety/liveness guarantees.

## Purpose

Define the 3CP consensus cycle: one cycle produces exactly one anchored block
with instant finality.

## Scope

The consensus cycle (spec §6), single- and multi-node modes, and cycle
parameters. VRF details are in [vrf](vrf.md); SMT details in
[sparse-merkle-tree](sparse-merkle-tree.md); the state-machine view in
[protocol-state-machine](protocol-state-machine.md).

## Overview

One consensus cycle produces one anchored block with **instant finality** — no
forks, no rollbacks, no reorganization (spec §6.1). The cycle is ticker-driven.

## Single-Node Mode

`QuorumConfig = {TotalValidators: 1, RequiredSigs: 1}` (spec §6.2). The single
node acts as both proposer and signer.

## Multi-Node Mode (M-of-N)

### Control Flow

```mermaid
sequenceDiagram
    participant P as All Peers
    participant Pr as Proposer (lowest Gamma)
    participant N as Non-proposers
    P->>P: 1. VRF.Prove(alpha), gossip proof
    P->>P: 2. Collect + verify all VRF proofs
    P->>Pr: 3. Select proposer (lowest Gamma)
    Pr->>Pr: 4. Collect entries, dedup, insert into SMT
    Pr->>Pr: 5. Compute StateRoot, BlockHash; sign
    Pr->>N: 6. Broadcast block + signature
    N->>N: 7. Replay entries; verify StateRoot, BlockHash, sig
    N->>P: 8. Co-sign BlockHash; broadcast
    P->>P: 9. Quorum check (RequiredSigs distinct validators)
    P->>P: 10. State transition, append, persist
```

### 1. VRF Proposer Selection (spec §6.3.1)

- Input: `cycle` (uint64, 8 bytes LE), `stateRoot` (32 bytes), each peer's VRF
  keypair.
- `alpha = LE64(cycle) || stateRoot` (40 bytes).
- Each peer computes `VRF.Prove(sk, alpha)` → `Gamma || C || S` (96 bytes),
  publishes it, collects all proofs, and verifies each against the sender's VRF
  public key.
- **Proposer** = peer with the lowest `Gamma` in lexicographic byte order.
- Any peer whose proof fails verification MUST be excluded from selection this
  cycle.
- Implementation: alpha via `makeAlpha`, selection via `SelectProposer`
  (`pkg/consensus/engine.go`).

### 2. Block Proposal (spec §6.3.2)

The proposer:

1. Collects pending entries (gossip snapshot or local queue).
2. Deduplicates by entry `Hash` — first occurrence kept.
3. Inserts `(Hash, Hash)` into the SMT for each entry.
4. Computes `StateRoot`.
5. Computes `BlockHash` ([cryptography](cryptography.md) §1.2).
6. Signs `BlockHash` with its Dilithium3 private key.
7. Broadcasts block + signature.

### 3. Non-Proposer Verification (spec §6.3.3)

Each non-proposer replays entries in the same order, computes
`localStateRoot`, and MUST reject if `localStateRoot != block.StateRoot`. It
then recomputes `BlockHash`, verifies the proposer's signature, co-signs, and
broadcasts its co-signature. Root mismatch is verified at
`pkg/consensus/engine.go` (StateRoot equality check).

### 4. Quorum Verification (spec §6.3.4)

For each signature in `block.Sigs`, iterate `block.Validators`, skip
already-credited validators, verify against `block.BlockHash`, and credit the
first match (marking that key used). After processing, if
`usedValidatorCount < quorum.RequiredSigs`, the block MUST NOT be appended. Each
distinct validator MUST be counted at most once.

### 5. State Transition (spec §6.3.5)

1. Process heartbeat latencies (update topology).
2. Recompute Laplacian λ₁.
3. Increment cycle counter.
4. Append block.
5. Persist state.

Implementation: `state.Apply` in `pkg/state/apply.go`, invoked from `RunCycle`.

## State Machine

| State | Event | Next state | Timeout / failure |
|-------|-------|------------|-------------------|
| Idle | Ticker tick | VRFSelect | — |
| VRFSelect | Proofs collected | Propose (if self is proposer) / AwaitProposal | Missing proofs → exclude peer |
| Propose | Block built + signed | AwaitCoSign | — |
| AwaitProposal | Proposal received | Verify | Leader timeout → new VRF round |
| Verify | StateRoot + sig valid | CoSign | Mismatch → reject |
| CoSign | Co-sign broadcast | Quorum | — |
| Quorum | RequiredSigs reached | Commit | Below threshold → block dropped |
| Commit | State applied, persisted | Idle | Persist failure → recovery |

See [protocol-state-machine](protocol-state-machine.md) for the full diagram.

## Cycle Parameters (spec §6.4)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `cycleInterval` | 3s | Time between automatic cycle ticks |
| `LambdaInterval` | 0 (every cycle) | Recompute λ₁ every N cycles |
| `Eta` | 0.28 | Diffusion rate for Laplacian supervision |
| `DecayRate` | 0.05 | Node status decay per cycle |
| `MinLambda1` | 0.10 | Minimum λ₁ threshold |

> Implementation note: `pkg/state/config.go` (`DefaultConfig`) sets
> `LambdaInterval = 10` (recompute every 10 cycles), diverging from the spec's
> default of 0 (every cycle). `cycleInterval` is hardcoded to 3s in
> `server.NewServer`.

## Safety and Liveness

| Guarantee | Basis |
|-----------|-------|
| Safety (no conflicting finalized blocks) | Instant finality; one block per cycle; quorum on a single `BlockHash` |
| Liveness | Ticker-driven cycles; leader timeout triggers a new VRF round |
| Partition handling | λ₁ < `MinLambda1` (≥2 nodes) → `ErrNetworkFragmented`; nodes SHOULD fail closed |
| Replay protection | Signed payloads with timestamps; deduplication by entry hash |

## Failure Modes

| Failure | Symptom | Recovery |
|---------|---------|----------|
| Leader timeout | No proposal within window | New VRF round, new proposer, proposal retransmitted |
| Lost peer | Fewer co-signatures | Consensus continues if quorum still met |
| Persist failure | Commit incomplete | Recover persistent state on restart, rebuild SMT, re-sync |
| Network fragmentation | λ₁ below threshold | `ErrNetworkFragmented`; fail closed |

## Implementation Divergence (Gleipnir)

> - The shipped `provenanced` daemon runs **single-node only**; libp2p gossip
>   (`pkg/transport/p2p`) is not wired into `main.go`. Multi-node consensus is
>   exercised in tests, not the default daemon.
> - The explicit `VerifyQuorum` call is invoked in the multi-node test harness;
>   `RunCycle` collects signatures but the standalone quorum-count assertion is
>   test-driven (see comment in `pkg/consensus/engine.go`).
> - `LambdaInterval` default differs (10 vs spec 0) — see Cycle Parameters.

## References

- [`spec/3CP.md`](../../spec/3CP.md) §6
- [vrf](vrf.md), [sparse-merkle-tree](sparse-merkle-tree.md),
  [protocol-state-machine](protocol-state-machine.md)
- [Section 03 — Engine](../03-gleipnir/engine.md)
