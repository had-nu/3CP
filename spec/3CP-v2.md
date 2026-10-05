> **NOTE:** This document specifies 3CP Protocol v2.0. New implementations should conform to v2.0.

---
title: 3CP — Cryptographic Chain-of-Custody Protocol
description: An application-layer protocol for producing, preserving, and verifying cryptographic chain-of-custody evidence across distributed systems
author: André Ataíde
status: Draft
type: Standards Track
created: 2026-07-28
protocol-version: 2
---

# 3CP (Cryptographic Chain-of-Custody Protocol) v2.0

**PROTOCOL_VERSION**: `2`

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## 1. Scope and Normative Language

**Design Principles:**
1. **Accountability without good faith:** Conformance cannot depend on the audited entity's good faith.
2. **Independent verifiability:** Any party with access to the public chain MUST be able to verify integrity, quorum, and genealogy without trusting the network operator.
3. **Finality by cycle:** Each consensus cycle produces zero or one final block. Final blocks are immutable and irreversible.
4. **On-chain governance:** Network operating rules (quorum, latency, publication) are declared in Mandates anchored in the chain itself.

---

## 2. Changelog: v1.0 to v2.0

| # | Change | v1.0 | v2.0 |
|---|--------|------|------|
| 1 | **Consensus** | VRF election plus M-of-N co-signing, no defined phases | Two atomic phases (PREPARE/COMMIT) with a fixed `ceil(2N/3)` quorum |
| 2 | **ValidatorSet** | Network state held only `UID` and `Status` | Network state MUST hold `Dilithium3PK` and `VRFPK` for every validator |
| 3 | **Key rotation** | Unspecified, out of band | A protocol operation via `3cp:key-rotation:v1` |
| 4 | **Block format** | v1.0 fields | Additional v2.0 fields: `ProtocolVersion`, `PrepareSigsBitmap`, `PrepareSigs`, `CommitSig`, `ExternalAnchors`, `KeyRotationEpoch`, `LegacyAnchor`, `Metadata` |
| 5 | **Cycle** | Fixed at 3s, entries discarded on timeout | Adaptive (`BaseInterval + EWMA(RTT)`), entries retained on abort |
| 6 | **Laplacian λ₁** | Full recomputation every `LambdaInterval` | Incremental rank-one update when the topology is unchanged |
| 7 | **Verifiability** | Theoretical ("public chain") | A Light Client protocol and Anchor Publishers are mandatory |
| 8 | **ZK** | A conceptual mention of CARCOSA | A versioned `ZKBridge` interface (v1.0.0) specified as a protocol contract |
| 9 | **Identity** | 32-byte `FEntropy`, derivation not normalised | Derivation via `HKDF-SHA256(salt=NetworkID, ...)` with a minimum of 128 bits of entropy |
| 10 | **Conformance** | 33 operational tests | Suite extended with adversarial BFT tests (12 minimum cases) |

---

## 3. System Model and Entities

### 3.1 Entities

- **Submitter:** Entity submitting a `ProvenanceEntry` for anchoring. Need not be a validator.
- **Validator:** Entity participating in consensus, producing and co-signing blocks. Each validator possesses a Dilithium3 key pair and a VRF key pair.
- **Anchor Publisher:** Entity (validator or external service) publishing final blocks to public-readable storage.
- **Light Client:** Entity verifying blocks and SMT proofs without participating in consensus or maintaining full state.
- **Auditor:** Entity verifying Mandate conformance against the anchored entry chain.

### 3.2 Threat Model

- **A1 — Network adversary:** Observes, delays, drops, or reorders messages. Does not control validator keys.
- **A2 — Compromised operator:** Controls infrastructure of one or more nodes, but less than quorum (`< ceil(2N/3)`).
- **A3 — Compromised submitter:** Controls submission credentials. Can submit false entries but cannot forge validator signatures.
- **A4 — External adversary:** No key access. May attempt DoS or replay.
- **A5 — Quantum adversary:** Possesses a quantum computer capable of breaking classical cryptography. 3CP v2.0 uses post-quantum primitives (Dilithium3, ML-KEM) to resist A5.

