# 3CP Test Vectors

**Protocol version**: v2.0
**Generated**: 2026-07-28
**Source**: Gleipnir conformance test suite fixtures

---

## 1. Entry Hashes

Derived from Masthead finding fixtures (see `integration.go`). These hashes are
submitted to 3CP via `SubmitHash`. The hash algorithm is SHA-256.

| # | Path | Method | Header | Present | SHA-256 |
|---|------|--------|--------|---------|---------|
| 1 | `/login` | `GET` | `Strict-Transport-Security` | `false` | `62e3391cf9506246869a9a2828517c2dff1cf60c5c3d41798e693905cd4db509` |
| 2 | `/login` | `GET` | `Content-Security-Policy` | `false` | `de9e96f6c7c5ba98e476dabadfc75c9aa46c9a09a112c13e2807d89d9c535fe8` |
| 3 | `/api/v1/health` | `GET` | `Strict-Transport-Security` | `true` | `80be74254e5a2658c2027a53f53d826c456ec33e9693723d0a27de620b515c40` |

### Hash Computation (Entry 1)

```
Input:
  path:        "/login"
  method:      "GET"
  header_name: "Strict-Transport-Security"
  present:     "missing" (false → string "missing")

Preimage:
  "/login" || "GET" || "Strict-Transport-Security" || "missing"

SHA-256:
  62e3391cf9506246869a9a2828517c2dff1cf60c5c3d41798e693905cd4db509
```

---

## 2. Block v2.0 (Genesis, Index 0)

```
Block {
  Index:               0
  PrevHash:            0000000000000000000000000000000000000000000000000000000000000000
  StateRoot:           000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
  Proposer:            4e4f44453030312d2d2d2d2d2d2d2d2d
  Triad:               [reserved]
  Anchored:            [1 entry]
    [0]:
      Hash:      62e3391cf9506246869a9a2828517c2dff1cf60c5c3d41798e693905cd4db509
      Submitter: 4e4f44453030312d2d2d2d2d2d2d2d2d
      Timestamp: 1784030400000000000
      Label:     "release:gate"
      Approver:  (absent)
      Reference: (absent)
      Signature: (absent)
  Lambda1:             0.28
  Timestamp:           1784030400000000000
  Reserved:            null                           ; Key 8 — RESERVED in v2.0
  Validators:          [1 Dilithium3 public key, 1952 bytes]
  Quorum:              {TotalValidators: 1, RequiredSigs: 1}
  BlockHash:           9034d8157f01e4c2f742a4b2584ad971d4b4463c95e3fd7832c33771cbe84bd1
  ProtocolVersion:     2                              ; Key 12
  PrepareSigsBitmap:   0x01                           ; Key 13 — bit 0 set (single validator)
  PrepareSigsPayload:  [1 Dilithium3 signature, 2700 bytes] ; Key 14
  CommitSig:           1 Dilithium3 signature, 2700 bytes       ; Key 15
  ExternalAnchors:     ["ipfs://QmGenesis...", "file:///var/3cp/blocks/0.cbor"] ; Key 16
  KeyRotationEpoch:    0                              ; Key 17
}
```

### BlockHash Verification (v2.0)

```
Preimage:
  LE64(0)                                              = 00 00 00 00 00 00 00 00
  || PrevHash (32 bytes, zeros for genesis)            = 00×32
  || StateRoot (32 bytes, 0x00..0x1f)                  = 00 01 02 … 1e 1f
  || Proposer (16 bytes, "NODE001---------")           = 4e 4f 44 45 30 30 31 2d 2d 2d 2d 2d 2d 2d 2d 2d
  || HashOfAnchoredEntries (BLAKE3-256 of key 5)       = <computed from anchored array>
  || LE64(1784030400000000000)                          = 00 50 4f 7d 57 18 00 00
  || QuorumConfigCanonical                              = CBOR(10: {0: 1, 1: 1})

SHA-256(preimage) = 9034d8157f01e4c2f742a4b2584ad971d4b4463c95e3fd7832c33771cbe84bd1
```

