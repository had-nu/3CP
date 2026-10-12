# Key Rotation in 3CP v3.0

> **Protocol Version:** 3
> **Source:** [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §8, §11.1

## Problem

Long-running evidence chains require key rotation. If a validator's ML-DSA-65
private key is compromised, all future blocks signed with it are suspect.
Without protocol-native rotation, operators must coordinate out-of-band, which
breaks the chain's auditability.

## Solution: `3cp:key-rotation:v1`

A validator rotates keys by submitting a labelled `ProvenanceEntry`. The entry
carries the new public keys plus **two** signatures — one from the current key
and one from the new key — over a single domain-separated payload:

```
Hash              = BLAKE3-256(canonicalCBOR(fields 1,2,3,20,21,22,23))
KeyRotationPayload = "3CP-KEYROT-v3" || Hash
SignatureOld = ML-DSA65_Sign(sk_current, KeyRotationPayload, ctx="")
SignatureNew = ML-DSA65_Sign(sk_new,      KeyRotationPayload, ctx="")
```

Both signatures cover the identical payload, proving **possession** of the new
key and **authorisation** by the current key.

> **Correction from v2.0.** v2.0 defined `Hash` as "BLAKE3-256 of the payload"
> without stating which fields were excluded. v3.0 fixes the exact preimage
> (fields 0, 24, 25 excluded) and adds the `3CP-KEYROT-v3` domain tag.

## Overlap Period

- Before `EffectiveCycle`: only the old key is valid.
- During `[EffectiveCycle, ExpiryCycle]`: both keys are accepted.
- After `ExpiryCycle`: only the new key is accepted for new signatures.
- Blocks before `EffectiveCycle` remain verifiable with the old key
  indefinitely, so historical verification never depends on keeping a retired
  key online.

## Signature sizes

Both signatures are exactly **3,309 bytes** (ML-DSA-65, FIPS 204); public keys
exactly **1,952 bytes**. An entry whose lengths differ MUST be rejected.

## NetworkID Binding

`NetworkID` — `BLAKE3-256` of the canonical CBOR of the genesis block with key 11
omitted — is the HKDF salt in UID0 derivation and is bound into every
`BlockHash`. This binds identities and block identities to one chain,
preventing cross-chain identity and block replay.

## References

- [`spec/SPEC-3CP-V3.md`](../SPEC-3CP-V3.md) §8, §11.1
- [`spec/examples/test-vectors-key-rotation.md`](../examples/test-vectors-key-rotation.md)
- [FIPS 204](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf) — ML-DSA
