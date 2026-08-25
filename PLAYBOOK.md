# 3CP Onboarding Playbook

**Status**: Draft — internal orientation document  
**Audience**: New collaborators on the 3CP protocol specification  
**Last updated**: 2026-08-25

---

## 1. What Is 3CP?

3CP (Cryptographic Chain-of-Custody Protocol) is an **application-layer protocol** for producing, preserving, and verifying cryptographic chain-of-custody evidence across distributed systems.

**Core thesis**: *Accountability cannot depend on the good faith of the audited entity.*

3CP enables any party — regulator, auditor, consumer, third-party verifier — to independently verify that a digital artifact existed at a given point in time, that its provenance was attested by known identities, and that no retroactive reconstruction of events is possible without detection.

---

## 2. Protocol Stack Overview

| Layer | Specification | Status |
|-------|---------------|--------|
| Wire format | Canonical CBOR (integer-keyed, deterministic) | Normative — `spec/3CP-v2.md` §4 |
| Consensus | Two-phase BFT (PREPARE/COMMIT) with ECVRF leader election + `ceil(2N/3)` Dilithium3 quorum | Normative — `spec/3CP-v2.md` §5 |
| State | Sparse Merkle Tree (BLAKE3, depth 256) | Normative — `spec/3CP-v2.md` §8 |
| Transport (canonical) | gRPC over Protocol Buffers | Normative — `spec/3CP-v2.md` §6 |
| Sub-chains | Per-service SMT + cross-chain anchors | Normative — `spec/3CP-v2.md` §13 |
| Identity | UID0 soulbound tokens with contract-derived binding | Normative — `spec/3CP-v2.md` §13 |
| Mandates | Signed, versioned, anchored declarations of anchoring obligations | Normative — `spec/3CP-v2.md` §14 |
| Key Rotation | Protocol-native `3cp:key-rotation:v1` entries | Normative — `spec/3CP-v2.md` §7 |
| Verifiability | Light Client protocol + Anchor Publishers | Normative — `spec/3CP-v2.md` §11 |
| ZK Interface | ZKBridge v1.0.0 (stable contract for ZK consumers) | Normative — `spec/3CP-v2.md` §12 |

---

## 3. Repository Structure

```
3CP/
├── README.md            ← Public project overview
├── README.pt-BR.md      ← Portuguese version
├── CONTRIBUTING.md      ← How to propose changes
├── PLAYBOOK.md          ← This file (onboarding)
├── LICENSE              ← All Rights Reserved (pre-publication)
├── MOVE-LOG.md          ← Reorganization history (2026-08-21)
├── spec/
│   ├── 3CP-v2.md        ← **Current normative specification (v2.0)**
│   ├── 3CP.md           ← Historical v1.0 specification (preserved for reference)
│   ├── glossary.md      ← Unified terminology
│   ├── schemas/         ← CDDL wire format definitions
│   ├── notes/           ← Explanatory notes on design decisions
│   └── examples/        ← Test vectors and conformance fixtures
└── .gitignore           ← Excludes internal/, .old/
```

**Key files for implementers:**
- Start with `spec/3CP-v2.md` — the complete normative spec
- Reference `spec/schemas/` for CDDL definitions of all wire formats
- Use `spec/examples/test-vectors.md` for conformance testing

---

## 4. Key Concepts to Internalize

### 4.1 Contestability Fabric
3CP is not merely an audit log. Evidence can be challenged, but the challenge occurs over the **intact chain**, not over a chain reconstructed after the incident.

### 4.2 Third-Party Verifiability
Any party with the public chain can independently verify SMT proofs, signatures, and block hashes **without contacting the producing network**.

### 4.3 Mandatory Event Anchoring (§13)
Signed, versioned, protocol-native declarations (`Mandates`) define which events MUST be recorded. Compliance is independently verifiable by comparing the chain against the active mandate set. Omission becomes detectable.

### 4.4 Hard Fork Migration (v1 → v2)
The migration from v1.0 to v2.0 is a **hard fork** (§15). Legacy chains are anchored via `LegacyAnchor` in the v2.0 genesis block. No gradual transition — the wire format and consensus rules change atomically.

---

## 5. Reference Implementations (Suggested)

The following are **reference implementations** maintained in separate repositories. They are **suggestions for study and interoperability testing**, not normative requirements.

| Name | Language | Role | Repository |
|------|----------|------|------------|
| **Gleipnir** | Go | Reference implementation of the 3CP node | `github.com/had-nu/gleipnir` (AGPL-3.0) |
| **CARCOSA** | Rust | ZK audit framework consuming the 3CP anchoring layer | `github.com/had-nu/carcosa` |

**Relationship:**
```
3CP (protocol specification) ← Gleipnir (implements) ← CARCOSA (consumes)
```

**Notes:**
- These repositories are **not part of this specification repository**
- Protocol conformance is defined by passing the test vectors in `spec/examples/` and the adversarial test cases in `spec/3CP-v2.md` §16
- Any independent implementation achieving interoperability with Gleipnir on the test vectors is conformant