This is an exact, reproducible computation. Any conformant implementation
producing a block with identical field values MUST compute the same BlockHash.

---

## 3. SMT Proof Format

An SMT proof for depth 256 is 256 consecutive sibling hashes (8192 bytes).

### Proof Structure

```
SMT Proof (8192 bytes):
  bytes 0-31:    sibling at depth 0
  bytes 32-63:   sibling at depth 1
  ...
  bytes 8160-8191: sibling at depth 255
```

### Leaf Hash

```
leaf = BLAKE3("leaf" || key || value)
  key   = entry.Hash (32 bytes)
  value = entry.Hash (32 bytes)
```

For a proof of absence (key does not exist):
```
leaf = BLAKE3("leaf" || zero[32] || zero[32])  // zero hash for both key and value
```

### Verification Algorithm

```
function verifySMTProof(root, key, value, proof):
    current = BLAKE3("leaf" || key || value)
    for depth = 0 to 255:
        sibling = proof[depth * 32 : (depth + 1) * 32]
        bit = (key[depth / 8] >> (depth % 8)) & 1
        if bit == 0:
            current = BLAKE3(current || sibling)
        else:
            current = BLAKE3(sibling || current)
    return current == root
```

---

## 4. AnchorProof (Verification Output)

Returned by `WaitForAnchor` or `VerifyHash`:

```
AnchorProof {
  Found:      true
  BlockIndex: 0
  BlockTime:  1784030400000000000
  StateRoot:  000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
  SMTProof:   [8192 bytes]
  Submitter:  4e4f44453030312d2d2d2d2d2d2d2d2d
  Label:      "release:gate"
}
```

### Verification Procedure

```
1. Compute leaf = BLAKE3("leaf" || entry.Hash || entry.Hash)
2. Walk proof using §5.4 algorithm
3. Verify result == AnchorProof.StateRoot
4. Retrieve block at AnchorProof.BlockIndex
5. Verify block.StateRoot == AnchorProof.StateRoot
6. Verify block's PrepareSigsPayload contains valid ceil(2N/3) PREPARE signatures
   and CommitSig verifies against Proposer
```

---

## 5. Cross-Chain Proof

```
CrossChainProof {
  EntryHash:    62e3391cf9506246869a9a2828517c2dff1cf60c5c3d41798e693905cd4db509
  SubChainRoot: 000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
  SubChainProof: [8192 bytes]
  AnchorHash:   9034d8157f01e4c2f742a4b2584ad971d4b4463c95e3fd7832c33771cbe84bd1
  AnchorBlock:  0
  ParentRoot:   000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
  ParentProof:  [8192 bytes]
}
```

### Verification Procedure

```
1. Verify sub-chain SMT proof:
   - Walk SubChainProof against SubChainRoot
   - Confirm it proves EntryHash

2. Verify parent chain SMT proof:
   - Walk ParentProof against ParentRoot
   - Confirm it proves AnchorHash

3. Retrieve block at AnchorBlock
4. Verify block contains an entry with Hash == AnchorHash
5. Verify AnchorHash's label == "subchain:anchor"
```

---

## 6. MandateEntry (Example — CRA-Compliant Policy)

