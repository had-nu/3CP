# Migration Guide: 3CP v1.0 / v2.0 → v3.0

> **Protocol Version:** 3
> **Source:** [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §15, Appendix C, Appendix D

## Overview

3CP v3.0 is a **hard fork**. The `BlockHash` construction, signature naming,
canonical CBOR float rule, VRF suite, and SMT preimages all changed. There is
no in-place upgrade path.

| Area | v1.0 / v2.0 | v3.0 |
|---|---|---|
| Signature name | "Dilithium3" | **ML-DSA-65** (FIPS 204) |
| Signature size | 2,700 B (v1/v2, corrected to 3,309 upstream in #1) | **3,309 B** |
| CBOR floats | fixed 64-bit | **shortest round-trip** (RFC 8949 §4.2.1) |
| VRF suite | mislabelled / bespoke | **ECVRF-RISTRETTO255-SHA512** |
| VRF nonce | HMAC-SHA512(sk) | **domain-separated `vrf-r255`** |
| `BlockHash` | no domain tag, no `NetworkID` | **`3CP-BLOCKHASH-v3` + `NetworkID`** |
| SMT preimages | `BLAKE3("leaf"‖…)`, no tags | **`3CP-SMT-LEAF-v3` / `3CP-SMT-NODE-v3`** |
| Key 8 | version-dependent | **reserved; omitted or null** |
| `CommitSig` | self-referential | **covers `B_final` minus key 15** |

## Pre-migration

1. Halt all v1.0/v2.0 validators at the same final block `B_final`.
2. Export: `ValidatorSet` (ValidatorIDs, ML-DSA-65 public keys, VRF public
   keys), active `MandateEntry` objects, final SMT root, and `H(B_final)`.

## Genesis construction (v3.0)

1. Build a v3.0 genesis block with `Index: 0`, `PrevHash: 0x00…00`,
   `ProtocolVersion: 3`, `Anchored: [GenesisMandate]` (mandates converted to
   v3.0 form), `Validators: [validator-info …]`, `LegacyAnchor: H(B_final)`.
2. Re-sign all genesis structures under the v3.0 rules (ML-DSA-65 at 3,309 B;
   `BlockHash` recomputed with the domain tag and `NetworkID` binding).
3. Compute `NetworkID` and `BlockHash` in the normative order (spec §11.1):
   1. Build the genesis block with key 11 (`BlockHash`) set to 32 zero bytes.
   2. `NetworkID = BLAKE3-256(canonical CBOR(genesis block with key 11 omitted))`.
   3. `BlockHash = SHA-256` over the §5.5 preimage using that `NetworkID`.
   4. Store the computed `BlockHash` at key 11.
4. Distribute the genesis block to all v3.0 validators.

> **Note.** `NetworkID` depends on the genesis block and the genesis `BlockHash`
> depends on `NetworkID`, so the ordering above is normative: `NetworkID` is
> computed with key 11 omitted, which removes the circularity.

## Post-migration

- v1.0/v2.0 blocks remain readable via archive nodes for historical audit.
- v3.0 validators do not process v1.0/v2.0 blocks.
- `LegacyAnchor` lets auditors link the v3.0 genesis to the prior chain.

## References

- [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §15, Appendix C, Appendix D