---

## 4. Cryptographic Primitives

### 4.1 Hash Functions

| Function | Output | Use |
|----------|--------|-----|
| `BLAKE3-256` | 32 bytes | Entry hashes, SMT leaves, SMT nodes, block identifiers |
| `SHA-256` | 32 bytes | `BlockHash` (block identifier) |
| `HKDF-SHA256` | Variable | Key derivation, UID0 seeds |

### 4.2 Signatures

**Algorithm:** ML-DSA-65 (Dilithium3), per FIPS 204.

- Public key size: 1,952 bytes
- Private key size: 4,032 bytes
- Signature size: 3.309 bytes
- NIST security level: 3 (AES-192 equivalent)

**[MUST]** All protocol signatures use Dilithium3 unless explicitly stated otherwise.

### 4.3 VRF (Verifiable Random Function)

**Algorithm:** ECVRF over Ristretto255, suite string `ristretto255_XMD:SHA-512_R255MAP_RO_`.

- Curve: Ristretto255 (prime-order, cofactorless)
- Internal hash: SHA-512
- Public key size: 32 bytes
- Private key size: 32 bytes
- Proof size: 96 bytes (`Gamma || C || S`)
- Output `Gamma` size: 32 bytes

**[MUST]** `HashToCurve(alpha)` is computed as:

```
buf   = expand_message_xmd(SHA-512, alpha, "ristretto255_XMD:SHA-512_R255MAP_RO_", 64)
h0    = buf[0:32]
h1    = buf[32:64]
p0    = Elligator2Map(h0)
p1    = Elligator2Map(h1)
Gamma_base = p0 + p1
```

This is a 2-point Elligator2 sum, not the single-point map of RFC 9381 §4.1.2. **[MUST]** Every conformant implementation MUST replicate exactly this 2-point procedure for interoperability.

**[MUST]** Nonce derivation in the Schnorr proof MUST be deterministic via HMAC-SHA512 with the VRF private key as HMAC key. Challenge and response follow Schnorr-style construction (not IETF ECVRF-EDWARDS25519 bitwise challenge/response).

### 4.4 KEM (Key Encapsulation Mechanism)

**Algorithm:** ML-KEM-1024 (Kyber1024), per FIPS 203.

- Public key size: 1,568 bytes
- Ciphertext size: 1,568 bytes
- Shared secret size: 32 bytes

**[MUST]** Used exclusively for secure transport handshake between peers (Kyber encapsulate/decapsulate + AEAD).

### 4.5 AEAD

**Algorithm:** ChaCha20-Poly1305 (RFC 8439).

**[MUST]** Used for transport encryption after Kyber handshake. The 32-byte key is derived via HKDF-SHA256 from the Kyber shared secret.

---

## 5. Block Format v2.0

### 5.1 Encoding

**[MUST]** All blocks and entries are serialized in canonical CBOR (RFC 8949), per CBOR section 4.2.1.

### 5.2 Block Structure

```
Block {
    ;; v1.0 fields (retained, unchanged semantics)
    0  => uint64,                    ; Index
    1  => bytes .size 32,            ; PrevHash (BLAKE3-256)
    2  => bytes .size 32,            ; StateRoot (SMT root, BLAKE3-256)
    3  => bytes .size 16,            ; Proposer (leader RootID)
    4  => [* bytes],                 ; Triad (reserved)
    5  => [* provenance-entry],       ; Anchored entries
    6  => float64,                    ; Lambda1 (Fiedler eigenvalue)
    7  => int64,                      ; Timestamp (UnixNano)
    8  => null,                       ; RESERVED in ProtocolVersion == 2 (was Sigs in v1.0)
    9  => [* validator-info],        ; Validators (canonical set, see §8.1)
    10 => quorum-config,              ; Quorum
    11 => bytes .size 32,            ; BlockHash (SHA-256)

    ;; v2.0 fields (new, MUST in v2 networks)
    12 => uint16,                     ; ProtocolVersion (default: 2)
    13 => bytes,                      ; PrepareSigsBitmap (bitfield, N bits)
    14 => [* bytes .size 3309],       ; PrepareSigsPayload (active signers only)
    15 => bytes .size 3309,           ; CommitSig (leader's commit signature)
    16 => [* tstr],                   ; ExternalAnchors (publication URIs/CIDs)
    17 => uint64,                     ; KeyRotationEpoch (reference cycle for active keys)
}
```

