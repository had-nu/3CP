# 3CP Glossary

## A

**Anchor** — The act of recording a hash into the 3CP chain. An anchored entry
becomes part of an immutable block and can be verified by any third party.

**AnchorProof** — A structure containing the SMT proof, block index, and block
timestamp for a given entry. Produced by `WaitForAnchor` or `VerifyHash`.

**Anchoring policy** — A set of rules, external to the protocol, that declares
which events an organisation requires to be recorded in the 3CP chain. 3CP does
not define anchoring policy; it provides the `Mandate` mechanism
(see [Mandate]) for anchoring the policy itself as a signed, versioned,
verifiable record.

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

**Compliance gap** — An event that should exist in the chain per an active
mandate but does not, or exists without the required fields (see [Mandate]).
Detected by comparing the mandate's rules against the chain at verification time.

**Chain-of-custody** — The chronological documentation of the sequence of
custody, control, transfer, analysis, and disposition of digital evidence. 3CP
produces cryptographic chain-of-custody evidence.

**CommitSig** — The leader's Dilithium3 signature over the final block hash,
including the PREPARE quorum evidence. Introduced in protocol v2.0.

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

**GenesisValidatorSet** — The initial set of validators declared in the genesis
block, containing each validator's ValidatorID, Dilithium3PK, and VRFPK.
Immutable after genesis.

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

**Light Client** — A protocol participant that verifies blocks and SMT proofs
without executing consensus or maintaining full network state.

## M

**Mandate** — A signed, versioned, anchored declaration defining which event
classes require anchoring under which conditions. A Mandate is itself a
`ProvenanceEntry` with a special label and a `MandateEntry` CBOR payload.
Mandates are the protocol-native mechanism for making anchoring obligations
verifiable by independent third parties. See [§13](3CP.md#13-mandates).

**Mandatory event** — An event whose absence from the chain constitutes a
detectable compliance gap (as opposed to a voluntary event, whose absence
carries no protocol-level implication). Mandatory status is declared by an
active `Mandate` rule with `mandatory: true`.

**M-of-N quorum** — A configurable threshold of validators required to finalize
a block. For example, `3/5` means 3 out of 5 validators must co-sign.

**Mode Degraded** — Consensus mode activated when TotalValidators < 4. Finality
requires only 1-of-N signatures, but each block must be labelled as degraded.

**GraceCycles** — Number of consecutive cycles with normal quorum required to
exit Mode Degraded. Default: 10. Configurable via Mandate.

## N

**NetworkID** — The BLAKE3-256 hash of the genesis block. Used as cryptographic
salt for UID0 derivation and domain separation between distinct 3CP networks.

**Anchor Publisher** — An entity (validator or external service) that publishes
finalized blocks to publicly-readable storage (IPFS, S3, filesystem) so that
light clients and auditors can access the chain without relying on validator
node availability.

**Non-repudiation** — A cryptographic guarantee that a party cannot deny having
submitted a specific entry. In 3CP, achieved via Dilithium3 signatures on both
block-level and entry-level payloads.

## P

**PrepareSigsBitmap** — Bitfield indicating which validators contributed a
PREPARE signature for a given block. Bit `i` corresponds to validator at index
`i` in the block's `Validators` array. Introduced in protocol v2.0.

**Proposer** — The validator selected by ECVRF leader election to build the next
block. Selected as the peer with the lowest VRF Gamma output.

**ProvenanceEntry** — A single anchored record containing a hash, submitter
identity, timestamp, optional label, optional approver, optional reference,
optional per-entry signature, and optional mandate reference (see [Mandate]).

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

**ZKBridge** — A versioned interface contract between the 3CP anchoring layer
and zero-knowledge proof consumers. Defined as ZKBridge v1.0.0 in protocol v2.0.