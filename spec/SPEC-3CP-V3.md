# 3CP Protocol — Normative Specification v3.0

**Title:** 3CP — Cryptographic Chain-of-Custody Protocol with BFT Consensus, On-chain Governance, and Third-party Verifiability
**Status:** NORMATIVE
**Version:** 3.0.0
**Date:** 2026-10-11
**Base:** 3CP Protocol v2.0 (`SPEC-3CP-V2.md`, consolidated here), itself based on v1.0 (`spec/3CP.md`)
**Scope:** Abstract protocol specification. Implementation-language, operating-system, cryptographic-library, and execution-framework agnostic.
**Relationship to implementations:** The 3CP protocol is implementable by any party. Known reference implementations — Gleipnir (Go) and CARCOSA (Rust) — live in separate repositories and are **not part of this specification**. This standard is intended to be sufficient for an independent implementation, without access to reference source code, to achieve full interoperability.

> **Note (consolidation):** This v3.0 document supersedes and replaces three divergent drafts that previously co-existed (`spec/SPEC-3CP-V2.md`, `spec/3CP-v2.md`, `internal/SPEC-3CP-V2.md`). Those drafts disagreed on CBOR reserved-key handling and on the VRF construction. v3.0 resolves each disagreement in favour of the cryptographic canon; see [Appendix C — v2.0 → v3.0 Change Log](#appendix-c--v20--v30-change-log) and [Appendix D — Corrections Applied](#appendix-d--corrections-applied-in-v30).

---

## 1. Scope, Principles, and Normative Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 ([RFC 2119], [RFC 8174]) when, and only when, they appear in all capitals, as shown here.

### 1.1 Guiding Principles

1. **Accountability without good faith:** Conformance cannot depend on the good faith of the audited entity.
2. **Independent verifiability:** Any party holding the public chain can verify integrity, quorum, and genealogy without trusting the network operator.
3. **Finality per cycle:** One consensus cycle produces zero or one final block. Final blocks are immutable and irreversible.
4. **On-chain governance:** Network operating rules (quorum, latency, publication) are declared in Mandates anchored in the chain itself.
5. **Cryptographic canon:** Every primitive is a published standard used as specified. Where 3CP historically diverged from a standard, v3.0 either adopts the standard or explicitly, narrowly documents a divergence with a byte-exact procedure and test vectors. 3CP does not claim conformance to a standard it does not meet.

---

## 2. System Model and Entities

### 2.1 Entities

- **Submitter:** Entity submitting a `ProvenanceEntry` for anchoring. Need not be a validator.
- **Validator:** Entity participating in consensus, producing and co-signing blocks. Each validator holds an ML-DSA-65 key pair and a VRF key pair.
- **Anchor Publisher:** Entity (validator or external service) publishing final blocks to publicly-readable storage.
- **Light Client:** Entity verifying blocks and SMT proofs without participating in consensus and without maintaining full state.
- **Auditor:** Entity verifying Mandate conformance against the anchored entry chain.

### 2.2 Threat Model

- **A1 — Network adversary:** Observes, delays, drops, or reorders messages. Does not control validator keys.
- **A2 — Compromised operator:** Controls infrastructure of one or more nodes, but less than quorum (`< ceil(2N/3)`).
- **A3 — Compromised submitter:** Controls submission credentials. Can submit false entries but cannot forge validator signatures.
- **A4 — External adversary:** No key access. May attempt denial of service or replay.
- **A5 — Quantum adversary:** Possesses a quantum computer capable of breaking classical cryptography. 3CP v3.0 uses NIST post-quantum primitives (ML-DSA-65, ML-KEM-1024) to resist A5. Residual classical assumptions (BLAKE3 collision resistance, SHA-256 preimage resistance) are stated in [§17 Security Considerations](#17-security-considerations).

---

## 3. Cryptographic Primitives

This section is normative. Sizes are the exact byte lengths defined by the cited standards; an implementation **MUST** use the parameter set as specified.

### 3.1 Hash Functions

| Function | Output | Use |
|----------|--------|-----|
| `BLAKE3-256` | 32 bytes | Entry hashes, SMT leaves, SMT nodes, state root, `NetworkID` |
| `SHA-256` | 32 bytes | `BlockHash` (block identifier) |
| `HKDF-SHA256` | Variable (`L` octets) | Key derivation, UID0 seeds, AEAD key derivation |

References: BLAKE3 (reference implementation and specification), FIPS 180-4 (SHA-256), RFC 5869 (HKDF).

### 3.2 Digital Signatures

**Algorithm:** ML-DSA-65, per FIPS 204. (3CP historically called this "Dilithium3"; "Dilithium3" is the pre-standardization name. v3.0 uses the standardized name **ML-DSA-65** whenever FIPS 204 is cited.)

| Parameter | Value (FIPS 204, Table 2) |
|-----------|---------------------------|
| Public key | 1,952 bytes |
| Private key | 4,032 bytes |
| **Signature** | **3,309 bytes** |
| Security level | NIST category 3 (AES-192 equivalent) |

**[MUST]** Every signature in the 3CP protocol uses ML-DSA-65 unless explicitly stated otherwise. Signature sizes **MUST** be 3,309 bytes; a conformant verifier **MUST** reject any signature that is not exactly 3,309 bytes (FIPS 204 §10.2 length checks).

**[SHOULD]** Signers **SHOULD** use the hedged (randomized) signing mode of FIPS 204 §3.4 rather than the deterministic variant, because hedged signing mitigates fault and side-channel attacks. Verifiers are unaffected: the same verification algorithm accepts both variants.

### 3.3 Verifiable Random Function (VRF)

**Suite:** `ECVRF-RISTRETTO255-SHA512`, as specified by the Crypto Verification Working Group specification `vrf-r255` (C2SP), which realises the ECVRF construction of RFC 9381 over the prime-order group Ristretto255.

- Group: Ristretto255 (prime order, cofactorless)
- Internal hash: SHA-512
- Public key / secret key: 32 bytes each
- Proof: 96 bytes (`Gamma || C || S`), where `Gamma` is 32 bytes, `C` is 32 bytes, `S` is 32 bytes
- Output `Gamma`: 32 bytes

**Correction from v2.0.** v2.0 documents variously described the suite as `ECVRF-EDWARDS25519-SHA512-Elligator2` (a suite over the cofactor-8 edwards25519 group, not Ristretto255) or as a bespoke "2-point Elligator2 sum" hash-to-curve that matches no published standard. v3.0 adopts the published `ECVRF-RISTRETTO255-SHA512` suite. Implementations **MUST** use it exactly.

**Non-conformance of the edwards25519 suites.** RFC 9380 §8.5 states that its `edwards25519` / `curve25519` Elligator2 ciphersuites **MUST NOT** be used when hashing to `ristretto255`. 3CP targets Ristretto255 and therefore uses the Ristretto255 suite, not the edwards25519 suites.

**Nonce derivation.** The ECVRF nonce `k` is derived deterministically with domain separation as specified in the `vrf-r255` suite: `k_string = Hash(suite_string || 0x81 || int_to_string(SK, 32) || h_string)`; `k = string_to_int(k_string) mod q`. Every `Hash` instantiation in the suite is domain-separated by a fixed prefix. v2.0's "HMAC-SHA512 with the VRF private key as HMAC key" derivation is non-standard and is **removed**.

**[MUST]** A verifier **MUST** verify every received VRF proof against the sender's VRF public key obtained from the canonical `ValidatorSet` (never from local verifier state).

### 3.4 Key Encapsulation Mechanism (KEM)

**Algorithm:** ML-KEM-1024, per FIPS 203.

| Parameter | Value |
|-----------|-------|
| Public key (encapsulation key) | 1,568 bytes |
| Ciphertext | 1,568 bytes |
| Shared secret | 32 bytes |

**[MUST]** ML-KEM-1024 is used exclusively for the peer transport handshake (encapsulate/decapsulate + AEAD). **[MUST]** Decapsulation uses the implicit-rejection behaviour of FIPS 203 §7.3 (a decapsulation with an incorrect ciphertext yields a pseudorandom shared secret, not an error that distinguishes failure).

### 3.5 AEAD

**Algorithm:** ChaCha20-Poly1305, per RFC 8439 (AEAD_CHACHA20_POLY1305, 256-bit key, 96-bit nonce, 128-bit tag).

**[MUST]** Transport encryption after the KEM handshake uses ChaCha20-Poly1305. The 32-byte key is derived via HKDF-SHA256 from the ML-KEM shared secret. **[MUST]** A peer **MUST NOT** reuse a (key, nonce) pair. Nonce generation and key rotation are transport-construction concerns and **MUST** follow RFC 8439 §12.

### 3.6 Domain Separation

All authenticated structures **MUST** include an explicit, fixed-length domain-separation tag in the hashed preimage so that no preimage in this protocol is a prefix of, or equal to, a preimage of another. See the preimage constructions in [§5.5](#55-blockhash-construction-with-domain-separation), [§8.2](#82-key-rotation-signed-payload-domain-separated), and [§9.1](#91-parameters).

---

## 4. Wire Encoding (Canonical CBOR)

### 4.1 Encoding Rules

**[MUST]** All blocks, entries, mandates, and state structures are encoded using the Deterministic Encoding Requirements of RFC 8949 §4.2.1. Conformance to this document therefore requires:

- Map keys sorted by their encoded byte representation (length-first, then lexicographic). Integer keys 0–23 are encoded in the single-byte form `0x00`–`0x17`.
- Integers encoded in the shortest form that round-trips.
- **Floating-point values encoded in the shortest of IEEE 754 binary16, binary32, or binary64 that represents the value exactly** (RFC 8949 §4.2.1). A value **MUST NOT** be encoded in a wider format than necessary. `Lambda1` and `SeverityMin`/`SeverityMax` are affected by this rule.
- **NaN and Infinity are not permitted** anywhere in the 3CP encoding. An encoder **MUST NOT** emit them; a decoder **MUST** reject a structure containing one.
- Strings, byte strings, and arrays use definite-length encoding.

**Correction from v1.0/v2.0.** Earlier drafts stated "float values use 64-bit IEEE 754 encoding". That is incompatible with RFC 8949 §4.2.1 deterministic encoding and is **replaced** by the shortest-round-trip rule above. Encoding `Lambda1` as a fixed 8-byte double regardless of value would produce a different byte string (and therefore a different `BlockHash`) than the canonical encoding; a peer that decodes and re-encodes canonically must obtain the identical byte string.

### 4.2 Structure Identifier (Domain Tag)

Every top-level structure **MUST** be hashed and, where applicable, signed over a domain-separated preimage that begins with a fixed ASCII tag. The tags are:

| Structure | Tag |
|-----------|-----|
| Block hash preimage | `"3CP-BLOCKHASH-v3"` |
| Key-rotation signed payload | `"3CP-KEYROT-v3"` |
| SMT leaf preimage | `"3CP-SMT-LEAF-v3"` |
| SMT internal-node preimage | `"3CP-SMT-NODE-v3"` |
| Mandate content preimage | `"3CP-MANDATE-v3"` |
| Entry signed payload | `"3CP-ENTRY-v3"` |

The tag is prepended (as a length-prefixed byte string in canonical CBOR, or as fixed ASCII bytes in a fixed-length binary preimage) to the relevant body before hashing/signing. This prevents cross-structure and cross-version confusion when two structures share field layouts.

---

## 5. Block Format

### 5.1 Block Structure

All keys are integers in ascending order. Field sizes are exact.

```
validator-info = {
  0 => bytes .size 16,              ; ValidatorID (RootID)
  1 => bytes .size 1952,            ; ML-DSA-65 public key
  2 => bytes .size 32,              ; VRF public key (Ristretto255 point)
  3 => bytes .size 32,              ; ContractHash (zeros if absent)
}

Block {
  0  => uint64,                     ; Index
  1  => bytes .size 32,             ; PrevHash  (BLAKE3-256 of previous BlockHash; zeros at genesis)
  2  => bytes .size 32,             ; StateRoot (BLAKE3-256 SMT root after this block's insertions)
  3  => bytes .size 16,             ; Proposer  (RootID of the leader)
  ? 4 => [3 * bytes],               ; Triad     (reserved; omitted in v3.0)
  5  => [* provenance-entry],       ; Anchored  (entries anchored in this block)
  6  => float,                      ; Lambda1   (Fiedler eigenvalue; see §4.1 float rule)
  7  => int64,                      ; Timestamp (UnixNano at finalization)
  ? 8 => null,                      ; Reserved  (was Sigs in v1.0; MUST be null in v3.0 — see §5.2)
  9  => [* validator-info],         ; Validators (canonical validator set, see §7.1)
  10 => quorum-config,              ; Quorum
  11 => bytes .size 32,             ; BlockHash (SHA-256; see §5.5)
  12 => uint16,                     ; ProtocolVersion (== 3 in v3.0 networks)
  13 => bytes,                      ; PrepareSigsBitmap (see §5.4)
  14 => [* bytes .size 3309],       ; PrepareSigsPayload (see §5.4)
  15 => bytes .size 3309,           ; CommitSig (leader signature over B_final — see §5.6)
  16 => [* tstr],                   ; ExternalAnchors (publication URIs/CIDs)
  17 => uint64,                     ; KeyRotationEpoch (reference cycle for active keys)
  18 => bytes .size 32,             ; LegacyAnchor (genesis only; zeros otherwise)
  19 => {* tstr => bytes},          ; Metadata (e.g. "3cp:degraded-block", §6.5)
}
```

**[MUST]** In a v3.0 network, `ProtocolVersion` (key 12) **MUST** equal 3. A node **MUST** reject a block whose `ProtocolVersion` it does not support.

**Adopted from upstream #2.** `Validators` (key 9) is `[* validator-info]` — a canonical ordered structure carrying each validator's `ML-DSA-65` and VRF public keys — rather than a bare array of public-key bytes. This makes the validator set self-describing (so a verifier can reconstruct the VRF keys from the block) while `Metadata` (key 19) is the declared home for mode flags such as `3cp:degraded-block`. `Triad` (key 4) and reserved key 8 are **optional** members (`?`) and MUST be omitted, not encoded empty.

**[MUST]** A verifier **MUST NOT** derive the validator set from the block being verified. The set must come from an external trust anchor — the genesis block, an Anchor Publisher, or a full node the operator vouches for (§13.4). A block that names its own validators can be signed entirely by whoever wrote it.

### 5.2 Reserved Key 8 (Correction from v2.0 Divergence)

**[MUST]** Key 8 is reserved in `ProtocolVersion == 3`. It is an **optional** member and **MUST** be omitted; when present it **MUST** be CBOR `null`. A v3.0 block in which key 8 is present with a non-null value **MUST** be rejected with `ErrReservedFieldPresent`.

Rationale: v1.0 used key 8 for undifferentiated validator signatures. Reusing key 8 in v2.0 with the meaning "PREPARE signatures" made the meaning of the field depend on `ProtocolVersion`, forcing every parser to branch on version before interpreting key 8. v3.0 makes PREPARE signatures live exclusively in keys 13/14 and keeps key 8 unambiguously null.

A node operating against a **legacy v1.0** chain continues to interpret key 8 per `spec/3CP.md` §4.1; v1.0 and v3.0 never coexist on the same network (migration is a hard fork, §15).

### 5.3 Epoch Extension

**[MUST]** Implementations **MUST** ignore map keys greater than 19 when decoding a `Block`, and **MUST NOT** reject a block solely because it carries an unknown higher key. This forward-compatibility rule is required for extensibility (RFC 8949 §4.2.1, RFC 9413).

### 5.4 Quorum Signature Fields

**[MUST]** `PrepareSigsBitmap` (key 13) is a bitfield of `ceil(N/8)` bytes where `N` is the number of validators. **Bits are numbered most-significant-bit-first within each byte**: bit `i` of the field is the bit at position `7 - (i mod 8)` of byte `floor(i/8)`. Bit `i` is `1` if and only if the validator at index `i` in `Validators` (key 9) signed PREPARE. Bits with index `i >= N` **MUST** be zero.

*Example:* `N = 4`, validators `val0..val3`, PREPARE from `val0, val1, val2` → bitmap `0xE0` (`1110 0000`).

**[MUST]** `PrepareSigsPayload` (key 14) contains exactly `popcount(PrepareSigsBitmap)` signatures of 3,309 bytes each, ordered by ascending validator index. A validator's signature occupies the position given by the ordering of the set bits of the bitmap.

**[MUST]** Each PREPARE signature is an ML-DSA-65 signature by the corresponding validator over the block hash `H(B)` as defined in §5.5, with an empty context string (FIPS 204 `ctx` = `""`). The verifying key is the `ML-DSA-65 public key` (field 1) of the `validator-info` at that index in key 9.

### 5.5 BlockHash Construction (with Domain Separation)

**[MUST]** The `BlockHash` (key 11) is computed as a domain-separated SHA-256:

```
BlockHash = SHA-256(
    "3CP-BLOCKHASH-v3"            ; 17 ASCII bytes, domain-separation tag
    || LE64(Index)                ; 8 bytes little-endian
    || PrevHash                   ; 32 bytes
    || StateRoot                  ; 32 bytes
    || Proposer                   ; 16 bytes
    || HashOfAnchoredEntries      ; 32 bytes, see below
    || LE64(Timestamp)            ; 8 bytes little-endian
    || QuorumConfigCanonical      ; canonical CBOR bytes of the quorum-config map
)
```

**[MUST]** `HashOfAnchoredEntries` is `BLAKE3-256` of the canonical CBOR serialization of the array at key 5.

**Rationale for the changes from v1.0/v2.0.** v1.0 concatenated the individual entry hashes directly into the preimage. v2.0 replaced that with `HashOfAnchoredEntries`. v3.0 adds the `NetworkID` binding and the explicit domain tag. Including `NetworkID` (§11.1) in the preimage cryptographically binds every block to its genesis and prevents cross-chain replay of a block's identity. See [Appendix D](#appendix-d--corrections-applied-in-v30) item D4 for the analysis of the v2.0 commitment gap.

### 5.6 CommitSig Construction

**[MUST]** `CommitSig` (key 15) is the leader's ML-DSA-65 signature over `B_final` as defined in §6.3 (the block including keys 13 and 14, excluding key 15 itself). Context string is empty.

---

## 6. Two-Phase BFT Consensus

### 6.1 Consensus Cycle

A **cycle** `c` is the atomic unit of time. Each cycle produces zero or one final block of index `c`. **[MUST]** A cycle is divided into the two atomic phases PREPARE and COMMIT.

### 6.2 PREPARE Phase

**Input:** network state at the start of cycle `c`, pending entry set `E`, validator set `V`.

**Step 1 — Leader election.** The VRF input is:

```
alpha_c = "3CP-VRF-ALPHA-v3" || LE64(c) || StateRoot_at_cycle_start
```

**[MUST]** `StateRoot_at_cycle_start` is the SMT root **before** any cycle-`c` insertions. `alpha_c` is fixed at cycle start and **MUST NOT** change during the phases.

Each validator `v_i` produces `proof_i = VRF_Prove(sk_i^VRF, alpha_c)` and publishes it. A validator verifies every received proof against the canonical `ValidatorSet` VRF public key. The leader is the validator with the lexicographically smallest `Gamma`; ties are broken by the smallest `ValidatorID`.

**Step 2 — Proposal.** The leader proposes candidate block `B` with:
- `Index = c`
- `PrevHash = BLAKE3-256(BlockHash of B_{c-1})` (32 zero bytes at genesis)
- `StateRoot =` SMT root after inserting `E`
- `Proposer =` leader `ValidatorID`
- `Anchored = E`
- `Lambda1`, `Timestamp = now()`
- `Validators = [v_0.pk, …, v_{N-1}.pk]` in canonical order
- `Quorum = {TotalValidators: N, RequiredSigs: ceil(2N/3)}`

**Step 3 — PREPARE vote.** Each validator `v_j` verifies:
1. `B.PrevHash` matches its locally accepted previous block.
2. `B.StateRoot` equals the root it computes after inserting `E` in the same order.
3. Every `e ∈ E` passes `validateEntry(e)` (§6.7).
4. `B.Lambda1 >= MinLambda1` where applicable.
5. `B.ProtocolVersion == 3`.

If all pass, `v_j` signs `H(B)` (ML-DSA-65, empty context) and broadcasts its PREPARE signature.

### 6.3 COMMIT Phase

**Step 4 — Quorum collection.** The leader collects PREPARE signatures until it has `Q = ceil(2N/3)`. **[MUST]** If `Q` signatures are not collected within `CycleTimeout`, the cycle is **aborted**; the entries in `E` remain pending and cycle `c` produces no block.

**Step 5 — Final block construction.**

```
B_final = B || PrepareSigsBitmap || PrepareSigsPayload || CommitSig
```

where `CommitSig = ML-DSA65_Sign(sk_leader, H(B_final_excluding_key15), ctx="")`.

**[MUST]** The PREPARE signatures cover `H(B)` (the proposal, keys 0–12), and the `CommitSig` covers `H(B_final)` including keys 13/14 but excluding key 15. PREPARE and COMMIT signatures therefore cover distinct, non-overlapping commitments, eliminating the self-reference that made the v2.0 `CommitSig` definition circular.

**Step 6 — Diffusion and validation.** The leader broadcasts `B_final`. Each validator verifies:
1. `PrepareSigsBitmap` has at least `Q` bits set, with high bits (index `>= N`) zero.
2. `popcount(PrepareSigsBitmap) == len(PrepareSigsPayload)`, and each PREPARE signature verifies against the validator indicated by its bit position.
3. `CommitSig` verifies against `B.Proposer` per §5.6.

If all pass, `v_j` accepts `B_final` as final, appends it, and transitions state.

### 6.4 Aborted Cycle

**[MUST]** On abort (PREPARE timeout, insufficient quorum, or `λ₁ < MinLambda1`):
- No block is appended.
- Entries in `E` remain pending.
- Cycle `c+1` starts a fresh VRF election.

### 6.5 Degraded Mode (Non-BFT)

**[MUST]** If `N < 4`, the protocol runs in **degraded mode**: `Q = 1`. Every degraded block **MUST** set `Metadata["3cp:degraded-block"] = true` (key 19), so that any third party can tell a degraded block from a quorum-backed one. Degraded mode is a **single-signature trust configuration**, equivalent to a single-node deployment; it does not provide Byzantine fault tolerance and **MUST NOT** be relied upon for non-repudiation against a single compromised key. Exiting degraded mode requires `N >= 4` and `GraceCycles` consecutive cycles (default 10) with normal quorum.

**[MUST]** A mandate applies regardless of the operating mode. The quorum rule above governs block finality, not anchoring obligations: a mandate **MUST NOT** be treated as satisfied with less than its own rules require, even when the chain is degraded.

### 6.6 Slashing Evidence

A Byzantine leader that double-proposes for a cycle produces cryptographic evidence: two distinct blocks for the same `Index` with valid leader `CommitSig`s over each. Any validator **MAY** submit this pair as a `ProvenanceEntry` with `Label = "3cp:slash-evidence:v1"`.

### 6.7 Entry Validation

**[MUST]** A node **MUST** reject a submission where any of the following holds:

| Condition | Error |
|-----------|-------|
| `Hash` is all zeros | `ZERO_HASH` |
| `Submitter` is empty | `EMPTY_SUBMITTER` |
| `Label` exceeds 1024 bytes | `LABEL_TOO_LONG` |
| `Signature` present but not a valid ML-DSA-65 signature by the claimed `Submitter` over the §4.2 `3CP-ENTRY-v3` payload | `INVALID_SIGNATURE` |
| `Submitter` is not a recognised identity | `SUBMITTER_MISMATCH` |
| Submit rate exceeds the configured limit | `RATE_LIMITED` |
| `MandateRef` present but unknown | `UNKNOWN_MANDATE` |
| Referenced mandate not yet active | `MANDATE_NOT_ACTIVE` |
| Referenced mandate expired | `MANDATE_EXPIRED` |
| Entry missing a field required by the referenced mandate rule | `MISSING_REQUIRED_FIELD` |

When an entry does **not** carry a `MandateRef` but matches an active mandatory mandate rule, the node **SHOULD** reject with `MANDATE_REF_REQUIRED`. A node **MUST NOT** reject entries that match no active mandatory rule.

---

## 7. Network State and ValidatorSet

### 7.1 GenesisValidatorSet

**[MUST]** The genesis block (index 0) anchors a `GenesisValidatorSet`, immutable for the life of the chain:

```
genesis-validator-info = {
  0 => bytes .size 16,    ; ValidatorID (RootID)
  1 => bytes .size 1952,  ; ML-DSA-65 public key
  2 => bytes .size 32,    ; VRF public key (Ristretto255 point)
  3 => bytes .size 32,    ; ContractHash (zeros if absent)
}
```

Changes to the validator set (addition, removal, key rotation) occur via subsequent protocol entries (§8).

### 7.2 NodeState

**[MUST]** Network state maintains, per known node: `RootID`, `Status` (float), `Consecutive` (uint), `ML-DSA-65 public key`, and `VRF public key`.

**[MUST]** Verification of a peer's VRF proof **MUST** use the `VRFPK` from the canonical `ValidatorSet`, never from the verifier's local identity or from mutable gossip.

---

## 8. Key Rotation

### 8.1 Key-Rotation Entry

**[MUST]** 3CP defines a protocol-native key-rotation entry, `EventClass = "3cp:key-rotation:v1"`:

```
key-rotation-entry = {
  ;; base provenance-entry fields
  0  => bytes .size 32,    ; Hash (BLAKE3-256 of the §8.2 signed payload)
  1  => bytes .size 16,    ; Submitter (ValidatorID)
  2  => int64,             ; Timestamp (UnixNano)
  3  => tstr,              ; Label: "3cp:key-rotation:v1"

  ;; rotation-specific fields
  20 => bytes .size 1952,  ; NewPublicKey (ML-DSA-65)
  21 => bytes .size 32,    ; NewVRFPublicKey (Ristretto255 point)
  22 => uint64,            ; EffectiveCycle
  23 => uint64,            ; ExpiryCycle
  24 => bytes .size 3309,  ; SignatureOld (ML-DSA-65 with the current key)
  25 => bytes .size 3309,  ; SignatureNew (ML-DSA-65 with NewPublicKey)
}
```

### 8.2 Key-Rotation Signed Payload (Domain-Separated)

**[MUST]** `SignatureOld` and `SignatureNew` are ML-DSA-65 signatures (empty context) over the **same** domain-separated payload:

```
KeyRotationPayload = "3CP-KEYROT-v3" || Hash
```

where `Hash` is field 0: `BLAKE3-256` of the canonical CBOR encoding of fields 1–3 and 20–23 (the entry with the `Hash` and both signature fields removed). Both the old and the new key sign this identical payload, proving possession of the new key and authorisation by the current key.

**Correction from v2.0.** v2.0 defined `Hash` ambiguously as "BLAKE3-256 of the payload" without stating which fields were excluded, and the reference implementation's vectors disagreed on this point. v3.0 defines the exact preimage.

### 8.3 Validation Rules

**[MUST]** A key-rotation entry is valid if and only if:
1. `SignatureOld` verifies against the submitter's active ML-DSA-65 key in the current cycle's `ValidatorSet`.
2. `SignatureNew` verifies against `NewPublicKey`.
3. `EffectiveCycle >= currentCycle + KeyRotationLeadTime` (default 10).
4. `ExpiryCycle >= EffectiveCycle + MinKeyOverlap` (default 10).
5. `EffectiveCycle > lastRotationCycle` for the same validator (no overlapping rotations).

### 8.4 Overlap Period

During `[EffectiveCycle, ExpiryCycle]` both keys are accepted for verification. After `ExpiryCycle` only the new key is accepted for new signatures. **[MUST]** Blocks before `EffectiveCycle` remain verifiable with the old key indefinitely, so historical verification never depends on a retired key being retained online.

---

## 9. Sparse Merkle Tree (SMT)

### 9.1 Parameters

- Depth: 256
- Leaf hash: `BLAKE3-256("3CP-SMT-LEAF-v3" || key || value)`, `key` and `value` each 32 bytes
- Internal node: `BLAKE3-256("3CP-SMT-NODE-v3" || left || right)`, `left`/`right` each 32 bytes
- Empty-subtree hash: `BLAKE3-256("3CP-SMT-NODE-v3" || EMPTY || EMPTY)` where `EMPTY` = `BLAKE3-256("")`
- Bit ordering for path selection: LSB-first within each byte of the key (bit `i` of the key = `(key[i//8] >> (i%8)) & 1`)
- Key = `entry.Hash`; Value = `entry.Hash`

**Correction from v1.0/v2.0.** The v1.0 leaf was `BLAKE3("leaf" || key || value)` and the node `BLAKE3(left || right)` with no domain separation and an empty hash of `BLAKE3("")`. Adding explicit tags (§4.2) prevents second-preimage confusion across the leaf, node, and empty encodings; all three preimages are distinct by construction.

### 9.2 Proof Size

**[MUST]** An SMT proof is an array of exactly 256 sibling hashes of 32 bytes each (8,192 bytes total), ordered leaf-to-root. A proof of absence **MUST** produce a sibling path that recomputes to the current `StateRoot`.

### 9.3 Verification

```
function VerifySMTProof(root, key, value, proof):
    current = BLAKE3-256("3CP-SMT-LEAF-v3" || key || value)
    for depth = 0 to 255:
        sibling = proof[depth]
        bit = (key[depth // 8] >> (depth % 8)) & 1
        if bit == 0:
            current = BLAKE3-256("3CP-SMT-NODE-v3" || current || sibling)
        else:
            current = BLAKE3-256("3CP-SMT-NODE-v3" || sibling || current)
    return current == root
```

---

## 10. Adaptive Cycle and Entry Retention

### 10.1 Cycle Duration

**[MUST]** The cycle duration is adaptive:

```
CycleDuration = BaseInterval + EWMA(RTT) * SafetyFactor
```

with `BaseInterval` default 3 s, `SafetyFactor` default 1.5, and a hard cap `MaxCycleDuration` of 10 s. `BaseInterval` and `SafetyFactor` are configurable by Mandate.

**Correction from v2.0.** The v2.0 Portuguese text wrote `BaseInterval` default as "3.000ms" and `MaxCycleDuration` as "10.000ms" — i.e. three to ten **milliseconds** — while English prose and v1.0 specified `cycleInterval = 3 s`. A consensus cycle is meaningless below one second (signing + broadcast + validation across peers takes far longer than 10 ms), so the v2.0 "ms" figures are a unit error; v3.0 fixes the unit to seconds.

### 10.2 Entry Retention

**[MUST]** Pending entries are **never** dropped due to cycle expiry. **[MUST]** Entries pending for more than `MaxPendingTTL` cycles (default 100) are rejected with `ErrPendingExpired`.

### 10.3 Empty Cycle

**[MAY]** If `SkipEmptyCycles` is true and there are no pending entries, a cycle may be skipped with no block produced.

---

## 11. Identity (UID0) and NetworkID

### 11.1 NetworkID

**[MUST]** `NetworkID = BLAKE3-256(canonical CBOR of the genesis block with the `BlockHash` field (key 11) omitted)`, and is immutable for the life of the chain.

**Genesis ordering (normative).** Because `BlockHash` (§5.5) binds `NetworkID`, `NetworkID` **MUST NOT** be computed over a genesis block whose `BlockHash` already incorporates `NetworkID`. A conformant implementation **MUST** compute, in this order:

1. Construct the genesis block with key 11 (`BlockHash`) set to 32 zero bytes.
2. `NetworkID = BLAKE3-256(canonical CBOR(genesis block with key 11 omitted))`.
3. `BlockHash = SHA-256` over the §5.5 preimage using that `NetworkID`.
4. Store the computed `BlockHash` at key 11.

The same ordering applies to the genesis `BlockHash` and to `LegacyAnchor`
during migration (§14).

**Correction from v2.0.** v2.0 wrote `NetworkID = BLAKE3-256(GenesisBlock)` without specifying the encoding, which left both the serialization and this ordering implicit. Because `NetworkID` is a cryptographic salt and is bound into every `BlockHash` (§5.5), the encoding **MUST** be the canonical CBOR serialization and the ordering **MUST** be fixed as above; omitting key 11 removes the circularity.

### 11.2 UID0 Seed Derivation

**[MUST]** The 32-byte identity seed is derived with HKDF-SHA256 (RFC 5869) with the **output length parameter `L` fixed to 32**:

```
seed = HKDF-SHA256(
    salt = NetworkID,                  ; 32 bytes
    info = "3CP-UID0-v3",              ; 12 ASCII bytes
    ikm  = entropySource,              ; 32 bytes
    L    = 32                          ; output length in octets
)
```

**Correction from v1.0/v2.0.** v1.0 derived the seed from a 16-byte `FEntropy` and v2.0 from a 32-byte `entropySource`; v3.0 standardises on a **32-byte** `entropySource` sourced from the operating system CSPRNG (FIPS 140-3 / SP 800-90A `getrandom`). v2.0's phrasing "reject `entropySource` with estimated entropy < 80 bits" is **removed**: entropy of a byte buffer cannot be measured by an implementation, making the requirement untestable. The requirement is instead testable and unambiguous: the input is 32 bytes drawn from an approved RBG.

---

## 12. Third-Party Verifiability

### 12.1 Anchor Publishers

**[MUST]** An Anchor Publisher publishes final blocks to publicly-readable storage. Supported backends: local filesystem; IPFS (CIDv1, `raw` codec, `blake3-256` or `sha2-256` multihash); S3-compatible blob store (SHA-256 checksum). `MinRedundancy` defaults to 2 publishers.

### 12.2 Light Client Protocol

**[MUST]** The light-client read interface is:

```
GetBlock(index: uint64) -> Block
StreamBlocks(startIndex: uint64) -> stream Block
GetValidatorSet(cycle: uint64) -> [* validator-info]
GetMerkleProof(key: bytes .size 32, blockIndex: uint64) -> SMTProof
```

**[MUST]** A light client verifies block `B` without executing consensus:
1. Obtain the canonical `ValidatorSet` for cycle `B.Index`.
2. Verify `popcount(B.PrepareSigsBitmap) >= Q = ceil(2N/3)` and each PREPARE signature against the validator at its bit position (§5.4).
3. Verify `B.CommitSig` against `B.Proposer` per §5.6.
4. Recompute `B.BlockHash` per §5.5 and check it matches key 11.
5. Verify the `PrevHash` chain back to a trusted genesis (whose `BlockHash` is the client's trust anchor).

---

## 13. Mandates

*Adopted from upstream PR #3 (`72a5e4b`), re-verified against the v3.0 suite.*

### 13.1 Structure

See `spec/schemas/mandate.cddl` for the complete CDDL definition.

**[MUST]** A mandate is a signed, versioned, anchored declaration that defines anchoring obligations. `Authority` is the RootID that issued it; `Version` and `PrevVersion` form the revision lineage; `Rules` declares the event classes it covers.

**[MUST]** A mandate's `MandateID` is `BLAKE3-256` of the canonical CBOR of the mandate with `MandateID` and `Signature` excluded, domain-tagged `3CP-MANDATE-v3` (§4.2). An implementation **MUST** recompute it and **MUST NOT** trust the stored value: a body whose rules differ from the identifier it presents **MUST** be rejected.

### 13.2 Authenticity

**[MUST]** `Signature` (key 10) is an ML-DSA-65 signature (3,309 bytes) by the `Authority` over that same canonical payload, with `MandateID` and `Signature` excluded, empty context.

**[MUST]** An implementation **MUST** verify this signature before accepting a mandate as binding policy, and **MUST** resolve it against a registered ML-DSA-65 key — the ValidatorSet or network state. A mandate whose `Authority` does not resolve to a known key **MUST** be rejected, and an absent or mis-sized signature **MUST** be rejected before any expensive verification.

Without this check `Authority` is only an assertion: any submitter could install a mandate in another node's name, and the network would treat its rules as authentic policy. A missing signature **MUST** be treated as an authorisation failure, not as the requirement not applying.

**[MUST]** The validity window (`ValidFrom`, `ValidUntil`) **MUST NOT** be enforced at admission. An auditor must be able to load an expired mandate to assess a window it covered; refusing it at admission would erase the record the audit is looking for.

### 13.3 Submission-Time Enforcement

**[MUST]** An entry carrying `MandateRef` **MUST** be checked against the mandate it references: the mandate **MUST** exist, be in force at the entry's timestamp, and have every field its rules require.

An entry with no `MandateRef` makes no claim and **MUST** be accepted. Most entries in a provenance chain are governed by no mandate, and rejecting them would make the mechanism unusable.

This half of enforcement is structural: it stops claims of compliance that fail the most basic requirements. It **CANNOT** detect omitted events, because the protocol never sees an event that was never submitted.

### 13.4 Verification and Enforcement

See `spec/schemas/mandate.cddl` for the `compliance-verification` and `compliance-gap` structures.

**[MUST]** An implementation **MUST** offer an operation that checks the chain against a mandate over a time window, comparing the anchored entries against the declared obligations.

**[MUST]** Omission **MUST** be cryptographically detectable: an event a mandate requires that was never anchored **MUST** produce a `missing` gap, distinguishable from an entry that is present but incomplete (`missing_fields`).

**[MUST]** Verification **MUST NOT** derive the validator set from the block being verified. The set has to come from an external trust anchor — the genesis block, an Anchor Publisher, or a full node the operator vouches for. A block that names its own validators can be signed entirely by whoever wrote it (see also §5.1).

### 13.5 Degraded Mode

**[MUST]** A mandate applies regardless of the operating mode. The quorum rule of §6.5 governs block finality, not anchoring obligations: a mandate **MUST NOT** be treated as satisfied with less than its own rules require, even when the chain is degraded.

---

## 14. ZK Bridge (Informative Contract)

**[MUST]** The ZK Bridge interface is versioned; breaking changes require a major version bump, with backward compatibility maintained for at least two major versions. See `spec/schemas/zk-bridge.cddl`.

**[SHOULD]** A ZK circuit consuming 3CP hashes should use STARK-friendly hashes (e.g. Poseidon2, Rescue-Prime) for hashing **inside** the circuit, while keeping BLAKE3-256 for hashing **outside** it.

---

## 15. Migration v1.0/v2.0 → v3.0

**[MUST]** Migration is a **hard fork**. No in-place upgrade exists.

1. Halt all v1.0 validators at the same final block `B_final`.
2. Export: `ValidatorSet`, active Mandates, final SMT root, `H(B_final)`.
3. Create a v3.0 genesis block with `ProtocolVersion = 3`, the exported `InitialValidators`, converted `GenesisMandate`, and `LegacyAnchor = H(B_final)`.
4. Start the v3.0 network from the new genesis.

---

## 16. Conformance

An implementation claiming conformance with 3CP v3.0 **MUST** pass the extended test suite in `spec/examples/` and satisfy the normative requirements of this document. The conformance suite covers entry hashing, block-hash construction, SMT inclusion/absence, key rotation, light-client verification, and ML-DSA-65 / ML-KEM-1024 primitive conformance against FIPS 203/204 sizes.

**Non-normative note:** 3CP has not yet been validated by an independent second implementation. Until one exists and demonstrates interoperability, the specification may contain untested edge cases.

---

## 17. Security Considerations

*(Model on RFC 3552 §5/§6. This section discusses the concrete threats 3CP faces, what it protects and does not protect, key-management consequences, and residuals.)*

### 16.1 What 3CP Protects

Against an adversary that cannot forge ML-DSA-65 signatures or break BLAKE3/SHA-256 collision resistance, 3CP provides: integrity of anchored entries via SMT proofs; non-repudiation of block-level submissions via ML-DSA-65 quorum signatures; contestability of evidence (challenges occur over the intact chain); and third-party verifiability without operator cooperation.

### 16.2 What 3CP Does Not Protect

- **Access control:** 3CP records who did what; it does not decide who may do what.
- **Preimage confidentiality:** entry hashes are public; hash sensitive preimages before submission.
- **Degraded mode (§6.5):** with `N < 4` a single signature finalizes a block. This is a single-key trust model, not BFT.
- **Availability:** 3CP does not prevent a validator majority from censoring (deliberately aborting cycles); it makes the *absence* of blocks detectable, not preventable.

### 16.3 Replay, Downgrade, and Cross-chain Replay

`PrevHash` chains blocks; `Index` and `NetworkID` bind each `BlockHash` to its chain (§5.5), so a valid block from another 3CP network does not replay. The domain-separation tags (§4.2) prevent cross-structure preimage reuse.

### 16.4 Key Management Consequences

All guarantees are void if signing keys are co-located with the credentials they are meant to constrain; custody separation is the operator's responsibility. ML-DSA-65 hedged signing (§3.2) is recommended against fault/side-channel attacks. Key rotation (§8) uses dual old/new signatures to prove continuity of authority across an overlap window.

### 16.5 Implementation Failure Consequences

A verifier that does not enforce exact signature lengths (§3.2), reserved-key rejection (§5.2), bitmap high-bit rules (§5.4), or ML-KEM implicit rejection (§3.4) can accept malformed or downgraded input. These are the recommended hardening points.

### 16.6 Privacy

3CP entries are public by design; correlation and metadata are out of scope for this version.

---

## 18. Normative References

- **BCP 14:** RFC 2119 (Key words for use in RFCs), RFC 8174 (Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words)
- **CBOR:** RFC 8949 (Concise Binary Object Representation)
- **CDDL:** RFC 8610 (Concise Data Definition Language)
- **Hashing to curves:** RFC 9380 (Hashing to Elliptic Curves)
- **VRF:** RFC 9381 (Verifiable Random Functions); C2SP `vrf-r255` (ECVRF-RISTRETTO255-SHA512 suite)
- **AEAD:** RFC 8439 (ChaCha20 and Poly1305 for IETF Protocols)
- **HKDF:** RFC 5869 (HMAC-based Extract-and-Expand Key Derivation Function)
- **Hash functions:** FIPS 180-4 (SHA-2); BLAKE3 specification (reference implementation)
- **Post-quantum KEM:** FIPS 203 (ML-KEM)
- **Post-quantum signatures:** FIPS 204 (ML-DSA)
- **Randomness:** SP 800-90A/B/C (approved DRBGs)
- **Robustness:** RFC 9413 (Maintaining Robust Protocols)

---

## Appendix A — CDDL Summary

The complete, canonical CDDL definitions live in `spec/schemas/`:

- `block.cddl`, `mandate.cddl`, `genesis.cddl`, `network-state.cddl`, `key-rotation.cddl`, `light-client.cddl`, `zk-bridge.cddl`, `smt.cddl`, `vrf.cddl`, `cross-chain.cddl`

All signature fields are **3,309 bytes** (ML-DSA-65); all public-key fields are **1,952 bytes**.

---

## Appendix B — Formal Algorithms

### B.1 Proposer Selection

```
function SelectProposer(proofs, alpha, validator_set):
    best_gamma = null; best_proposer = null
    for (signer_id, proof) in proofs:
        pk = validator_set[signer_id].VRFPK
        if pk == null: continue
        result = VRF_Verify(pk, alpha, proof)   ; per vrf-r255
        if result == INVALID: continue
        gamma = result.output
        if best_gamma == null or gamma < best_gamma
           or (gamma == best_gamma and signer_id < best_proposer):
            best_gamma = gamma; best_proposer = signer_id
    return best_proposer
```

### B.2 Batch Quorum Verification

```
function VerifyQuorum(block, Q):
    bitcount = popcount(block.PrepareSigsBitmap)
    if bitcount < Q: return ErrQuorumNotMet
    if bitcount != len(block.PrepareSigsPayload): return ErrMalformed
    idx = 0
    for i in 0 .. len(block.Validators)-1:
        if bit(block.PrepareSigsBitmap, i) == 1:
            pk = ML-DSA65_PKFromBytes(block.Validators[i])
            if not ML-DSA65_Verify(pk, H(B), block.PrepareSigsPayload[idx], ctx=""):
                return ErrInvalidSignature
            idx += 1
    return nil
```

### B.3 Rank-One Laplacian Update

```
function IncrementalLaplacianUpdate(L_old, node_status_changes):
    L_new = L_old
    for (i, j, delta) in node_status_changes:
        L_new[i][i] += delta; L_new[i][j] -= delta
        L_new[j][i] -= delta; L_new[j][j] += delta
    return L_new
```

---

## Appendix C — v2.0 → v3.0 Change Log

| # | Change | v2.0 | v3.0 |
|---|--------|------|------|
| C1 | ML-DSA-65 signature size | 2,700 bytes (round-3 Dilithium3 value) | **3,309 bytes** (FIPS 204 Table 2) |
| C2 | Algorithm naming | "Dilithium3" | **ML-DSA-65** (FIPS 204 name) |
| C3 | CBOR float encoding | fixed 64-bit IEEE 754 | **shortest round-trip** (RFC 8949 §4.2.1); NaN/Inf forbidden |
| C4 | VRF suite | mislabelled edwards25519 / bespoke 2-point map | **ECVRF-RISTRETTO255-SHA512** (C2SP `vrf-r255`) |
| C5 | VRF nonce | HMAC-SHA512 with sk as key (non-standard) | **domain-separated `vrf-r255` nonce** |
| C6 | Block hash | no domain tag, no NetworkID | **`3CP-BLOCKHASH-v3` tag + `NetworkID` binding** |
| C7 | SMT | `"leaf"`/`""` tags, no separation | **explicit leaf/node/empty domain tags** |
| C8 | Key rotation | ambiguous "BLAKE3 of the payload" | **exact domain-separated preimage (§8.2)** |
| C9 | Cycle duration | "3.000ms / 10.000ms" (unit error) | **3 s / 10 s** |
| C10 | Identity entropy | "≥128 bits, reject <80" (untestable) | **32-byte OS CSPRNG input** |
| C11 | HKDF | `L` unspecified | **`L = 32` fixed** |
| C12 | PrepareSigsBitmap | bit order unspecified | **MSB-first within each byte, high bits zero** |
| C13 | Reserved key 8 | three conflicting treatments across drafts | **always null in v3.0; reject non-null** |
| C14 | Ed25519 suites | cited alongside Ristretto255 | **RFC 9380 §8.5 prohibition acknowledged** |
| C15 | Hedged signing | unspecified | **hedged signing SHOULD (FIPS 204 §3.4)** |
| C16 | ML-KEM rejection | unspecified | **FIPS 203 §7.3 implicit rejection** |
| C17 | Forward compatibility | unspecified | **ignore unknown keys > 18** |
| C18 | `NetworkID` / `BlockHash` ordering | circular (genesis hash feeds its own preimage) | **`NetworkID` computed with key 11 omitted; fixed genesis ordering (§11.1)** |
| C19 | Block `Validators` (key 9) | `[* bytes .size 1952]` | **`[* validator-info]`** (upstream #2) — self-describing set with ML-DSA-65 + VRF keys |
| C20 | Block `Metadata` | referenced by degraded mode but never defined | **key 19 `Metadata`** (upstream #2) — declared home of `3cp:degraded-block` |
| C21 | `Triad` (key 4) / key 8 | mandatory members | **optional members (`?`)** (upstream #2) |
| C22 | Mandates | underspecified | **§13** full normative section (upstream #3) — authenticity, enforcement, detection |

---

## Appendix D — Corrections Applied in v3.0

This appendix documents, for each cryptographic correction, what the prior drafts said, why it was wrong, and what the corrected canon is.

**D1 — Signature size (2700 → 3309).** All prior drafts asserted a 2,700-byte Dilithium3 signature while simultaneously citing FIPS 204. 2,700 bytes is the signature size of the *round-3 draft* Dilithium3; FIPS 204 Table 2 fixes ML-DSA-65 at **3,309 bytes** (44→2,420; 65→3,309; 87→4,627). This was verified empirically against OpenSSL 3.6, whose ML-DSA-65 signatures are exactly 3,309 bytes. Because signature size is embedded in every CDDL `.size` constraint, the old value would have caused every conformant verifier to reject every valid signature.

**D2 — Canonical float encoding.** v1.0/v2.0 mandated fixed 64-bit floats. RFC 8949 §4.2.1 requires the *shortest* round-trip representation and forbids NaN/Infinity. Fixed-width floats produce non-canonical CBOR, breaking byte-exact `BlockHash` agreement.

**D3 — VRF suite.** v2.0 named "ECVRF-EDWARDS25519-SHA512-Elligator2" (a cofactor-8 edwards25519 suite) while also claiming a Ristretto255 curve, and separately proposed a "2-point Elligator2 sum" matching no standard. RFC 9380 §8.5 explicitly forbids using the edwards25519 suites to hash to ristretto255. v3.0 adopts the published `ECVRF-RISTRETTO255-SHA512` suite and standard domain-separated nonce derivation; byte-exact test vectors are provided so a second implementation can verify interoperability without reference source.

**D4 — BlockHash commitment.** v2.0's `BlockHash` committed to `Index, PrevHash, StateRoot, Proposer, HashOfAnchoredEntries, Timestamp, QuorumConfig` but **not** to `NetworkID`, `ProtocolVersion`, or the PREPARE/COMMIT signature fields (keys 13–15). This left the block identifier under-committed: a block's identity did not bind it to its network, and quorum evidence was outside the hash. v3.0 adds the `3CP-BLOCKHASH-v3` domain tag and the `NetworkID` binding, and §5.5/§5.6 make explicit which commitments the block hash and the two signature fields each cover.

**D5 — Signature circularity.** v2.0 defined `B_final = B || bitmap || payload || CommitSig` with `CommitSig = Sign(H(B_final))`, which is circular (the signature is defined over a preimage that contains itself). v3.0 defines PREPARE signatures over `H(B)` (keys 0–12) and `CommitSig` over `B_final` excluding key 15, removing the self-reference.

**D6 — Reserved key 8.** v2.0 drafts disagreed: one made key 8 carry PREPARE signatures, another reserved it. v3.0 unambiguously reserves key 8 (always null) and rejects non-null key 8, with PREPARE signatures exclusively in keys 13/14.

**D7 — Entropy testability.** v2.0 required rejecting an "estimated entropy < 80 bits" buffer — untestable, since implementations cannot measure buffer entropy. v3.0 specifies a 32-byte input from an OS CSPRNG, which is testable and FIPS-aligned.

**D8 — Cycle-duration unit error.** v2.0 Portuguese text specified 3–10 **millisecond** cycles while prose and v1.0 specified seconds. Millisecond consensus cycles are physically infeasible; v3.0 fixes the unit to seconds.

**D9 — HKDF output length.** v2.0 used `HKDF-SHA256` without the `L` parameter, which is a required input to RFC 5869. v3.0 fixes `L = 32`.

**D10 — Bitmap bit order.** v2.0's bitmap lacked a bit-order convention. v3.0 specifies MSB-first within each byte with high (out-of-range) bits zero, and the example `0xE0` is disambiguated.

**D11 — `NetworkID` / `BlockHash` circularity.** v2.0 defined `NetworkID` as the hash of the genesis block while (in the corrected construction) the genesis `BlockHash` binds `NetworkID` — a fixed point with no defined solution. v3.0 computes `NetworkID` over the genesis block with key 11 omitted and fixes the genesis construction order in §11.1, which terminates the recursion.

---

**Final note:** This document is a protocol specification. No implementation source, build script, CLI, or executable artifact is part of this repository. Implementations are developed in separate repositories under their own governance.