### 5.3 Signature Field Semantics

**[MUST]** Key 8 (`Sigs` in v1.0) is **reserved and MUST NOT be used** in blocks with `ProtocolVersion == 2`. A block with `ProtocolVersion == 2` and key 8 present and non-null **MUST** be rejected with `ErrReservedFieldPresent`.

**[MUST]** `PrepareSigsBitmap` (key 13) is a bitfield where bit `i` is `1` iff validator at index `i` in `Validators` signed PREPARE.

**[MUST]** `PrepareSigsPayload` (key 14) contains only signatures of validators whose bit in `PrepareSigsBitmap` is `1`, in ascending index order. It is the sole PREPARE-signature field in `ProtocolVersion == 2`.

**[MUST]** `CommitSig` (key 15) is the leader's Dilithium3 signature over the final block hash `H(B_final)` (including `PrepareSigsBitmap` and `PrepareSigsPayload`).

**[MUST]** A node operating with `ProtocolVersion == 1` (legacy network) continues to interpret key 8 per `spec/3CP.md` semantics. Key 8 never has two meanings within the same network: v1→v2 migration is a hard fork (§16), not a gradual field-semantics transition.

### 5.4 BlockHash Calculation

**[MUST]** `BlockHash` is computed as:

```
BlockHash = SHA-256(
    LE64(Index)
    || PrevHash
    || StateRoot
    || Proposer
    || HashOfAnchoredEntries
    || LE64(Timestamp)
    || QuorumConfigCanonical
)
```

**[MUST]** `HashOfAnchoredEntries` is the BLAKE3-256 hash of the canonical CBOR serialization of the array in key 5 (`Anchored`). This allows verifying entry integrity without retaining the full block.

---

## 6. Two-Phase BFT Consensus

### 6.1 Consensus Cycle

A **cycle** is the atomic time unit of the protocol. Each cycle `c` produces zero or one final block of index `c`.

**[MUST]** The cycle is divided into two atomic phases: PREPARE and COMMIT.

### 6.2 PREPARE Phase

**Input:** Network state at cycle `c` start, pending entries `E`, validator set `V`.

**Step 1 — Leader Election:**

```
alpha_c = c || StateRoot_at_cycle_start
```

For each validator `v_i ∈ V`:
```
proof_i = VRF_Sign(sk_i^VRF, alpha_c)
gamma_i = VRF_ProofToHash(proof_i)
```

The leader is the validator with the lowest `gamma_i` (lexicographic comparison of 32 bytes). Ties (extremely unlikely with Ristretto255) broken by lowest `ValidatorID` lexicographically.

**[MUST]** `StateRoot_at_cycle_start` is the SMT root **before** inserting any entries of cycle `c`. `alpha` is fixed at cycle start and does not change during phases.

**Step 2 — Proposal:**
The leader proposes candidate block `B` containing:
- `Index = c`
- `PrevHash = H(B_{c-1})`
- `StateRoot = Root(SMT_after_inserting_E)`
- `Proposer = leader.ValidatorID`
- `Anchored = E`
- `Lambda1 = λ₁` (if computed this cycle)
- `Timestamp = now()`
- `Validators = [v_0.pk, v_1.pk, ..., v_{N-1}.pk]`
- `Quorum = {TotalValidators: N, RequiredSigs: ceil(2N/3)}`