```
MandateEntry {
  MandateID:    6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7
  Authority:    4e4f44453030312d2d2d2d2d2d2d2d2d ("NODE001---------")
  Version:      1
  PrevVersion:  0000000000000000000000000000000000000000000000000000000000000000
  ValidFrom:    1784030400000000000
  ValidUntil:   0                                         (never expires)
  Supersedes:   0000000000000000000000000000000000000000000000000000000000000000
  Rules:        [2 rules]
    [0]:
      EventClass:         "release_gate"
      Description:        "CRA Article 14 — critical severity releases MUST be anchored"
      SeverityMin:        9.0
      SeverityMax:        10.0
      AssetCriticalityMin: 3
      RegulatoryScope:    ["CRA"]
      Mandatory:          true
      RequiredFields:     ["Approver", "Signature", "Reference"]
      MaxDeferralSec:     0
    [1]:
      EventClass:         "exception_grant"
      Description:        "All risk acceptances MUST be anchored with justification"
      SeverityMin:        0.0
      SeverityMax:        10.0
      AssetCriticalityMin: 1
      RegulatoryScope:    []
      Mandatory:          true
      RequiredFields:     ["Approver", "Signature"]
      MaxDeferralSec:     0
  PolicyHash:   b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b
  PolicyURI:    "https://example.com/policies/cra-release-gate-v1.pdf"
  Signature:    [2700-byte Dilithium3 signature]
}
```

### MandateID Computation

```
MandateID = BLAKE3-256(canonicalCBOR(MandateEntry without Signature field))
```

The MandateID is the content-addressed identifier. Any party recomputing it from
the same fields obtains the same hash, which is then anchored as a
`ProvenanceEntry.Hash`.

### Active Period

```
ValidFrom (1784030400000000000 UnixNano) corresponds to:
  Date: 2026-07-14T00:00:00Z

ValidUntil == 0 means the mandate does not expire.
```

### Submission as ProvenanceEntry

When submitted via `SubmitMandate`, the `ProvenanceEntry` would be:

```
ProvenanceEntry {
  Hash:      a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b
             (= BLAKE3-256 of canonical MandateEntry CBOR excluding Signature)
  Submitter: 4e4f44453030312d2d2d2d2d2d2d2d2d
  Timestamp: 1784030400000000000
  Label:     "3cp:mandate:v1"
  Approver:  4e4f44453030312d2d2d2d2d2d2d2d2d
  Reference: (absent)
  Signature: 4e4f... (same Dilithium3 signature as MandateEntry.Signature)
  MandateRef: (absent — mandates are self-referential)
}
```

---

## 7. Block v2.0 (Index 1, 4-node network)

```cbor
Block {
  0: 1,
  1: h'9034d8157f01e4c2f742a4b2584ad971d4b4463c95e3fd7832c33771cbe84bd1', ; PrevHash (genesis BlockHash)
  2: h'b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2', ; StateRoot
  3: h'val0------------', ; Proposer
  5: [ /* 2 entries */ ],
  6: 0.42,
  7: 1784030403000000000,
  8: null,                              ; Key 8 — RESERVED in ProtocolVersion == 2
  9: [h'[pk0]', h'[pk1]', h'[pk2]', h'[pk3]'], ; 4 validators
  10: {0: 4, 1: 3},                     ; QuorumConfig: 4 validators, ceil(2*4/3)=3
  11: h'blockhash1................................', ; BlockHash
  12: 2,                                 ; ProtocolVersion
  13: h'0xe0',                           ; PrepareSigsBitmap: val0,val1,val2 signed (bits 0,1,2)
  14: [h'[sig0]', h'[sig1]', h'[sig2]'], ; PrepareSigsPayload (3 sigs)
  15: h'[CommitSig from val0]',          ; CommitSig (leader = val0)
  16: ["ipfs://QmX4z...", "file:///var/3cp/blocks/1.cbor"],
  17: 0,                                 ; KeyRotationEpoch
}
```

### Field-by-Field Explanation

