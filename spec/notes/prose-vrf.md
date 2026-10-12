# Verifiable Random Function (ECVRF)

> **Document Level:** C — Protocol
> **Classification:** Normative
> **Status:** Draft
> **Protocol Version:** 3
> **Source:** [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §3.3, §6.2
> **Schema:** [`spec/schemas/vrf.cddl`](../schemas/vrf.cddl)

---

## Purpose

Define the ECVRF used for leader selection, including proof structure,
serialization, and verification.

## Parameters (spec §3.3)

| Parameter | Value |
|---|---|
| Suite | `ECVRF-RISTRETTO255-SHA512` (C2SP `vrf-r255`) |
| Group | Ristretto255 (prime order, cofactorless) |
| Internal hash | SHA-512 |
| Public key | 32 bytes |
| Private key | 32 bytes |
| Proof | 96 bytes (`Gamma ‖ C ‖ S`) |
| Output `Gamma` | 32 bytes |

### Nonce derivation (domain-separated)

```
k_string = Hash(suite_string || 0x81 || int_to_string(SK, 32) || h_string)
k        = string_to_int(k_string) mod q
```

Every `Hash` instantiation in the suite is domain-separated by a fixed prefix,
as required for a proper deterministic ECVRF.

## VRF proof structure (96 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 32 | `Gamma` | VRF output — Ristretto255 point |
| 32 | 32 | `C` | Fiat-Shamir challenge scalar |
| 64 | 32 | `S` | Schnorr response scalar |

## Operations

| Operation | Description |
|-----------|-------------|
| `VRF_KeyGen()` | → `(sk, pk)`, 32-byte scalar / point |
| `VRF_Prove(sk, alpha)` | → 96-byte proof |
| `VRF_Verify(pk, alpha, proof)` | → `gamma` or `INVALID` |

## Role in consensus

`alpha = "3CP-VRF-ALPHA-v3" || LE64(cycle) || StateRoot_at_cycle_start`.

The validator with the lexicographically smallest `Gamma` is the proposer for
that cycle; ties break by smallest `ValidatorID`. A proof that fails
verification excludes its producer from selection for that cycle.

## Security

| Property | Basis |
|----------|-------|
| Verifiability | Any peer verifies a proof against the sender's 32-byte VRF public key |
| Determinism / uniqueness | RFC 9381 construction: a unique output per (key, input) |
| Grinding resistance | Output is pseudorandom and bound to the key; a validator cannot bias its `Gamma` for a fixed `alpha` without its key |

> **Correction from v2.0.** v2.0 documented a "2-point Elligator2 sum"
> hash-to-curve and an "HMAC-SHA512(sk) nonce" that match no published standard,
> and inconsistently named the suite `ECVRF-EDWARDS25519-SHA512-Elligator2`
> (an edwards25519 suite) while targeting Ristretto255. RFC 9380 §8.5 explicitly
> forbids the edwards25519 suites for hashing to ristretto255. v3.0 adopts the
> published `ECVRF-RISTRETTO255-SHA512` suite and its domain-separated nonce;
> the divergence is eliminated, not merely documented. Byte-exact vectors ship
> in [`spec/examples/vectors/`](../examples/vectors/).

## Failure modes

| Failure | Error | Handling |
|---------|-------|----------|
| Malformed proof bytes | `ErrVRFInvalidInput` | Reject; exclude peer |
| Verification failure | `ErrVRFVerifyFailed` | Exclude peer from selection this cycle |
| No valid proposer | `ErrVRFSelectionFailed` | New VRF round next cycle |

## References

- [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §3.3, §6.2
- [`spec/schemas/vrf.cddl`](../schemas/vrf.cddl)
- [RFC 9381](https://www.rfc-editor.org/rfc/rfc9381.html) — Verifiable Random Functions
- [RFC 9380](https://www.rfc-editor.org/rfc/rfc9380.html) — Hashing to Elliptic Curves
- [cryptography](prose-cryptography.md), [consensus](prose-consensus.md)