**Step 3 — PREPARE Voting:**
Each validator `v_j` receives `B` and verifies:
1. `B.PrevHash == H(B_{c-1})` accepted locally.
2. `B.StateRoot == Root(SMT_after_inserting_E)` computed locally.
3. Each `e ∈ E` passes `validateEntry(e)`.
4. `B.Lambda1 >= MinLambda1` (if applicable).
5. `B.ProtocolVersion == 2` (in v2 networks).

If all checks pass, `v_j` signs `H(B)` and broadcasts `PREPARE-SIG_j`.

### 6.3 COMMIT Phase

**Step 4 — Quorum Collection:**
The leader collects `PREPARE-SIG_j` until reaching `Q = ceil(2N/3)`.

**[MUST]** If the leader fails to collect `Q` signatures within `CycleTimeout`, the cycle is **aborted**. Entries `E` remain pending. Cycle `c` produces no block.

**Step 5 — Final Block Construction:**
```
B_final = B || PrepareSigsBitmap || PrepareSigsPayload || CommitSig_leader
```
Where `CommitSig_leader = Dilithium3_Sign(sk_leader, H(B_final))`.

**Step 6 — Broadcast and Validation:**
The leader broadcasts `B_final`. Each validator `v_j` verifies:
1. `PrepareSigsBitmap` contains at least `Q` bits set to `1`.
2. Each signature in `PrepareSigsPayload` verifies against the corresponding validator's public key in `Validators`.
3. `CommitSig_leader` verifies against `B.Proposer`.

If all checks pass, `v_j` accepts `B_final` as the final block of cycle `c`, appends to local chain, and transitions network state.

### 6.4 Aborted Cycle

**[MUST]** If a cycle aborts (PREPARE timeout, insufficient quorum, or `λ₁ < MinLambda1`):
- No block is appended.
- Entries `E` remain in the pending queue.
- Cycle `c+1` starts with new VRF election using `alpha_{c+1} = (c+1) || H(B_{c-1})`.

### 6.5 Degraded Mode

**[MUST]** If `N < 4`, the protocol operates in `degraded` mode:
- `Q = 1` (any single signature suffices).
- Each block MUST contain `Metadata["3cp:degraded-block"] = true`.
- Exiting `degraded` mode requires `N >= 4` and `GraceCycles` consecutive cycles (default: 10) with normal quorum.

---

## 7. Network State and ValidatorSet

### 7.1 GenesisValidatorSet

**[MUST]** The genesis block (index 0) contains a `GenesisValidatorSet`:

```
GenesisValidatorSet = [* validator-info]

validator-info = {
    0 => bytes .size 16,    ; ValidatorID (RootID)
    1 => bytes .size 1952,   ; Dilithium3PK
    2 => bytes .size 32,    ; VRFPK
    3 => bytes .size 32,    ; ContractHash (optional)
}
```

**[MUST]** The `GenesisValidatorSet` is immutable. Validator set changes (addition, removal, key rotation) occur via subsequent protocol entries.

### 7.2 NodeState

**[MUST]** Network state (`NetworkState`) maintains for each known node:

```
NodeState = {
    0 => bytes .size 16,    ; UID (RootID)
    1 => float64,           ; Status
    2 => uint64,            ; Consecutive
    3 => bytes .size 1952,  ; Dilithium3PK (NEW v2.0)
    4 => bytes .size 32,    ; VRFPK (NEW v2.0)
}
```

**[MUST]** Verification of a peer's `VRFProof` MUST use the `VRFPK` from the peer's `NodeState`, never the verifier's local identity.

---

## 8. Key Rotation

### 8.1 EventClass `3cp:key-rotation:v1`

**[MUST]** The protocol defines a key-rotation entry:

