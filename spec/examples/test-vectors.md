# 3CP Test Vectors

**Protocol version**: v1
**Generated**: 2026-07-14
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

## 2. Block (Genesis, Index 0)

```
Block {
  Index:       0
  PrevHash:    0000000000000000000000000000000000000000000000000000000000000000
  StateRoot:   000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
  Proposer:    4e4f44453030312d2d2d2d2d2d2d2d2d
  Triad:       [reserved]
  Anchored:    [1 entry]
    [0]:
      Hash:      62e3391cf9506246869a9a2828517c2dff1cf60c5c3d41798e693905cd4db509
      Submitter: 4e4f44453030312d2d2d2d2d2d2d2d2d
      Timestamp: 1784030400000000000
      Label:     "release:gate"
      Approver:  (absent)
      Reference: (absent)
      Signature: (absent)
  Lambda1:     0.28
  Timestamp:   1784030400000000000
  Sigs:        [1 Dilithium3 signature, 2700 bytes]
  Validators:  [1 Dilithium3 public key, 1952 bytes]
  Quorum:      {TotalValidators: 1, RequiredSigs: 1}
  BlockHash:   9034d8157f01e4c2f742a4b2584ad971d4b4463c95e3fd7832c33771cbe84bd1
}
```

### BlockHash Verification

```
Preimage:
  LE64(0)                                              = 00 00 00 00 00 00 00 00
  || PrevHash (32 bytes, zeros for genesis)            = 00×32
  || StateRoot (32 bytes, 0x00..0x1f)                  = 00 01 02 … 1e 1f
  || Proposer (16 bytes, "NODE001---------")           = 4e 4f 44 45 30 30 31 2d 2d 2d 2d 2d 2d 2d 2d 2d
  || anchored[0].Hash (32 bytes, Entry 1 hash)         = 62 e3 39 1c … 09
  || LE64(1784030400000000000)                          = 00 50 4f 7d 57 18 00 00

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
6. Verify block's Sigs field contains valid M-of-N+ signatures
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

## Conformance Notes

- All SHA-256 hashes above are computed with Go's `crypto/sha256`.
- All BLAKE3 hashes use BLAKE3-256 (32-byte output).
- Timestamps are UnixNano (`time.UnixNano()`).
- The test vector block uses single-node mode (QuorumConfig 1/1).
- For multi-node, Sigs contains M × 2700-byte Dilithium3 signatures.
- Binary values are hex-encoded. The canonical representation is raw bytes.
