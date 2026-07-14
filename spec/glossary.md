# 3CP Glossary

## A

**Anchor** — The act of recording a hash into the 3CP chain. An anchored entry
becomes part of an immutable block and can be verified by any third party.

**AnchorProof** — A structure containing the SMT proof, block index, and block
timestamp for a given entry. Produced by `WaitForAnchor` or `VerifyHash`.

**Application-layer protocol** — A protocol that operates at the application
layer of the OSI model, equivalent to HTTP, DNS, or SMTP. 3CP is an
application-layer protocol, not a platform or blockchain.

## B

**Block** — An atomic unit of the 3CP chain. Each block contains zero or more
`ProvenanceEntry` values, a reference to the previous block, a state root,
signatures, and metadata. One block is produced per consensus cycle.

**BlockHash** — The SHA-256 identifier of a block, computed over a canonical
preimage of the block's fields. Used for chain linking and signature targets.

## C

**Canonical CBOR** — Deterministic CBOR encoding as specified by CDDL. Integer
keys in ascending order, smallest integer representation, definite-length
strings.

**Chain-of-custody** — The chronological documentation of the sequence of
custody, control, transfer, analysis, and disposition of digital evidence. 3CP
produces cryptographic chain-of-custody evidence.

**Contestability** — The property that evidence can be challenged, but the
challenge must occur over the intact chain, not over a chain reconstructed after
the incident. This is the core architectural property of 3CP.

**Consensus cycle** — A single tick-driven round that produces one anchored
block. Includes VRF proposer selection, block proposal, SMT verification, M-of-N
co-signing, and state transition.

**Cross-chain proof** — A dual-Merkle proof that demonstrates an entry exists in
a sub-chain AND that the sub-chain root was anchored in the parent chain.

## D

**Dilithium3** — ML-DSA-65, a NIST-standardized post-quantum digital signature
algorithm. Security level: NIST Level 3 (AES-256 equivalent). Signature size:
2700 bytes.

## E

**ECVRF** — Elliptic Curve Verifiable Random Function, as specified in RFC 9381.
Used by 3CP for leader election. Implemented over Ristretto255.

**Entry** — See ProvenanceEntry.

## F

**Finality** — In 3CP, finality is instant: one cycle produces one block, and
that block is final. No forks, no rollbacks, no reorganization.

## G

**gRPC** — The canonical transport for the 3CP wire protocol. Conformant
implementations MUST expose the 3CP gRPC service.

## I

**Instant finality** — See Finality.

## K

**Kyber1024** — ML-KEM-1024, a NIST-standardized post-quantum key encapsulation
mechanism. Security level: NIST Level 5.

## L

**Laplacian λ₁** — The Fiedler eigenvalue (second-smallest eigenvalue) of the
network's Laplacian matrix. Used by 3CP to supervise network diffusion health.
Values below `MinLambda1` indicate possible network fragmentation.

## M

**M-of-N quorum** — A configurable threshold of validators required to finalize
a block. For example, `3/5` means 3 out of 5 validators must co-sign.

## N

**Non-repudiation** — A cryptographic guarantee that a party cannot deny having
submitted a specific entry. In 3CP, achieved via Dilithium3 signatures on both
block-level and entry-level payloads.

## P

**Proposer** — The validator selected by ECVRF leader election to build the next
block. Selected as the peer with the lowest VRF Gamma output.

**ProvenanceEntry** — A single anchored record containing a hash, submitter
identity, timestamp, optional label, optional approver, optional reference, and
optional per-entry signature.

## Q

**QuorumConfig** — A structure specifying the total number of validators and the
minimum number of signatures required to finalize a block.

## R

**RootID** — A 16-byte unique identifier for a validator node. Derived from the
UID0 identity token.

## S

**SMT** — Sparse Merkle Tree. A compact, verifiable binary tree structure used
by 3CP to commit to the set of all anchored entries. Depth: 256. Hash function:
BLAKE3-256.

**SMT proof** — A sequence of sibling hashes (each 32 bytes) from leaf to root
that proves inclusion or exclusion of a key in the SMT.

**Sub-chain** — An isolated SMT instance for a specific service, periodically
checkpointed into the parent chain via cross-chain proofs.

## T

**Third-party verifiability** — The property that any party with access to the
public chain can independently verify proofs, signatures, and block hashes
without contacting the network that produced them.

## U

**UID0** — A soulbound identity token that binds a validator node to a
deterministic, verifiable identity. Serialized as canonical CBOR.

## V

**VRF** — Verifiable Random Function. See ECVRF.

## Z

**Zero hash** — A 32-byte array of all zeros. Rejected by entry validation as
invalid (`ZERO_HASH` error).