```
key-rotation-entry = {
    ;; Inherits base provenance-entry structure
    0 => bytes .size 32,    ; Hash (BLAKE3-256 of payload)
    1 => bytes .size 16,    ; Submitter (ValidatorID)
    2 => int64,              ; Timestamp
    3 => tstr,               ; Label: "3cp:key-rotation:v1"

    ;; Specific fields
    20 => bytes .size 1952,  ; NewPublicKey (Dilithium3)
    21 => bytes .size 32,    ; NewVRFPublicKey
    22 => uint64,             ; EffectiveCycle
    23 => uint64,             ; ExpiryCycle
    24 => bytes .size 3309,  ; SignatureOld (Dilithium3 with old key)
    25 => bytes .size 3309,  ; SignatureNew (Dilithium3 with new key)
}
```

### 8.2 Validation Rules

**[MUST]** A `key-rotation-entry` is valid iff:
1. `SignatureOld` verifies against the `Submitter`'s active `Dilithium3PK` in the current cycle's state.
2. `SignatureNew` verifies against `NewPublicKey`.
3. `EffectiveCycle >= currentCycle + KeyRotationLeadTime` (default: 10).
4. `ExpiryCycle >= EffectiveCycle + MinKeyOverlap` (default: 10).
5. `EffectiveCycle > lastRotationCycle` of the same validator (prohibits overlapping rotations).

### 8.3 Overlap Period

During `[EffectiveCycle, ExpiryCycle]`:
- Both old and new keys are accepted for block verification.
- After `ExpiryCycle`, only the new key is accepted for new signatures.

**[MUST]** Blocks prior to `EffectiveCycle` remain verifiable with the old key indefinitely.

---

## 9. Sparse Merkle Tree (SMT)

### 9.1 Parameters

- **Depth:** 256
- **Leaf hash:** `BLAKE3("leaf" || key || value)`
- **Node hash:** `BLAKE3(left || right)`
- **Key:** `entry.Hash` (32 bytes)
- **Value:** `entry.Hash` (32 bytes)

### 9.2 Inclusion Proof

**[MUST]** An SMT proof is an array of 256 32-byte hashes (8,192 bytes total), representing siblings on the path from leaf to root.

### 9.3 Verification

```
function VerifySMTProof(root, key, value, proof):
    current = BLAKE3("leaf" || key || value)
    for depth = 0 to 255:
        sibling = proof[depth]
        bit = (key[depth / 8] >> (depth % 8)) & 1
        if bit == 0:
            current = BLAKE3(current || sibling)
        else:
            current = BLAKE3(sibling || current)
    return current == root
```

---

## 10. Adaptive Cycle and Entry Retention

### 10.1 Cycle Duration

**[MUST]** Cycle duration is adaptive:

```
CycleDuration = BaseInterval + NetworkLatencyEstimate
NetworkLatencyEstimate = EWMA(RTT) * SafetyFactor
```

Where:
- `BaseInterval`: default 3,000ms (configurable by Mandate)
- `SafetyFactor`: default 1.5 (configurable by Mandate)
- `MaxCycleDuration`: 10,000ms (protocol hard cap)
- `EWMA(RTT)`: exponential moving average of round-trip time between peers

### 10.2 Pending Entry Retention

**[MUST]** Pending entries are **never discarded** due to cycle expiration.

**[MUST]** Entries pending for more than `MaxPendingTTL` cycles (default: 100) are rejected with error `ErrPendingExpired`.

### 10.3 Empty Cycle

**[MAY]** If `SkipEmptyCycles == true` (configurable by Mandate) and no entries are pending, the cycle may be skipped without block production.

---

## 11. Laplacian λ₁ Calculation

### 11.1 Definition

The network graph is represented by adjacency matrix `A`, where `A[i][j] = Status_j` if node `i` knows node `j`. Laplacian `L = D - A`, where `D` is the diagonal degree matrix.

`λ₁` is the smallest non-zero eigenvalue of `L` (Fiedler eigenvalue).