| Key | Field | Value | Notes |
|-----|-------|-------|-------|
| 0 | Index | 1 | Cycle 1 |
| 1 | PrevHash | genesis BlockHash | Links to block 0 |
| 2 | StateRoot | SMT root after inserting 2 entries | |
| 3 | Proposer | `val0` RootID | Leader per VRF election |
| 5 | Anchored | 2 entries | Application payload |
| 6 | Lambda1 | 0.42 | Fiedler eigenvalue |
| 7 | Timestamp | 1784030403000000000 | UnixNano |
| 8 | Reserved | null | MUST be null in v2.0 |
| 9 | Validators | 4 × Dilithium3PK | Complete validator set |
| 10 | QuorumConfig | {Total: 4, Required: 3} | ceil(2×4/3) = 3 |
| 11 | BlockHash | SHA-256 of preimage | Excludes keys 12-17 |
| 12 | ProtocolVersion | 2 | v2.0 |
| 13 | PrepareSigsBitmap | `0xe0` (binary `11100000`) | Bits 0,1,2 = 1 (val0,val1,val2) |
| 14 | PrepareSigsPayload | 3 × 2700-byte sigs | Only active signers, in index order |
| 15 | CommitSig | 2700 bytes | Leader (val0) signs H(B_final) |
| 16 | ExternalAnchors | 2 URIs | IPFS + local filesystem |
| 17 | KeyRotationEpoch | 0 | No rotation yet |

---

## 8. Key Rotation Entry (Valid Rotation)

```
key-rotation-entry {
  0: h'hash32...',                    ; Hash (BLAKE3-256 of payload)
  1: h'val1------------',              ; Submitter (ValidatorID)
  2: 1784030400000000000,              ; Timestamp
  3: "3cp:key-rotation:v1",            ; Label
  20: h'[1952-byte Dilithium3 PK]',    ; NewPublicKey
  21: h'[32-byte VRF PK]',             ; NewVRFPublicKey
  22: 15,                              ; EffectiveCycle
  23: 25,                              ; ExpiryCycle
  24: h'[2700-byte sig with OLD key]', ; SignatureOld
  25: h'[2700-byte sig with NEW key]', ; SignatureNew
}
```

### Validation Checklist

- [ ] `SignatureOld` verifies against `val1`'s active Dilithium3PK at current cycle
- [ ] `SignatureNew` verifies against `NewPublicKey` (field 20)
- [ ] `EffectiveCycle (15) >= currentCycle + KeyRotationLeadTime (default 10)`
- [ ] `ExpiryCycle (25) >= EffectiveCycle + MinKeyOverlap (default 10)`
- [ ] `EffectiveCycle > lastRotationCycle` for `val1` (no overlapping rotations)

---

## 9. Light Client Verification (Block 1)

Given block `B` (index 1) and trusted `ValidatorSet_V` (obtained from Anchor Publisher):

1. `RequiredSigs = ceil(2*4/3) = 3`. `PrepareSigsPayload` has 3 signatures. ✓
2. Verify `PrepareSigsPayload[0]` against `Validators[0]` (val0). ✓
3. Verify `PrepareSigsPayload[1]` against `Validators[1]` (val1). ✓
4. Verify `PrepareSigsPayload[2]` against `Validators[2]` (val2). ✓
5. Verify `CommitSig` against `Validators[0]` (val0, the proposer). ✓
6. Verify `BlockHash` algorithm per §4.4. ✓
7. Verify `PrevHash` chain to genesis. ✓

---

## Conformance Notes

- All SHA-256 hashes above are computed with Go's `crypto/sha256`.
- All BLAKE3 hashes use BLAKE3-256 (32-byte output).
- Timestamps are UnixNano (`time.UnixNano()`).
- The genesis block uses single-node mode (QuorumConfig 1/1) for simplicity.
- For multi-node, `PrepareSigsPayload` contains `ceil(2N/3)` × 2700-byte Dilithium3 signatures.
- Binary values are hex-encoded. The canonical representation is raw bytes.
- Key 8 is `null` in all v2.0 blocks; presence of non-null value MUST cause rejection.
- `PrepareSigsBitmap` length matches `len(Validators)`; unused high-order bits MUST be zero.
- `PrepareSigsPayload` length MUST equal popcount of `PrepareSigsBitmap`.