---

## 6. How to Propose Changes

See `CONTRIBUTING.md` for the formal process. Summary:

1. **Open an issue** describing the proposed change, motivation, and protocol impact
2. **Discuss** with maintainers. Editorial changes may proceed directly. Protocol-level changes require consensus.
3. **Submit a pull request** with:
   - Changes to `spec/3CP-v2.md` (normative text, RFC 2119 keywords)
   - Updated CDDL schemas in `spec/schemas/` if wire formats change
   - Updated test vectors in `spec/examples/` if data formats change
   - Security Considerations section if the change affects the threat model

---

## 7. Common Tasks

### 7.1 Adding a New Protocol Feature
1. Draft the normative text in `spec/3CP-v2.md` using RFC 2119 keywords
2. Add/update CDDL schemas in `spec/schemas/`
3. Add test vectors in `spec/examples/test-vectors.md`
4. Add conformance test case to `spec/3CP-v2.md` §16 table
5. Open PR with rationale and security considerations

### 7.2 Fixing an Editorial Issue
- Typos, formatting, clarifications: direct PR to `spec/3CP-v2.md`, `glossary.md`, or notes
- No consensus required beyond maintainer review

### 7.3 Updating Test Vectors
- Modify `spec/examples/test-vectors.md`
- Ensure vectors are reproducible (include computation steps)
- Binary values as hex; canonical form is raw bytes

### 7.4 Adding Explanatory Notes
- Create `spec/notes/<topic>.md` for design rationale, non-normative explanations
- Reference from `spec/3CP-v2.md` with "See `spec/notes/<topic>.md` for discussion"

---

## 8. Conformance Testing

To claim 3CP v2.0 conformance, an implementation **MUST** pass:

1. **All test vectors** in `spec/examples/test-vectors.md`
2. **12+ adversarial BFT test cases** (TC-BFT-01 through TC-MEM-01 per `spec/3CP-v2.md` §16)
3. **Key rotation vectors** (`spec/examples/test-vectors-key-rotation.md`)
4. **Light client verification vectors** (`spec/examples/test-vectors-light-client.md`)

The reference implementation (Gleipnir) includes an automated conformance suite. Run it with:
```bash
# In gleipnir repo
go test -v ./... -run Conformance
```

---

## 9. Security Considerations

- **Threat model**: See `spec/3CP-v2.md` §2.2 (A1–A5)
- **Post-quantum**: Dilithium3 (ML-DSA-65), Kyber1024 (ML-KEM-1024), ECVRF (Ristretto255)
- **VRF divergence**: 3CP uses a 2-point Elligator2 sum, not RFC 9381 `hash_to_ristretto255`. See `spec/3CP-v2.md` §3.3 for byte-exact procedure.
- **Key rotation**: Overlap period ensures backward verifiability; old keys never expire for historical blocks
- **Fragmentation detection**: `λ₁ < MinLambda1` triggers cycle abort and `fragmented` state

---

## 10. Glossary Quick Reference

See `spec/glossary.md` for complete definitions. Key terms:

| Term | Meaning |
|------|---------|
| `ProvenanceEntry` | Anchored evidence unit (hash + metadata + optional signatures) |
| `Block` | Consensus output containing ordered entries, signatures, and state root |
| `SMT` | Sparse Merkle Tree — authenticated state map |
| `ValidatorSet` | Current set of validators with Dilithium3PK + VRFPK |
| `PrepareSigsBitmap` | Bitfield of validators who signed PREPARE |
| `PrepareSigsPayload` | Actual PREPARE signatures (only active signers) |
| `CommitSig` | Leader's signature on final block hash |
| `Mandate` | Signed declaration of anchoring obligations |
| `Anchor Publisher` | Entity publishing final blocks to public storage |
| `Light Client` | Verifier that checks blocks without full state |
| `NetworkID` | BLAKE3-256(GenesisBlock) — immutable chain identifier |
| `LegacyAnchor` | Hash of v1.0 final block, embedded in v2.0 genesis |

---

## 11. Current Status

- **Protocol version**: v2.0 — wire-format stable
- **Specification**: Draft (complete, validated by extended conformance suite against reference implementation)
- **Publication**: Pre-publication (repository visible for LLM research; may return to private)
- **License**: All Rights Reserved © 2026

---

## 12. Next Steps for New Collaborators

1. Read `spec/3CP-v2.md` end-to-end
2. Review `spec/schemas/block.cddl` and `spec/schemas/mandate.cddl`
3. Work through `spec/examples/test-vectors.md` — reproduce BlockHash computations
4. Run Gleipnir conformance suite locally
5. Pick a `spec/notes/` topic or open issue to contribute

---

## 13. Contact

**Maintainer**: André Ataíde  
**Questions/Issues**: Use GitHub Issues on this repository

---

*This playbook is an orientation guide. The normative specification is `spec/3CP-v2.md`. In case of conflict, the spec prevails.*