### 11.2 Incremental Update

**[MUST]** If no nodes were added or removed between consecutive cycles, and only `Status` values of existing nodes changed, the implementation MUST use **rank-one update** on the Laplacian instead of full reconstruction.

**[MUST]** Full Laplacian recomputation is only permitted when:
- A node is added or removed.
- `LambdaInterval` cycles have passed without full recomputation.
- State was marked `dirty` by structural change.

### 11.3 Algorithm for N > 100

**[MUST]** For `N > 100`, the protocol permits approximation via **Lanczos method** with iterations:

```
k = min(50, max(30, floor(N / 10)))
```

**[MUST]** Lanczos stopping criterion MUST include Ritz value convergence check. If convergence occurs before `k` iterations, computation MAY stop early.

### 11.4 Fragmentation

**[MUST]** If `λ₁ < MinLambda1` and `N >= 2`, the current cycle is aborted and the network enters `fragmented` state until `λ₁` recovers or nodes are removed.

---

## 12. Third-Party Verifiability

### 12.1 Anchor Publishers

**[MUST]** An Anchor Publisher publishes final blocks to public-readable storage:

```
AnchorPublisherConfig = {
    0 => tstr,               ; Mode: "all" / "designated" / "external"
    1 => [* bytes .size 16], ; DesignatedPublishers (only if Mode == "designated")
    2 => uint,                ; MinRedundancy (default: 2)
}
```

**[MUST]** Mandatory backends:
- Local filesystem
- IPFS (CIDv1, codec `raw`, hash `blake3-256` or `sha2-256`)
- S3-compatible blob store (with SHA-256 checksum)

### 12.2 Light Client Protocol

**[MUST]** The protocol defines read operations for light clients:

```
GetBlock(index: uint64) -> Block
StreamBlocks(startIndex: uint64) -> stream Block
GetValidatorSet(cycle: uint64) -> [* validator-info]
GetMerkleProof(key: bytes .size 32, blockIndex: uint64) -> SMTProof
```

**[MUST]** A light client verifies a block `B` without executing consensus:
1. Obtains `ValidatorSet` for cycle `B.Index`.
2. Verifies `B.PrepareSigsPayload` contains `Q = ceil(2N/3)` valid signatures against `ValidatorSet` keys (using `PrepareSigsBitmap` to map each entry to its validator).
3. Verifies `B.CommitSig` against `B.Proposer`.
4. Verifies the `PrevHash` chain.

---

## 13. ZK Bridge Protocol

### 13.1 ZKBridge v1.0.0 Interface

**[MUST]** The protocol defines a stable interface for zero-knowledge proof consumers:

```
ZKBridge v1.0.0 {
    GetBlockRange(start: uint64, end: uint64) -> [* Block]
    GetMerkleProof(key: bytes .size 32, blockIndex: uint64) -> SMTProof
    GetValidatorSet(cycle: uint64) -> [* validator-info]
}
```

**[MUST]** Breaking changes require major version bump. Backward compatibility MUST be maintained for at least 2 major versions.

### 13.2 Hash Functions for ZK Circuits

**[SHOULD]** ZK circuits consuming 3CP hashes SHOULD use STARK-friendly hash functions (e.g., Poseidon2, Rescue-Prime) for internal circuit hashing, keeping BLAKE3 for external hashing only.

---

## 14. UID0 Identity Derivation

### 14.1 NetworkID

**[MUST]** `NetworkID` is the BLAKE3-256 hash of the genesis block:

```
NetworkID = BLAKE3-256(GenesisBlock)
```

**[MUST]** `NetworkID` is immutable for the chain's lifetime.

### 14.2 Seed Derivation

**[MUST]** The 32-byte seed for identity derivation is:

```
seed = HKDF-SHA256(
    salt = NetworkID,
    info = "3cp-uid0-v2",
    ikm = entropySource
)
```

