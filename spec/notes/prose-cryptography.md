# 3CP Cryptography

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
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §3
> - Implementation: `gleipnir-ipc/pkg/identity/` (`dilithium.go`, `kyber.go`, `vrf.go`, `blake3.go`), `pkg/consensus/engine.go`
> - Tests: `pkg/identity/*_test.go`, `pkg/identity/kyber_audit_test.go`

---

Per `DOCUMENTATION_SPEC.md` §13, each cryptographic component below documents:
algorithm, security assumptions, key lifecycle, inputs, outputs, failure
conditions, complexity, and interoperability notes.

## Purpose

Define the fixed cryptographic suite of 3CP v1 and its usage.

## Scope

The v1 suite: BLAKE3-256, SHA-256, Dilithium3 (ML-DSA-65), Kyber1024
(ML-KEM-1024), ECVRF (Ristretto255), and canonical CBOR. ECVRF details are in
[vrf](vrf.md); CBOR details in [protobuf](protobuf.md).

## Suite (v1, fixed)

The v1 suite is fixed (spec §12.4): **Dilithium3, Kyber1024, Ristretto255
ECVRF, BLAKE3-256, SHA-256.** Implementations MUST negotiate the suite at
connection establishment; future versions MAY introduce new suites.

## 1. Hash Functions

### 1.1 BLAKE3-256

- **Use:** all general hashing (SMT nodes, supervision root, identity digest,
  contract binding) unless otherwise noted (spec §3.1).
- **Output:** 32 bytes.
- **Implementation:** `pkg/identity/blake3.go` (`Hash`, `Derive`); SMT in
  `pkg/smt/smt.go`; supervision root in `pkg/state/root.go`.
- **Domain separation:** SMT leaves use prefix `"leaf"` (spec §5.1;
  `pkg/smt/smt.go`).

### 1.2 SHA-256

- **Use:** the block identifier `BlockHash` only (spec §3.2).
- **Preimage (fixed-length binary):**

```
BlockHash = SHA-256(
    LE64(Index)       // 8 bytes little-endian
  || PrevHash         // 32 bytes BLAKE3-256 of previous block (zeros for genesis)
  || StateRoot        // 32 bytes SMT root
  || Proposer         // 16 bytes RootID
  || anchored[0].Hash // 32 bytes, first entry hash
  || ...              // remaining entry hashes, each 32 bytes, in Anchored order
  || LE64(Timestamp)  // 8 bytes UnixNano little-endian
)
```

- A conformant implementation MUST produce identical `BlockHash` for identical
  block content. Entry hashes MUST appear in `Anchored` array order (spec §3.2).
- **Implementation:** `computeBlockHash` in `pkg/consensus/engine.go`.

## 2. Post-Quantum Signatures — Dilithium3 (ML-DSA-65)

| Parameter | Value |
|-----------|-------|
| Algorithm | ML-DSA-65 (Dilithium3) |
| Public key | 1952 bytes |
| Signature | 2700 bytes |
| Security level | NIST Level 3 (AES-256 equivalent) |

- **Use:** client authentication (`SubmitHash`), block proposer/validator
  co-signatures, optional per-entry non-repudiation.
- **Key lifecycle:** generated within UID0 (spec §8); private keys MUST NOT be
  exported or transmitted (spec §8.2, keys 16/19). `WipeSecret` zeroes secret
  material in `pkg/identity/dilithium.go`.
- **Implementation:** `pkg/identity/dilithium.go`
  (`GenerateDilithiumKey`, `SignDilithium`, `VerifyDilithium`) over
  `circl/sign/dilithium/mode3`.
- **Failure conditions:** signature verification failure → `INVALID_SIGNATURE`
  (spec §7.2).

## 3. Key Encapsulation — Kyber1024 (ML-KEM-1024)

| Parameter | Value |
|-----------|-------|
| Algorithm | ML-KEM-1024 (Kyber1024) |
| Ciphertext | 1568 bytes |
| Shared secret | 32 bytes |
| Security level | NIST Level 5 |

- **Use:** peer-to-peer encrypted channels (spec §3.4).
- **Implementation:** `pkg/identity/kyber.go`
  (`KyberGenerateKey`, `KyberEncapsulate`, `KyberDecapsulate`) over
  `circl/kem/schemes`; consumed by `pkg/transport/secure_conn.go` for the KEM
  handshake, which derives a ChaCha20-Poly1305 AEAD key via HKDF-SHA256.

## 4. Verifiable Random Function — ECVRF (RFC 9381)

Summarised here; full detail in [vrf](vrf.md).

| Parameter | Value |
|-----------|-------|
| Curve | Ristretto255 |
| Suite string | `ristretto255_XMD:SHA-512_R255MAP_RO_` |
| Proof size | 96 bytes (`Gamma \|\| C \|\| S`) |

- **Use:** proposer selection — lowest `Gamma` (lexicographic) wins (spec §3.5,
  §6.3.1).
- **Implementation:** `pkg/identity/vrf.go`.

> **Divergence:** the implementation's `HashToCurve` uses a custom two-point
> Elligator2 sum rather than the literal RFC 9381 `hash_to_ristretto255`. See
> [vrf](vrf.md) for the full analysis. Interoperability with a strictly
> RFC-9381-conformant VRF is **not guaranteed**.

## 5. Canonical CBOR

All CBOR structures use deterministic encoding (spec §3.6):

- Integer map keys in ascending order
- Smallest integer representation (`uint` ≤ 23 in one byte)
- Definite-length strings and byte strings
- 64-bit IEEE 754 floats

Implementation: `pkg/identity/cbor.go` (`CanonicalEncOptions().EncMode()`).

## Complexity and Performance

| Operation | Dominant cost | Notes |
|-----------|---------------|-------|
| Dilithium3 sign/verify | CPU | ~2700-byte signatures; hot path during co-signing |
| Kyber1024 encaps/decaps | CPU | Handshake only |
| VRF prove/verify | CPU | Per cycle, per peer |
| BLAKE3 | CPU/memory | SMT insert/verify (256-deep path) |

Concrete measurements are not asserted here; see
[Section 05 — Performance](../05-operations/performance.md). Any figure
presented elsewhere as measured MUST be labelled per `DOCUMENTATION_SPEC.md` §21.

## Security Considerations

- **Assets:** unforgeability of attribution (Dilithium3), confidentiality of
  peer channels (Kyber1024), grinding resistance of proposer selection (ECVRF),
  collision resistance of state (BLAKE3-256).
- **Assumption failure:** if NIST Level 3 assumptions break, non-repudiation is
  void; if BLAKE3 collision resistance breaks, SMT integrity is void.
- **Custody:** all guarantees are void if signing keys are co-located with the
  credentials they should constrain (spec §11.4).

## References

- [`spec/3CP.md`](../../spec/3CP.md) §3, §12.4
- [vrf](vrf.md), [sparse-merkle-tree](sparse-merkle-tree.md),
  [protobuf](protobuf.md)
- [FIPS 205](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.205.pdf) (ML-DSA),
  [FIPS 203](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.203.pdf) (ML-KEM),
  [RFC 9381](https://tools.ietf.org/html/rfc9381) (VRF),
  [BLAKE3](https://github.com/BLAKE3-team/BLAKE3)
