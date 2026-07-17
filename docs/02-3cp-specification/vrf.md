# Verifiable Random Function (ECVRF)

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
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §3.5, §6.3.1
> - Implementation: `gleipnir-ipc/pkg/identity/vrf.go`
> - Tests: `pkg/identity/identity_test.go`, `identity_property_test.go`
> - Schema: [`spec/schemas/vrf.cddl`](../../spec/schemas/vrf.cddl)

---

## Purpose

Define the ECVRF used for proposer selection, including proof structure,
serialization, and verification.

## Scope

The ECVRF primitive (spec §3.5) and its role in proposer selection (spec §6.3.1;
see [consensus](consensus.md)). This document also records a divergence between
the reference implementation and RFC 9381.

## Parameters (spec §3.5)

| Parameter | Value |
|-----------|-------|
| Curve | Ristretto255 |
| Suite string | `ristretto255_XMD:SHA-512_R255MAP_RO_` |
| Private key | 32 bytes (ristretto scalar) |
| Public key | 32 bytes (ristretto point) |
| Hash-to-curve | `expand_message_xmd(SHA-512, 64 bytes)` → two 32-byte halves → Elligator2 map → sum p0 + p1 |

## VRFProof Structure (96 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 32 | Gamma | VRF output — Ristretto255 point (hash-to-curve result) |
| 32 | 32 | C | Fiat-Shamir challenge — scalar bytes |
| 64 | 32 | S | Schnorr response — scalar bytes |

Serialization: `Gamma || C || S` (96 bytes). Implementation:
`MarshalVRFProof` / `UnmarshalVRFProof` in `pkg/identity/vrf.go`.

## Operations

| Operation | Signature | Description |
|-----------|-----------|-------------|
| `GenerateVRFKeyPair` | → (sk, pk) | 32-byte scalar / point |
| `Prove(sk, alpha)` | → proof | Schnorr-style over `HashToCurve(alpha)` |
| `Verify(pk, alpha, proof)` | → bool | MUST verify every received proof (spec §3.5) |

## Role in Consensus

`alpha = LE64(cycle) || stateRoot` (40 bytes). The peer with the lowest `Gamma`
in lexicographic byte order is the proposer for that cycle (spec §6.3.1). Any
proof failing verification excludes its producer from selection that cycle. See
[consensus](consensus.md) §1.

## Security

| Property | Basis |
|----------|-------|
| Grinding resistance | ECVRF output is pseudorandom and bound to the key; a proposer cannot bias `Gamma` without changing keys |
| Verifiability | Any peer verifies a proof against the sender's 32-byte VRF public key |
| Uniqueness | RFC 9381 VRFs bind a unique output to each (key, input) pair |

## Implementation Divergence (RFC 9381)

> **Behavior not fully conformant to RFC 9381.**
>
> `pkg/identity/vrf.go` `HashToCurve` uses a custom two-point Elligator2 sum
> (expand to 64 bytes via XMD-SHA-512, split into two 32-byte halves, Elligator2
> map each, and sum) rather than the literal RFC 9381 `hash_to_ristretto255`
> procedure. Nonce derivation uses HMAC-SHA-512 and challenge/response are
> Schnorr-style.
>
> Consequences:
> - The construction is internally consistent (prove/verify round-trip within
>   Gleipnir) and grinding-resistant for proposer selection.
> - Byte-level interoperability with a strictly RFC-9381-conformant ECVRF is
>   **not guaranteed**. A second implementation MUST replicate Gleipnir's
>   hash-to-curve to interoperate, or both MUST migrate to the literal RFC 9381
>   procedure.
> - This is a known gap to resolve before claiming RFC 9381 conformance.

## Failure Modes

| Failure | Error | Handling |
|---------|-------|----------|
| Malformed proof bytes | `ErrVRFInvalidInput` | Reject; exclude peer |
| Verification failure | `ErrVRFVerifyFailed` | Exclude peer from selection this cycle |
| No valid proposer | `ErrVRFSelectionFailed` | New VRF round |

## References

- [`spec/3CP.md`](../../spec/3CP.md) §3.5, §6.3.1
- [`spec/schemas/vrf.cddl`](../../spec/schemas/vrf.cddl)
- [RFC 9381](https://tools.ietf.org/html/rfc9381) — Verifiable Random Functions
- [consensus](consensus.md), [cryptography](cryptography.md)