**[MUST]** `entropySource` MUST contain at least 128 bits of effective entropy. Implementations SHOULD reject `entropySource` with estimated entropy < 80 bits.

---

## 15. Mandates

### 15.1 Structure

See `spec/schemas/mandate.cddl` for the complete CDDL definition.

**[MUST]** A mandate is a signed, versioned, anchored declaration that defines anchoring obligations. `Authority` is the RootID that issued it, `Version` and `PrevVersion` form the revision lineage, and `Rules` declares the event classes it covers.

**[MUST]** A mandate's `ID` is BLAKE3-256 of the canonical CBOR of the mandate with `ID` and `Signature` excluded. An implementation MUST recompute it and MUST NOT trust the stored value: a body whose rules differ from the identifier it presents MUST be rejected.

### 15.2 Authenticity

**[MUST]** `Signature` (key 10) is a Dilithium3 signature by the `Authority` over that same canonical payload, with `ID` and `Signature` excluded.

**[MUST]** An implementation MUST verify this signature before accepting a mandate as binding policy, and MUST resolve it against a registered ML-DSA-65 key -- the ValidatorSet or network state. A mandate whose `Authority` does not resolve to a known key MUST be rejected, and an absent or mis-sized signature MUST be rejected before any expensive verification.

Without this check `Authority` is only an assertion: any submitter could install a mandate in another node's name, and the network would treat its rules as authentic policy. A missing signature MUST be treated as an authorisation failure, not as the requirement not applying.

**[MUST]** The validity window (`ValidFrom`, `ValidUntil`) MUST NOT be enforced at admission. An auditor must be able to load an expired mandate to assess a window it covered; refusing it at admission would erase the record the audit is looking for.

### 15.3 Submission-Time Enforcement

**[MUST]** An entry carrying `MandateRef` MUST be checked against the mandate it references: the mandate MUST exist, be in force at the entry's timestamp, and have every field its rules require.

An entry with no `MandateRef` makes no claim and MUST be accepted. Most entries in a provenance chain are governed by no mandate, and rejecting them would make the mechanism unusable.

This half of enforcement is structural: it stops claims of compliance that fail the most basic requirements. It CANNOT detect omitted events, because the protocol never sees an event that was never submitted.

### 15.4 Verification and Enforcement

See `spec/schemas/mandate.cddl` for the `compliance-verification` and `compliance-gap` structures.

**[MUST]** An implementation MUST offer an operation that checks the chain against a mandate over a time window, comparing the anchored entries against the declared obligations.

**[MUST]** Omission MUST be cryptographically detectable: an event a mandate requires that was never anchored MUST produce a gap, distinguishable from an entry that is present but incomplete.

**[MUST]** Verification MUST NOT derive the validator set from the block being verified. The set has to come from an external trust anchor -- the genesis block, an Anchor Publisher, or a full node the operator vouches for. A block that names its own validators can be signed entirely by whoever wrote it.

### 15.5 Degraded Mode

**[MUST]** A mandate applies regardless of the operating mode. The quorum rule of §6.5 governs block finality, not anchoring obligations: a mandate MUST NOT be treated as satisfied with less than its own rules require, even when the chain is degraded.

---

## 16. Conformance Tests v2.0

**[MUST]** Implementations claiming 3CP v2.0 conformance MUST pass:

| ID | Category | Description |
|----|----------|-------------|
| TC-BFT-01 | Consensus | Leader proposes two distinct blocks → detected and slashed |
| TC-BFT-02 | Consensus | Invalid PREPARE signature → validator marked suspect |
| TC-BFT-03 | Consensus | `f < N/3` faults → consensus continues |
| TC-BFT-04 | Consensus | `f >= N/3` faults → liveness failure (not safety) |
| TC-NET-01 | Network | 5-cycle partition → reconciliation without forks |
| TC-NET-02 | Network | Latency > MaxCycleDuration → abort with retention |
| TC-ROT-01 | Keys | Valid rotation → both keys accepted in overlap |
| TC-ROT-02 | Keys | Rotation with too-early EffectiveCycle → rejected |
| TC-ZK-01 | ZK | SMT proof verifiable by light client |
| TC-PUB-01 | Publication | Block published → recoverable via ExternalAnchors |
| TC-SCA-01 | Scalability | 10,000 entries/min → sustained throughput |
| TC-MEM-01 | Durability | 1h continuous → state stability |

