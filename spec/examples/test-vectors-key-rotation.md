# Test Vectors: Key Rotation v2.0

**Protocol version**: v2.0
**EventClass**: `3cp:key-rotation:v1`

---

## Vector 1: Valid Rotation

### Validator Identity

- ValidatorID: `a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6`
- Current Dilithium3PK: `[1952 bytes, hex: 01a2b3...]`
- Current VRFPK: `[32 bytes, hex: 4d5e6f...]`

### Key Rotation Entry (canonical CBOR)

```cbor
{
  0: h'8f9e8d7c6b5a4938271605f4e3d2c1b0...',  ; Hash
  1: h'a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6',       ; Submitter
  2: 1784030400000000000,                       ; Timestamp
  3: "3cp:key-rotation:v1",
  20: h'02b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7...',  ; NewPublicKey
  21: h'5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b',       ; NewVRFPublicKey
  22: 100,                                        ; EffectiveCycle
  23: 110,                                        ; ExpiryCycle
  24: h'[2700-byte SignatureOld]',                ; Signs Hash with old key
  25: h'[2700-byte SignatureNew]'                 ; Signs Hash with new key
}
```

### Verification Steps

1. `SignatureOld` verifies against Current Dilithium3PK.
2. `SignatureNew` verifies against `NewPublicKey`.
3. `EffectiveCycle (100) >= currentCycle + 10`.
4. `ExpiryCycle (110) >= EffectiveCycle + 10`.

### Expected Result: VALID

---

## Vector 2: Premature EffectiveCycle

Same as Vector 1, but `EffectiveCycle = 5` when `currentCycle = 10`.

### Expected Result: REJECTED (`ErrRotationTooEarly`)