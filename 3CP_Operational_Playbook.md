# 3CP / Gleipnir Operational Playbook (Production-Oriented)

> **Status:** Draft reconstructed from the 3CP specification and the Gleipnir reference implementation structure.

## Purpose

This playbook replaces the tutorial-style "happy path" with an operational flow that more closely matches a production deployment.

## Boot Sequence

```text
Process start
 ├─ Parse configuration
 ├─ Load uID0 identity
 ├─ Open persistent storage
 ├─ Recover chain height
 ├─ Rebuild Sparse Merkle Tree
 ├─ Initialize Engine
 ├─ Start gRPC endpoint
 ├─ Start REST endpoint
 ├─ Start Prometheus metrics
 ├─ Discover peers
 ├─ Synchronize missing blocks
 ├─ Wait for quorum
 └─ Enter consensus
```

### Expected boot log

```text
12:00:00 loading identity...
12:00:00 opening state backend...
12:00:00 rebuilding SMT...
12:00:00 recovered height=248
12:00:00 grpc listening :50051
12:00:00 rest listening :8080
12:00:00 metrics listening :9090
12:00:00 waiting for quorum (1/5)
```

## Consensus Lifecycle

1. Peer discovery
2. State synchronization
3. Quorum formation
4. Leader election (VRF)
5. Proposal dissemination
6. Proposal validation
7. SMT update
8. Dilithium signing
9. Vote collection
10. Commit
11. Persist state

### Realistic delays

| Stage | Expected bottleneck |
|---|---|
| Identity loading | Disk I/O |
| SMT rebuild | CPU + storage |
| Peer sync | Network |
| Leader election | Network latency |
| Dilithium signing | CPU |
| Commit | Storage flush |

## Idle Behaviour

Heartbeats continue while no client requests are received.

```text
heartbeat
heartbeat
heartbeat
```

## Client Request

```text
SubmitHash
↓
Queue
↓
Batch window
↓
Build SMT
↓
Consensus
↓
Commit
↓
Anchor available
```

## Failure Scenarios

### Leader timeout

```text
leader timeout
starting new VRF round
new leader elected
pending proposal retransmitted
```

### Lost peer

```text
peer disconnected
quorum maintained (4/5)
consensus continues
```

### Node restart

```text
recover persistent state
rebuild SMT
sync missing blocks
rejoin cluster
```

## Operational Notes

The original playbook is suitable for onboarding but omits synchronization, batching, recovery, leader rotation, and realistic pauses. This version is intended as the basis for a production runbook and should be refined against the full Gleipnir source code module-by-module.
