# Security Operations

> **Document Level:** E — Operational
>
> **Classification:** Operational
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §11.4, §7.2.1
> - Implementation: `gleipnir-ipc/cmd/provenanced/main.go`, `pkg/server/server.go`, `pkg/rest/server.go`
> - Threat model: [Section 01 — Threat Model](../01-introduction/threat-model.md)

---

## Purpose

Operational security controls for running Gleipnir: identity custody, transport
security, and exposure hardening.

## Scope

Runtime security of the daemon and its data. Protocol threat model is in
[Section 01 — Threat Model](../01-introduction/threat-model.md).

## Identity Custody

- UID0 CBOR files are **root credentials**. Store on encrypted volumes; restrict
  the mount to the daemon only.
- The bootstrap service (`provectl init --validators 5`) generates UID0s — run it
  in a trusted environment, never persist keys in image layers.

## Transport Security

| Surface | Control |
|---------|---------|
| gRPC (50051) | Require mTLS at the network boundary; the daemon does not terminate mTLS itself |
| REST (8080) | Enable `--rest-tls-cert`/`--rest-tls-key`; daemon warns on plaintext |
| `/metrics` (9090) | Keep on a private network; no auth |

## Authentication

- gRPC: per-request Dilithium3 signature over `hash||submitter||LE64(ts)||label`.
- REST: `X-UID0-RootID` + `X-Signature` over `body||METHOD||PATH||ts`, 30s skew.

## Exposure Hardening

1. Front REST with an mTLS reverse proxy; do not expose 8080 publicly.
2. Restrict 50051 to the validator subnet.
3. Set `--rest-allowed-roots` to a whitelist.
4. Apply `--rest-rate-limit` to bound abuse.

## Key Compromise

Signing-key compromise is a custody problem, not a protocol failure (spec §11.4,
A3). Revoke the UID at the application layer; Gleipnir has no on-chain revocation
primitive yet.

## References

- [Section 03 — grpc-api](../03-gleipnir/grpc-api.md),
  [rest-api](../03-gleipnir/rest-api.md)
- [Section 01 — Threat Model](../01-introduction/threat-model.md)
- [incident-response](incident-response.md)
