# Light Client Verification in 3CP v3.0

> **Protocol Version:** 3
> **Source:** [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §12.2

## Principle

A light client executes no consensus, keeps no full state, and trusts no single
node. It relies only on:

1. A **trusted genesis `BlockHash`** (the trust anchor).
2. The canonical `ValidatorSet` for the cycle.
3. `PrepareSigsBitmap`, `PrepareSigsPayload`, `CommitSig`, and `BlockHash`.
4. SMT proofs for individual entries.

## Verification steps

1. Check `B.ProtocolVersion` is supported (== 3 for a v3.0 client).
2. Verify `popcount(B.PrepareSigsBitmap) >= ceil(2*N/3)` and bits `>= N` are zero.
3. Verify `len(B.PrepareSigsPayload) == popcount(bitmap)`.
4. Verify each PREPARE signature (exactly **3,309 bytes**, ML-DSA-65, empty
   context) against the validator at its bit position, over `H(B)`.
5. Verify `CommitSig` (3,309 bytes) against the proposer, over `B_final`
   excluding key 15.
6. Recompute `B.BlockHash` per §5.5 — preimage begins with `3CP-BLOCKHASH-v3`.
7. Reject if key 8 is present and non-null (`ErrReservedFieldPresent`).
8. Verify the `PrevHash` chain back to the trusted genesis.

> **Correction from v2.0.** v2.0 did not check `ProtocolVersion` before
> interpreting fields, did not enforce the reserved-key-8 rule, and referenced a
> v2.0 `BlockHash` without a domain tag. v3.0 makes each explicit.

## Signature size enforcement

Per FIPS 204 §10.2 a verifier MUST reject any signature whose length differs
from the parameter-set length: exactly **3,309 bytes** for ML-DSA-65; public keys
exactly **1,952 bytes**.

## External trust anchor

Verification MUST NOT derive the validator set from the block being verified —
a block that names its own validators can be signed entirely by whoever wrote
it. The set must come from genesis, an Anchor Publisher, or a vouched full node.

## SMT proof verification

For an entry `e` in block `B`: obtain the 8,192-byte `SMTProof` for `e.Hash` and
run `VerifySMTProof(B.StateRoot, e.Hash, e.Hash, SMTProof)` using the
`3CP-SMT-LEAF-v3` / `3CP-SMT-NODE-v3` tagged preimages (spec §9.3).

## References

- [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §12.2, §5.4–5.6, §9.3
- [`spec/examples/test-vectors-light-client.md`](../examples/test-vectors-light-client.md)
