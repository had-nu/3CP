# Key Rotation in 3CP v2.0

## Problem

Long-running evidence chains require key rotation. If a validator's Dilithium3
private key is compromised, all future blocks signed with that key are suspect.
Without protocol-native rotation, operators must coordinate out-of-band, which
breaks the chain's auditability.

## Solution: `3cp:key-rotation:v1`

A validator rotates keys by submitting a specially-labelled ProvenanceEntry.
The entry contains both the old and new public keys, plus signatures from
both keys proving possession.

## Overlap Period

The protocol defines an overlap window `[EffectiveCycle, ExpiryCycle]`:

- Before `EffectiveCycle`: only the old key is valid.
- During overlap: both keys are accepted for block verification.
- After `ExpiryCycle`: only the new key is accepted for new signatures.

This ensures no gap in verification capability during rotation.

## NetworkID Binding

The `NetworkID` (BLAKE3-256 of the genesis block) is used as salt in UID0
derivation. This binds all identities to a specific chain, preventing
cross-chain identity replay.