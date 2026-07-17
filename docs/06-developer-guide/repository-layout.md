# Repository Layout

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/` (Go), `carcosa/` (Rust)
> - See also: [Section 03 — architecture](../03-gleipnir/architecture.md)

---

## Purpose

Map the two reference repositories so contributors can navigate the code.

## Scope

Top-level layout of Gleipnir (Go) and Carcosa (Rust). Build/test commands are
in [Section 03 — deployment](../03-gleipnir/deployment.md) and
[Section 04 — Carcosa](../04-carcosa/architecture.md).

## Gleipnir (`gleipnir-ipc/`)

| Path | Role |
|------|------|
| `cmd/provenanced/` | Daemon entrypoint (flags, wiring) |
| `cmd/provectl/` | CLI (init, submit, verify) |
| `cmd/pipeline-sim/` | Load generator |
| `cmd/conformance-test/` | Conformance suite |
| `pkg/consensus/` | `Engine`, `RunCycle`, storage iface |
| `pkg/state/` | `NetworkState`, λ₁, supervision root |
| `pkg/smt/` | Sparse Merkle tree |
| `pkg/identity/` | VRF, Dilithium3, UID |
| `pkg/validation/` | Submission validation |
| `pkg/storage/` | BoltDB backend |
| `pkg/server/` | gRPC service |
| `pkg/rest/` | REST service |
| `deploy/` | `prometheus.yml` only |
| `docs/` | Internal specs |

## Carcosa (`carcosa/`)

| Path | Role |
|------|------|
| `air/` | Winterfell AIR (`approval_air.rs`) — **implemented** |
| `cli/` | Argument parsing — **implemented** |
| `prover/`, `verifier/` | Absent (planned) |
| `grpc/`, `proto/` | Stubs (planned) |
| `store/` | Print-only (planned) |
| `audit/` | Print-only (planned) |
| `test-data/`, `docker/` | Placeholders |

## Conventions

- Go: `pkg/` packages; `cmd/` binaries. Tests colocated (`*_test.go`).
- Rust: workspace crates; `BLUEPRINT.md` tracks intended structure.

## References

- [Section 03 — architecture](../03-gleipnir/architecture.md)
- [Section 04 — Carcosa](../04-carcosa/architecture.md)
