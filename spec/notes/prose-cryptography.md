# 3CP Cryptography

> **Document Level:** C — Protocol
> **Classification:** Normative
> **Status:** Draft
> **Protocol Version:** 3
> **Source:** [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §3, §4

---

## Purpose

Define the cryptographic suite of 3CP v3.0 and its usage.

## Scope

The v3.0 suite: BLAKE3-256, SHA-256, ML-DSA-65 (FIPS 204), ML-KEM-1024
(FIPS 203), ECVRF-RISTRETTO255-SHA512, ChaCha20-Poly1305 (RFC 8439),
HKDF-SHA256 (RFC 5869), and canonical CBOR (RFC 8949).

## Suite (v3.0, fixed)

**ML-DSA-65, ML-KEM-1024, ECVRF-RISTRETTO255-SHA512, BLAKE3-256, SHA-256,
ChaCha20-Poly1305, HKDF-SHA256.**

> **Correction from v2.0.** v2.0 specified a 2,700-byte "ML-DSA-65" signature
> while citing FIPS 204. 2,700 bytes is the *round-3 draft* ML-DSA-65 value;
> FIPS 204 Table 2 fixes ML-DSA-65 at **3,309 bytes**. Upstream PR #1 already
> corrected the sizes in the v2 files; v3.0 additionally standardises the name
> to **ML-DSA-65** everywhere FIPS 204 is cited, because the signature size is
> baked into every CDDL `.size` constraint and the old value would have caused
> every conformant verifier to reject every valid signature.

## 1. Hash Functions

### 1.1 BLAKE3-256

- **Use:** entry hashes, SMT leaves/nodes, state root, `NetworkID`.
- **Output:** 32 bytes.
- **Domain separation:** SMT preimages carry explicit tags — leaf
  `BLAKE3-256("3CP-SMT-LEAF-v3" ‖ key ‖ value)`, node
  `BLAKE3-256("3CP-SMT-NODE-v3" ‖ left ‖ right)`, empty
  `BLAKE3-256("3CP-SMT-NODE-v3" ‖ E ‖ E)` with `E = BLAKE3-256("")`.

> **Correction from v1.0/v2.0.** v1.0 used `BLAKE3("leaf" ‖ …)` and
> `BLAKE3(left ‖ right)` with no separation. The three preimages are now
> distinct by construction, removing second-preimage confusion between leaf,
> node, and empty encodings.

### 1.2 SHA-256

- **Use:** the block identifier `BlockHash` only.
- **Preimage (fixed-length, domain-separated):**

```
BlockHash = SHA-256(
    "3CP-BLOCKHASH-v3"        // 17 ASCII bytes — domain tag (v3.0)
  || LE64(Index)             // 8 bytes little-endian
  || PrevHash                // 32 bytes
  || StateRoot               // 32 bytes
  || Proposer                // 16 bytes
  || HashOfAnchoredEntries   // 32 bytes, BLAKE3-256 of canonical CBOR of key 5
  || LE64(Timestamp)         // 8 bytes little-endian
  || QuorumConfigCanonical   // canonical CBOR of key 10
)
```

A conformant implementation MUST produce identical `BlockHash` for identical
block content. The `3CP-BLOCKHASH-v3` tag prevents cross-structure preimage
reuse; the `NetworkID` binding prevents cross-chain replay.

### 1.3 Canonical CBOR floats

All CBOR floats MUST use the **shortest IEEE 754 form that round-trips**
(RFC 8949 §4.2.1); NaN and Infinity are forbidden. `Lambda1`,
`SeverityMin`, and `SeverityMax` are affected. v1.0/v2.0's fixed-64-bit rule is
withdrawn — a fixed-width float produces non-canonical CBOR and therefore a
different `BlockHash`.

## 2. Digital Signatures — ML-DSA-65 (FIPS 204)

| Parameter | Value |
|---|---|
| Algorithm | ML-DSA-65 |
| Public key | 1,952 bytes |
| Private key | 4,032 bytes |
| **Signature** | **3,309 bytes** |
| Security level | NIST category 3 (AES-192 equivalent) |

- **Use:** block PREPARE and COMMIT signatures, key-rotation old/new signatures,
  mandate signatures, optional per-entry non-repudiation.
- **Key lifecycle:** private keys MUST NOT be exported or transmitted. Rotation
  uses the dual old/new signature scheme to prove continuity of authority.
- **Hedged signing:** signers SHOULD use the hedged (randomized) mode of
  FIPS 204 §3.4 rather than the deterministic variant. Verifiers accept both.

### 2.1 Failure conditions

- Signature length ≠ 3,309 bytes → reject (`ErrInvalidSignatureLength`,
  FIPS 204 §10.2).
- Signature does not verify → `INVALID_SIGNATURE`.

## 3. Key Encapsulation — ML-KEM-1024 (FIPS 203)

| Parameter | Value |
|---|---|
| Algorithm | ML-KEM-1024 |
| Public key | 1,568 bytes |
| Ciphertext | 1,568 bytes |
| Shared secret | 32 bytes |
| Security level | NIST category 5 |

- **Use:** peer-to-peer transport handshake only.
- **Implicit rejection:** decapsulation uses FIPS 203 §7.3 implicit rejection —
  a wrong ciphertext yields a pseudorandom secret, never a distinguishable
  error.

## 4. Verifiable Random Function — ECVRF-RISTRETTO255-SHA512

- **Suite:** `ECVRF-RISTRETTO255-SHA512` (C2SP `vrf-r255`), realising the
  RFC 9381 construction over the prime-order Ristretto255 group.
- **Proof:** 96 bytes (`Gamma ‖ C ‖ S`); each part 32 bytes.
- **Nonce:** `k_string = Hash(suite_string ‖ 0x81 ‖ int_to_string(SK,32) ‖ h_string)`,
  `k = string_to_int(k_string) mod q` — every `Hash` is domain-separated.

> **Correction from v2.0.** v2.0 variously described the suite as
> `ECVRF-EDWARDS25519-SHA512-Elligator2` (an edwards25519 suite, not Ristretto255)
> or as a bespoke "2-point Elligator2 sum" matching no standard, with a
> non-standard "HMAC-SHA512(sk) nonce". RFC 9380 §8.5 forbids the edwards25519
> suites for hashing to ristretto255. v3.0 adopts the published `vrf-r255`
> suite. Byte-exact vectors ship in [`spec/examples/vectors/`](../examples/vectors/).

## 5. AEAD — ChaCha20-Poly1305 (RFC 8439)

Transport encryption after the KEM handshake. The 32-byte key is derived via
HKDF-SHA256 from the ML-KEM shared secret. A (key, nonce) pair MUST NOT be
reused (RFC 8439 §12).

## 6. HKDF-SHA256 (RFC 5869)

Key derivation for UID0 seeds (`L = 32`) and AEAD keys. The `L` parameter is
always explicit; v2.0's unspecified-`L` call is withdrawn. Vector: RFC 5869
Test Case 1 in [`spec/examples/test-vectors.md`](../examples/test-vectors.md).

## Complexity and Performance

| Operation | Dominant cost | Notes |
|-----------|---------------|-------|
| ML-DSA-65 sign/verify | CPU | 3,309-byte signatures; hot path during co-signing |
| ML-KEM-1024 encaps/decaps | CPU | Handshake only |
| VRF prove/verify | CPU | Per cycle, per peer |
| BLAKE3 | CPU/memory | SMT insert/verify (256-deep path) |

## Security Considerations

- **Assets:** unforgeability of attribution (ML-DSA-65), confidentiality of peer
  channels (ML-KEM-1024 + ChaCha20-Poly1305), grinding resistance of proposer
  selection (ECVRF-RISTRETTO255-SHA512), collision resistance of state
  (BLAKE3-256).
- **Assumption failure:** if ML-DSA-65 (cat. 3) assumptions break,
  non-repudiation is void; if BLAKE3 collision resistance breaks, SMT integrity
  is void.
- **Custody:** all guarantees are void if signing keys are co-located with the
  credentials they constrain.

## References

- [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §3, §4, §17
- [vrf](prose-vrf.md), [anchoring](prose-anchoring.md)
- [FIPS 204](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf) (ML-DSA),
  [FIPS 203](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.203.pdf) (ML-KEM)
- [RFC 9380](https://www.rfc-editor.org/rfc/rfc9380.html) (hashing to curves),
  [RFC 9381](https://www.rfc-editor.org/rfc/rfc9381.html) (VRF),
  [RFC 8439](https://www.rfc-editor.org/rfc/rfc8439.html) (ChaCha20-Poly1305),
  [RFC 5869](https://www.rfc-editor.org/rfc/rfc5869.html) (HKDF),
  [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949.html) (CBOR)
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3)