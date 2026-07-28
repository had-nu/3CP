# Light Client Verification in 3CP v2.0

## Principle

A light client does not execute consensus, maintain full state, or trust any
single node. It verifies blocks using only:

1. The `ValidatorSet` for the cycle in question.
2. The block's `PrepareSigs`, `CommitSig`, and `BlockHash`.
3. SMT proofs for individual entries.

## Verification Steps

Given a block `B` and a trusted `ValidatorSet_V` (obtained from an Anchor
Publisher or synced from a full node):

1. Verify `len(B.PrepareSigs) >= ceil(2*N/3)` where `N = len(ValidatorSet_V)`.
2. Verify each PREPARE signature against the corresponding validator's
   `Dilithium3PK` in `ValidatorSet_V`.
3. Verify `B.CommitSig` against the proposer's `Dilithium3PK`.
4. Verify `B.BlockHash` using the algorithm in §5.4 of SPEC-3CP-V2.md.
5. Verify the `PrevHash` chain back to a known anchor.

## SMT Proof Verification

For an individual entry `e` claimed to be in block `B`:

1. Obtain `SMTProof` for `e.Hash` at block `B.Index`.
2. Run `VerifySMTProof(B.StateRoot, e.Hash, e.Hash, SMTProof)`.
3. If true, `e` is provably anchored in `B`.

No consensus execution, no full chain sync, no trust in the operator.