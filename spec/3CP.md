---
title: 3CP — Cryptographic Chain-of-Custody Protocol
description: An application-layer protocol for producing, preserving, and verifying cryptographic chain-of-custody evidence across distributed systems
author: André Ataíde
status: Draft
type: Standards Track
created: 2026-07-14
protocol-version: 1
---

# 3CP (Cryptographic Chain-of-Custody Protocol) v1

**PROTOCOL_VERSION**: `1`

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

**Conformance status**: This specification has been validated by a conformance
test suite (33 tests) that exercises every normative requirement at the protocol
boundary. 3CP has NOT yet been validated by an independent implementation in
another language. Any party building a second implementation does so at their
own risk of discovering untested edge cases. See section
[Conformance](#10-conformance) for details.

## 1. Overview

3CP is an **application-layer protocol** for producing, preserving, and
verifying cryptographic chain-of-custody evidence across distributed systems.
It enables any party — regulator, auditor, consumer, third-party verifier — to
independently verify that a given digital artifact existed at a given point in
time, that its provenance was attested by known identities, and that no
retroactive reconstruction of events is possible without detection. 3CP uses
post-quantum cryptography (Dilithium3, Kyber1024) with a VRF-based leader
election protocol, requires no tokens, mining, or smart contracts, and is
transport-agnostic (gRPC is the canonical transport, §2).

**Key distinction**: Verification of chain-of-custody evidence does not depend
on the node that produced the evidence. Any party in possession of a chain's
blocks can independently verify every SMT proof (§5.4), signature (§3.3), and
block hash (§3.2) without contacting the originating network. This is the
property that makes 3CP a contestability fabric, not merely an audit log.

## 2. Transport

gRPC (using Protocol Buffers over TCP) is the **canonical transport** for the
3CP wire protocol. A conformant peer implementation MUST expose the gRPC
service defined in section [gRPC API](#7-grpc-api). The message formats and
consensus rules defined in this specification are the normative protocol
boundary; the gRPC framework is the transport that realizes that boundary.

This decision may change in the future if a clearly superior transport emerges.
Until then, gRPC is canon because it works reliably, is well-specified, and
has mature cross-language tooling.

## 3. Cryptographic Primitives

### 3.1 Hash Function

All hashing uses **BLAKE3-256** (32-byte output) unless otherwise noted.

### 3.2 Block Hash

The block identifier (`BlockHash`) uses **SHA-256** over a fixed-length binary
preimage:

```
BlockHash = SHA-256(
    LE64(Index)       // 8 bytes, little-endian unsigned integer
  || PrevHash         // 32 bytes, BLAKE3-256 of previous block (all zeros for genesis)
  || StateRoot        // 32 bytes, SMT root after block insertions
  || Proposer         // 16 bytes, RootID of the proposer
  || anchored[0].Hash // 32 bytes, first entry hash
  || ...              // all remaining entry hashes concatenated (each 32 bytes)
  || LE64(Timestamp)  // 8 bytes, UnixNano in little-endian
)
```

The `anchored[*].Hash` entries MUST be in the same order as the block's
`Anchored` array. A conformant implementation MUST produce identical
`BlockHash` values for identical block content.

### 3.3 Post-Quantum Signatures — Dilithium3 (ML-DSA-65)

| Parameter | Value |
|-----------|-------|
| Algorithm | ML-DSA-65 (Dilithium3) |
| Public key size | 1952 bytes |
| Signature size | 2700 bytes |
| Security level | NIST Level 3 (AES-256 equivalent) |

Conformant implementations MUST use the same parameter set. Private keys are
never transmitted on the wire.

### 3.4 Key Encapsulation — Kyber1024 (ML-KEM-1024)

| Parameter | Value |
|-----------|-------|
| Algorithm | ML-KEM-1024 (Kyber1024) |
| Ciphertext size | 1568 bytes |
| Shared secret size | 32 bytes |
| Security level | NIST Level 5 (AES-256 equivalent) |

Used for peer-to-peer encrypted channels.

### 3.5 Verifiable Random Function — ECVRF (RFC 9381)

| Parameter | Value |
|-----------|-------|
| Curve | Ristretto255 |
| Suite string | `ristretto255_XMD:SHA-512_R255MAP_RO_` |
| Private key | 32 bytes (ristretto scalar) |
| Public key | 32 bytes (ristretto point) |
| Hash-to-curve | `expand_message_xmd(SHA-512, 64 bytes)` → split into two 32-byte halves → Elligator2 map → sum p0 + p1 |

**VRFProof structure** (96 bytes total):

```
Offset  Size  Field    Description
0       32    Gamma    VRF output — Ristretto255 point bytes (hash-to-curve result)
32      32    C        Fiat-Shamir challenge — scalar bytes
64      32    S        Schnorr response — scalar bytes
```

**Serialization**: `Gamma || C || S` concatenated as 96 bytes.

**Verification**: A conformant implementation MUST verify every received VRF
proof against the sender's VRF public key (32 bytes, Ristretto255 point). The
proof output `Gamma` is used for proposer selection: the peer with the lowest
`Gamma` (lexicographic byte comparison) is selected as the proposer for that
cycle.

### 3.6 CBOR Canonical Encoding

All CBOR-encoded structures in 3CP use canonical (deterministic) CBOR:

- Integer map keys (field keys) MUST be encoded in ascending order
- Integer keys MUST be encoded as the smallest representation (`uint` ≤ 23 fits in one byte)
- Strings and byte strings use definite-length encoding
- Float values use 64-bit IEEE 754 encoding

## 4. Block Structure

Blocks are serialized as CBOR with integer field keys.

### 4.1 Block

| Key | Field | Type | Description |
|-----|-------|------|-------------|
| 0 | `Index` | uint64 | Sequential block number, MUST increment by 1 |
| 1 | `PrevHash` | []byte (32) | BLAKE3-256 of previous block's `BlockHash`; zeros for genesis |
| 2 | `StateRoot` | []byte (32) | SMT root after all entries in this block are inserted |
| 3 | `Proposer` | []byte (16) | RootID of the proposer who built this block |
| 4 | `Triad` | [3][]byte | Reserved |
| 5 | `Anchored` | []ProvenanceEntry | Entries anchored in this block |
| 6 | `Lambda1` | float64 | Laplacian eigenvalue λ₁ at block time (network diffusion metric) |
| 7 | `Timestamp` | int64 | UnixNano at block finalization |
| 8 | `Sigs` | [][]byte | Dilithium3 signatures (each 2700 bytes) |
| 9 | `Validators` | [][]byte | Dilithium3 public keys (each 1952 bytes) of the validator set |
| 10 | `Quorum` | QuorumConfig | Threshold configuration |
| 11 | `BlockHash` | []byte (32) | SHA-256 per §3.2. Computed by proposer; verified by every peer before signing |

### 4.2 ProvenanceEntry

| Key | Field | Type | Description |
|-----|-------|------|-------------|
| 0 | `Hash` | [32]byte | The anchored hash (content-addressed identifier) |
| 1 | `Submitter` | []byte | Who submitted this entry (RootID or caller identifier) |
| 2 | `Timestamp` | int64 | Submission time in UnixNano |
| 3 | `Label` | string | Human-readable label (OPTIONAL, SHOULD be ≤ 256 bytes) |
| 4 | `Approver` | []byte | Who approved/accepted the residual risk (OPTIONAL) |
| 5 | `Reference` | []byte (32) | Hash of a related entry (OPTIONAL). Enables cross-entry linking |
| 6 | `Signature` | []byte (2700) | Dilithium3 signature from Submitter (and Approver, if present) over the entry content (OPTIONAL). See §7.2.1 for signed payload format |

### 4.3 QuorumConfig

| Key | Field | Type | Description |
|-----|-------|------|-------------|
| 0 | `TotalValidators` | int | Total number of validators in the network |
| 1 | `RequiredSigs` | int | Minimum number of valid signatures required to finalize a block |

Default values: `{TotalValidators: 3, RequiredSigs: 3}` for multi-node mode;
`{TotalValidators: 1, RequiredSigs: 1}` for single-node mode.

## 5. Sparse Merkle Tree (SMT)

### 5.1 Parameters

| Parameter | Value |
|-----------|-------|
| Hash function | BLAKE3-256 |
| Depth | 256 |
| Leaf format | `BLAKE3("leaf" \|\| key \|\| value)` where key and value are both 32 bytes |
| Parent format | `BLAKE3(left \|\| right)` where left and right are 32-byte hashes |
| Empty hash | `BLAKE3("")` |
| Bit ordering | LSB-first within each byte of the key |

### 5.2 Path Determination

```
For depth from 0 to 255:
    byteIdx = depth / 8
    bitIdx  = depth % 8
    bit     = (key[byteIdx] >> bitIdx) & 1
    If bit == 0: go left (sibling is right)
    If bit == 1: go right (sibling is left)
```

### 5.3 Proof Format

An SMT proof is a sequence of sibling hashes traversed from leaf to root.
Each sibling is 32 bytes (BLAKE3-256). The proof is serialized by
concatenating all siblings in order:

```
proofBytes = sibling[0] || sibling[1] || ... || sibling[n-1]
            // each 32 bytes, total = n * 32 bytes
```

For a key that does not exist (proof of absence), the leaf hash is computed
from the zero value (all-zero key, all-zero value). A proof of absence MUST
produce a sibling path that recomputes to the current StateRoot.

### 5.4 Verification

A verifier walks the proof from leaf to root:

1. Compute leaf hash: `BLAKE3("leaf" || key || value)`
2. For each depth 0 to 255:
   - Read the next 32-byte sibling from the proof
   - Determine direction from key bit (same algorithm as §5.2)
   - Compute parent: if going left, `BLAKE3(current || sibling)`; if right, `BLAKE3(sibling || current)`
   - Set current = parent
3. After 256 iterations, `current` MUST equal the claimed `StateRoot`

## 6. Consensus Cycle

### 6.1 Overview

One consensus cycle produces one anchored block with **instant finality** —
no forks, no rollbacks, no reorganization. The cycle is ticker-driven.

### 6.2 Single-Node Mode

A single-node network uses `QuorumConfig = {TotalValidators: 1, RequiredSigs: 1}`.
The single node acts as proposer and signer.

### 6.3 Multi-Node Mode

For multi-node (M-of-N) mode, the cycle proceeds as follows:

#### 6.3.1 VRF Proposer Selection

```
Input:  cycle (uint64, 8 bytes LE), stateRoot (32 bytes), each peer's VRF keypair
Output: proposer ID (RootID of peer with lowest Gamma)
```

1. Every peer computes `alpha = LE64(cycle) || stateRoot` (40 bytes)
2. Every peer computes `VRF.Prove(sk, alpha)` → `proof (Gamma || C || S, 96 bytes)`
3. Every peer publishes its VRF proof via gossip
4. Every peer collects all VRF proofs from the network
5. Every peer verifies each proof using the signer's VRF public key (32 bytes)
6. Proposer = peer whose `Gamma` field is lowest in lexicographic byte order
7. If any proof fails verification, the peer that produced it MUST be excluded
   from proposer selection for this cycle

#### 6.3.2 Block Proposal

The proposer:

1. Collects pending entries (from gossip snapshot or local queue)
2. Deduplicates by entry hash — if two entries have identical `Hash` values,
   only the first is kept
3. For each entry, inserts `(Hash, Hash)` into the SMT:
   - key = entry.Hash (32 bytes)
   - value = entry.Hash (32 bytes)
4. Computes `StateRoot` from the updated SMT
5. Computes `BlockHash` per §3.2
6. Signs `BlockHash` with its Dilithium3 private key → signature (2700 bytes)
7. Broadcasts the block + its signature via gossip

#### 6.3.3 Non-Proposer Verification

Each non-proposer peer:

1. Reads the proposer's block from gossip
2. Replays all entries into its local SMT (same insertion order)
3. Computes `localStateRoot`
4. Verifies `localStateRoot == block.StateRoot` — MUST reject if mismatch
5. Verifies `block.BlockHash` recomputation
6. Verifies proposer's signature on `block.BlockHash`
7. Signs `block.BlockHash` with its own Dilithium3 private key
8. Broadcasts its co-signature via gossip

#### 6.3.4 Quorum Verification

After collecting co-signatures:

1. For each signature in `block.Sigs`:
   a. Iterate through `block.Validators` (Dilithium3 public keys)
   b. Skip validators already credited
   c. For each non-used validator key, verify signature against `block.BlockHash`
   d. First match wins — credit that validator, mark pubkey as used
   e. If no match, skip the signature
2. After processing all signatures: if `usedValidatorCount < quorum.RequiredSigs`,
   the block MUST NOT be appended
3. A conformant implementation MUST count each distinct validator at most once

#### 6.3.5 State Transition

```
1. Update network topology: process heartbeat latencies
2. Recompute Laplacian λ₁
3. Increment cycle counter
4. Append block to chain
5. Persist state
```

### 6.4 Cycle Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `cycleInterval` | 3s | Time between automatic cycle ticks |
| `LambdaInterval` | 0 (every cycle) | Recompute λ₁ every N cycles |
| `Eta` | 0.28 | Diffusion rate for Laplacian supervision |
| `DecayRate` | 0.05 | Node status decay per cycle |
| `MinLambda1` | 0.10 | Minimum λ₁ threshold |

## 7. gRPC API

### 7.1 Methods

| Method | Description |
|--------|-------------|
| `SubmitHash` | Submit a provenance hash for anchoring. Returns a Ticket with submission status. **MUST authenticate the caller** — see §7.2.1 |
| `WaitForAnchor` | Block until a specific hash is anchored; returns the SMT proof and block index |
| `VerifyHash` | Check if a hash is already anchored |
| `GetCurrentStateRoot` | Return the current SMT root bytes |
| `GetHealth` | Return node health metrics (block height, pending hashes, λ₁) |
| `GetBlock` | Retrieve a full block by its index |
| `StreamBlocks` | Stream blocks from a start index to end index |

### 7.2 Entry Validation

A conformant server MUST reject submissions where:

| Condition | Error Code |
|-----------|------------|
| `Hash` is all zeros (zero hash) | `ZERO_HASH` |
| `Submitter` is empty | `EMPTY_SUBMITTER` |
| `Label` exceeds 1024 bytes | `LABEL_TOO_LONG` |
| `signature` is empty | `INVALID_SIGNATURE` |
| `signature` does not verify | `INVALID_SIGNATURE` |
| `Submitter` is not a recognised identity | `SUBMITTER_MISMATCH` |
| Submit rate exceeds limit | `RATE_LIMITED` |

### 7.2.1 Client Authentication

The `SubmitHash` endpoint MUST authenticate the caller before accepting
submissions.

**Authentication payload**:
```
signedPayload = Hash || Submitter || LE64(Timestamp) || Label
```

The caller MUST include a Dilithium3 signature over `signedPayload`. The server
MUST verify this signature against the Dilithium3 public key corresponding to
the claimed `Submitter` identity (RootID).

**Entry-level non-repudiation**: When the `ProvenanceEntry.Signature` field
(CBOR key 6) is populated, it carries the same signed payload format (optionally
extended to include `Approver` and `Reference`). A verifier can independently
confirm that the claimed identities signed the entry content.

### 7.3 Error Code Registry

| Code | Description |
|------|-------------|
| `ZERO_HASH` | Submitted hash is all zeros |
| `EMPTY_SUBMITTER` | Submitter field is empty |
| `LABEL_TOO_LONG` | Label exceeds 1024 bytes |
| `INVALID_SIGNATURE` | Signature missing or does not verify |
| `SUBMITTER_MISMATCH` | Submitter identity not recognised |
| `RATE_LIMITED` | Submission rate exceeds limit |

### 7.4 Rate Limiting

A conformant server SHOULD enforce rate limiting per submitter using a sliding
window algorithm. Default limit: 5000 submissions per minute per submitter.
When a submitter exceeds the limit, the server MUST return `RATE_LIMITED`.
Implementations SHOULD bound the tracked submitter map to prevent unbounded
memory growth.

## 8. UID0 Identity

### 8.1 Overview

UID0 is a soulbound identity token that binds a validator node to a
deterministic, verifiable identity. It is serialized as canonical CBOR.

### 8.2 CBOR Field Map

| Key | Field | Type | Requirements |
|-----|-------|------|-------------|
| 0 | `RootID` | []byte (16) | REQUIRED. Unique node identifier |
| 1 | `FEntropy` | []byte (16) | REQUIRED. Entropy seed for key generation |
| 2 | `GenesisHash` | []byte | OPTIONAL |
| 3 | `FounderMerkleProofs` | [][]byte | OPTIONAL |
| 4 | `SigRecovery` | []byte | OPTIONAL |
| 5 | `SigUpdate` | []byte | OPTIONAL |
| 6 | `SigAudit` | []byte | OPTIONAL |
| 7 | `ReputationSum` | uint64 | OPTIONAL |
| 8 | `SiderealTime` | string | OPTIONAL |
| 9 | `CycleIndex` | uint64 | OPTIONAL |
| 10 | `GeneratedAt` | int64 | OPTIONAL |
| 11 | `MerkleRoot` | []byte (32) | OPTIONAL |
| 12 | `MerkleProof` | [][]byte | OPTIONAL |
| 13 | `Simulated` | bool | OPTIONAL |
| 14 | `FinalDigest` | []byte (32) | REQUIRED. BLAKE3-256 of canonical CBOR excluding this field |
| 15 | `PublicKey` | []byte (1952) | REQUIRED. Dilithium3 public key |
| 16 | `SecretKey` | []byte | MUST NOT be exported or transmitted |
| 17 | `ContractHash` | []byte | OPTIONAL. Hash of company contract for node↔contract binding |
| 18 | `VRFPublicKey` | []byte (32) | REQUIRED. Ristretto255 VRF public key |
| 19 | `VRFSecretKey` | []byte (32) | MUST NOT be exported or transmitted |

### 8.3 FinalDigest Computation

```
FinalDigest = BLAKE3-256(canonicalCBOR(UID with FinalDigest set to nil/empty))
```

### 8.4 Field Addition Rule

Any implementation adding new fields MUST use additive integer keys (next
available integer). Existing field keys MUST NOT be reused or reassigned.

## 9. Sub-Chains

Each service MAY register a sub-chain with its own isolated SMT. The sub-chain's
state root is periodically anchored into the parent chain via a cross-chain
proof.

### 9.1 Sub-Chain Lifecycle

| Method | Description |
|--------|-------------|
| `Register(name, owner)` | Create a new sub-chain with its own SMT |
| `Submit(id, entry)` | Submit an entry to a sub-chain's SMT |
| `Anchor(id)` | Flush pending entries, compute state root, enqueue anchor entry in parent chain |
| `Prove(id, entryHash)` | Generate a cross-chain proof (dual-Merkle) |
| `VerifyCrossChain(proof, parentRoot)` | Verify an entry is anchored in a sub-chain AND the sub-chain root is anchored in the parent |

### 9.2 Cross-Chain Proof

A `CrossChainProof` contains:

| Field | Type | Description |
|-------|------|-------------|
| `EntryHash` | [32]byte | The entry's hash in the sub-chain |
| `SubChainRoot` | [32]byte | Sub-chain's SMT root at anchor time |
| `SubChainProof` | []byte | SMT proof within the sub-chain |
| `AnchorHash` | [32]byte | The anchor entry hash in the parent chain |
| `AnchorBlock` | uint64 | Block index where the anchor was finalized |
| `ParentRoot` | [32]byte | Parent chain's SMT root at anchor time |
| `ParentProof` | []byte | SMT proof within the parent chain |

## 10. Conformance

A conformant implementation of 3CP MUST:

- Implement the consensus cycle as described in §6
- Produce and verify VRF proofs per §3.5
- Produce and verify SMT proofs per §5
- Encode blocks and identity as canonical CBOR per §4 and §8
- Implement the gRPC service per §7
- Enforce entry validation per §7.2
- Handle both single-node and multi-node modes per §6.2 and §6.3

**Non-normative note**: As of this writing, only one known implementation exists.
This specification is prepared for the purpose of enabling a second, independent
implementation. Until such an implementation exists and demonstrates
interoperability, the specification may contain untested gaps.

## 11. Implementation Guidance for Gate Consumers (Non-Normative)

This section provides RECOMMENDED guidance for implementers building a release
gate or deployment pipeline that consumes 3CP as a decision-verification layer.

### 11.1 Recommended Call Sequence

```
1. SubmitHash(hash, submitter, label) → ticket
2. WaitForAnchor(hash) → AnchorProof (may block up to cycleInterval + margin)
3. GetBlock(proof.BlockIndex) → Block (confirm entry is in the chain)
4. Verify SMT proof against Block.StateRoot (client-side verification)
```

### 11.2 Timeout and Error Handling

- If `WaitForAnchor` times out or returns an error, the gate MUST fail closed.
- The timeout value SHOULD be at least `3 × cycleInterval` (default: 9 s).
- If `VerifyHash` returns `found: false`, the gate MAY wait and retry.

### 11.3 Network Health Pre-Check

- Before submitting, the gate SHOULD call `GetHealth` and check:
  - `status == "running"`
  - `active_peers >= quorum.RequiredSigs` (if using multi-node mode)
- If the network appears degraded, the gate SHOULD fail closed.

### 11.4 Custody Separation

- The 3CP signing identity SHOULD be stored in a separate trust domain from
  CI/CD credentials.
- If the same compromise leaks both, the non-repudiation guarantee is
  undermined.

### 11.5 Example (Pseudocode)

```
function gateRelease(artifactHash):
    identity = loadNodeIdentity("prod-gate")
    engine = newEngine(identity, cycleInterval=3s)
    engine.start()

    ctx = context.withTimeout(10s)
    ticket = engine.submit(ctx, artifactHash, identity.rootID, "release:gate")
    if error:
        return "submit failed, blocking release"

    proof = engine.waitForAnchor(ctx, ticket.hash)
    if error or not proof.found:
        return "anchor failed, blocking release"

    return "release approved"
```

## 12. Security Considerations

### 12.1 Threat Model

3CP assumes the following adversarial capabilities:

- **A1 — Network adversary**: Can observe, delay, reorder, or drop messages
  between peers. Cannot forge Dilithium3 signatures or break ECVRF proofs.
- **A2 — Compromised operator**: Controls one or more validator nodes, their
  signing keys, and can deviate from the protocol. Cannot control the majority
  of the quorum (M-of-N threshold protects against A2).
- **A3 — Compromised submitter**: Holds a valid 3CP signing identity but is not
  the intended submitter. Can submit entries using that identity — this is a
  custody problem, not a protocol failure (§11.4).
- **A4 — External adversary**: No access to any key material. Can attempt
  denial of service, timing attacks, or replay attacks.

### 12.2 Guarantees

| Guarantee | Condition |
|-----------|-----------|
| **Non-repudiation** | Dilithium3 signatures are unforgeable under NIST Level 3 assumptions |
| **Integrity** | SMT proofs are collision-resistant under BLAKE3-256 |
| **Proposer fairness** | ECVRF leader election is grinding-resistant (RFC 9381) |
| **Finality** | No forks, no rollbacks — one cycle = one block = final |
| **Third-party verifiability** | Any party can verify a proof against the public chain without server access |

### 12.3 Limitations

| Limitation | Mitigation |
|------------|------------|
| **No access control** | 3CP records who did what; it does not decide who can do what. Access control is the responsibility of the consuming application |
| **No encryption of entries** | Entry hashes are public. If the preimage must remain confidential, hash it before submission |
| **No built-in key rotation** | Identity rotation requires out-of-band consensus among validators |
| **Custody dependency** | Non-repudiation is void if signing keys are co-located with the credentials they should constrain (§11.4) |
| **Laplacian supervision** | λ₁ below `MinLambda1` may indicate network fragmentation; implementations SHOULD fail closed |

### 12.4 Cryptographic Agility

Future versions of 3CP may introduce new cryptographic suites. Implementations
MUST negotiate the suite at connection establishment. The v1 suite is fixed:
Dilithium3, Kyber1024, Ristretto255 ECVRF, BLAKE3-256, SHA-256.

## References

- [RFC 2119](https://tools.ietf.org/html/rfc2119) — Key words for use in RFCs
- [RFC 8610](https://tools.ietf.org/html/rfc8610) — CDDL: Concise Data Definition Language
- [RFC 9381](https://tools.ietf.org/html/rfc9381) — Verifiable Random Functions (VRFs)
- [FIPS 205](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.205.pdf) — ML-DSA (Dilithium)
- [FIPS 203](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.203.pdf) — ML-KEM (Kyber)
- [NIST SP 800-185](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-185.pdf) — SHA-3 Derived Functions (cSHAKE, etc.)
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) — BLAKE3 hash function specification
- [CBOR](https://www.rfc-editor.org/info/rfc8949) — Concise Binary Object Representation (RFC 8949)