---

## 17. v1 → v2 Migration

**[MUST]** Migration from v1.0 to v2.0 is a **hard fork**.

**Procedure:**
1. Stop all v1.0 nodes at the same final block `B_final`.
2. Export: validator set, active mandates, SMT root, `H(B_final)`.
3. Create v2.0 genesis block with:
   - `InitialValidators` = exported validator set
   - `GenesisMandate` = mandates converted to v2 format
   - `LegacyAnchor` = `H(B_final)` (continuity proof)
4. Start v2.0 network from the new genesis.

---

## 18. Normative References

- FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard (ML-KEM)
- FIPS 204: Module-Lattice-Based Digital Signature Standard (ML-DSA)
- FIPS 205: Stateless Hash-Based Digital Signature Standard (SLH-DSA) — not used in 3CP, post-quantum reference
- RFC 2119: Key words for use in RFCs to Indicate Requirement Levels
- RFC 8439: ChaCha20 and Poly1305 for IETF Protocols
- RFC 8610: Concise Data Definition Language (CDDL)
- RFC 8949: Concise Binary Object Representation (CBOR)
- RFC 9381: Verifiable Random Functions (VRFs) — structural basis of 3CP ECVRF; see §4.3 for declared divergence in `HashToCurve` procedure

---

## 19. Appendix A: Complete v2.0 CDDLs

### A.1 Block (block-v2.cddl)
See `spec/schemas/block.cddl` (updated per §5).

### A.2 Network State (network-state.cddl)
See `spec/schemas/network-state.cddl`.

### A.3 Key Rotation (key-rotation.cddl)
See `spec/schemas/key-rotation.cddl`.

### A.4 Genesis (genesis.cddl)
See `spec/schemas/genesis.cddl`.

### A.5 Light Client (light-client.cddl)
See `spec/schemas/light-client.cddl`.

### A.6 ZK Bridge (zk-bridge.cddl)
See `spec/schemas/zk-bridge.cddl`.

## 20. Appendix B: Formal Algorithms

### B.1 Proposer Selection

```
function SelectProposer(proofs, alpha):
    best_gamma = 0xFF...FF  ; 32 bytes, max value
    best_proposer = null
    
    for each (signer_id, proof) in proofs:
        pk = LookupVRFPK(signer_id)  ; from NetworkState
        if pk == null: continue
        
        gamma, err = VRF_Verify(pk, alpha, proof)
        if err != null: continue  ; invalid proof
        
        if gamma < best_gamma:
            best_gamma = gamma
            best_proposer = signer_id
    
    return best_proposer
```

### B.2 Batch Quorum Verification

```
function VerifyQuorumBatch(blockHash, sigs, validators, required):
    if len(sigs) < required: return ErrQuorumNotMet
    
    valid = 0
    used_validators = set()
    
    for sig in sigs:
        for i, pk_bytes in validators:
            if i in used_validators: continue
            pk = Dilithium3_PKFromBytes(pk_bytes)
            if pk.Verify(blockHash, sig):
                valid++
                used_validators.add(i)
                break
    
    if valid < required: return ErrQuorumNotMet
    return nil
```

### B.3 Laplacian Rank-One Update

```
function IncrementalLaplacianUpdate(L_old, node_status_changes):
    L_new = L_old
    for (i, j, delta) in node_status_changes:
        ; delta = new_status - old_status
        L_new[i][i] += delta
        L_new[i][j] -= delta
        L_new[j][i] -= delta
        L_new[j][j] += delta
    return L_new
```