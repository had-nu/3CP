# Migration Guide: 3CP v1.0 → v2.0

## Overview

3CP v2.0 is a **hard fork**. The block format, consensus rules, and state
representation are incompatible with v1.0. There is no in-place upgrade path.

## Pre-Migration (v1.0 Network)

1. Halt all v1.0 validators at the same final block `B_final`.
2. Export the following state:
   - `ValidatorSet` (all ValidatorIDs, Dilithium3PKs, VRFPKs)
   - Active `MandateEntry` objects
   - Final `SMT` root hash
   - `H(B_final)` (for continuity proof)

## Genesis Construction (v2.0)

1. Create a new genesis block with:
   - `Index: 0`
   - `PrevHash: 0x00...00`
   - `Anchored: [GenesisMandate]` (converted from v1.0 mandates)
   - `Validators: [Dilithium3PKs from exported ValidatorSet]`
   - `ProtocolVersion: 2`
   - `LegacyAnchor: H(B_final)` (proves continuity with v1.0 chain)

2. Compute `NetworkID = BLAKE3-256(GenesisBlock)`.

3. Distribute the genesis block to all v2.0 validators.

## Post-Migration

- v1.0 blocks remain readable via archive nodes for historical audit.
- v2.0 validators do not process v1.0 blocks.
- The `LegacyAnchor` field allows auditors to cryptographically link the
  v2.0 genesis to the v1.0 final block.