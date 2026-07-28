# Test Vectors: Light Client Verification v2.0

**Protocol version**: v2.0

---

## Vector 1: Verify Block Quorum

### Block (Index 5)

```cbor
Block {
  0: 5,
  1: h'9034d8157f01e4c2f742a4b2584ad971...',  ; PrevHash
  2: h'a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6...',  ; StateRoot
  3: h'val1--------------------------------',  ; Proposer
  7: 1784030400000000000,
  8: [h'[2700-byte sig from val0]', h'[2700-byte sig from val1]', h'[2700-byte sig from val2]'],
  9: [h'[1952-byte pk val0]', h'[1952-byte pk val1]', h'[1952-byte pk val2]', h'[1952-byte pk val3]'],
  10: {0: 4, 1: 3},  ; TotalValidators=4, RequiredSigs=3
  11: h'blockhash5................................',
  12: 2,                    ; ProtocolVersion
  13: h'0xe0',              ; Bitmap: val0,val1,val2 signed (bits 0,1,2)
  14: [h'[sig0]', h'[sig1]', h'[sig2]'],
  15: h'[2700-byte CommitSig from val1]',
}
```

### ValidatorSet (Cycle 5)

```
[val0, val1, val2, val3] with Dilithium3PKs matching block key 9.
```

### Light Client Verification

1. `RequiredSigs = ceil(2*4/3) = 3`. `PrepareSigs` has 3 signatures. ✓
2. Verify `PrepareSigs[0]` against `Validators[0]` (val0). ✓
3. Verify `PrepareSigs[1]` against `Validators[1]` (val1). ✓
4. Verify `PrepareSigs[2]` against `Validators[2]` (val2). ✓
5. Verify `CommitSig` against `Validators[1]` (val1, the proposer). ✓
6. Verify `BlockHash` algorithm. ✓

### Expected Result: VALID

---

## Vector 2: Insufficient Quorum

Same block, but `PrepareSigsBitmap = 0x30` (only val2 and val3 signed).

### Expected Result: INVALID (`ErrQuorumNotMet`)