# Configuration Reference

> **Document Level:** B — Reference
>
> **Classification:** Cross-cutting
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `gleipnir-ipc/cmd/provenanced/main.go`
> - See also: [Section 03 — deployment](../03-gleipnir/deployment.md)

---

## Purpose

Full list of daemon configuration (flags + `IPC_*` env). No config file exists.

## Daemon Flags (`provenanced`)

| Flag | Env | Default | Required | Notes |
|------|-----|---------|----------|-------|
| `--grpc-port` | `IPC_GRPC_PORT` | 50051 | | |
| `--metrics-port` | `IPC_METRICS_PORT` | 9090 | | No series yet (D5) |
| `--uid-file` | `IPC_UID_FILE` | — | **Yes** | UID0 CBOR path |
| `--node-id` | `IPC_NODE_ID` | — | **Yes** | Node identity |
| `--peers` | `IPC_PEERS` | — | | Unused (D3) |
| `--rest-listen` | `IPC_REST_LISTEN` | :8080 | | |
| `--rest-tls-cert` | `IPC_REST_TLS_CERT` | — | | Enable REST TLS |
| `--rest-tls-key` | `IPC_REST_TLS_KEY` | — | | |
| `--rest-keys-dir` | `IPC_REST_KEYS_DIR` | — | | REST auth keys |
| `--rest-allowed-roots` | `IPC_REST_ALLOWED_ROOTS` | — | | Whitelist |
| `--rest-rate-limit` | `IPC_REST_RATE_LIMIT` | 5000 | | Per-minute |

## Engine / State Config (code defaults)

| Key | Default | Spec | Source |
|-----|---------|------|--------|
| `Eta` | 0.28 | 0.28 | `pkg/state/config.go` |
| `DecayRate` | 0.05 | 0.05 | `pkg/state/config.go` |
| `MinLambda1` | 0.10 | 0.10 | `pkg/state/config.go` |
| `LambdaInterval` | 10 | **0** | `pkg/state/config.go` (D7) |
| `SMTDepth` | 256 | 256 | `pkg/state/config.go` |
| `cycleInterval` | 3s (hardcoded) | — | `server.NewServer` |

## Notes

- `cycleInterval` is **not** configurable via flag/env.
- `LambdaInterval` diverges from spec (D7) — see
  [divergence](../04-divergence/state-machine.md).

## References

- [Section 03 — deployment](../03-gleipnir/deployment.md)
- [divergence D7](../04-divergence/state-machine.md)
