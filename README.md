<p align="center">
  <h1>3CP</h1>
</p>

<p align="center">
  <em>Cryptographic Chain-of-Custody Protocol</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-draft-yellow" alt="Status: Draft">
  <img src="https://img.shields.io/badge/protocol--version-v2-blue" alt="Protocol Version: v2">
  <img src="https://img.shields.io/badge/conformance-extended%20suite-brightgreen" alt="Conformance: Extended">
  <img src="https://img.shields.io/badge/pre--publication-private-red" alt="Pre-publication: Private">
</p>

---

An **application-layer protocol** for producing, preserving, and verifying
cryptographic chain-of-custody evidence across distributed systems. 3CP enables
any party — regulator, auditor, consumer, third-party verifier — to
independently verify that a given digital artifact existed at a given point in
time, that its provenance was attested by known identities, and that no
retroactive reconstruction of events is possible without detection.

## Core Thesis

> Accountability cannot depend on the good faith of the audited entity.

3CP extends this principle with **Mandatory Event Anchoring** (§13): signed,
versioned, protocol-native declarations that define which events MUST be
recorded. Compliance is independently verifiable by comparing the chain against
the active mandate set. Omission becomes detectable.

Existing compliance frameworks (ISO 27001, SOC 2, NIST CSF, DORA, CRA) assume
that organisations produce evidence honestly and that auditors can evaluate it.
When legal liability and economic interest conflict, there is an objective
incentive to alter, suppress, fragment, or reinterpret that evidence. 3CP
inverts this model: it makes the integrity of chain-of-custody evidence
verifiable by independent third parties, regardless of the operator's
cooperation.

## What 3CP Is Not

3CP is not a blockchain, not a token, not a smart-contract platform, and not a
general-purpose ledger. It is a protocol — a set of wire formats, consensus
rules, and verification algorithms — that any system can implement to produce
contestable evidence of decision provenance.

## Key Properties

| Property | What it means |
|---|---|
| **Contestability** | Evidence can be challenged, but the challenge occurs over the intact chain, not over a chain reconstructed after the incident |
| **Third-party verifiability** | Any party with the public chain can independently verify SMT proofs, signatures, and block hashes without contacting the producing network |
| **Non-repudiation** | Each entry is cryptographically bound to the submitting identity via Dilithium3 signatures; once quorum-validated, no party can deny the submission |
| **Epistemic preservation** | The system records who knew what, when, who approved, who signed, who altered — the epistemology of the incident, not just hashes |
| **Post-quantum security** | Dilithium3 signatures, Kyber1024 KEM, ECVRF (Ristretto255) — NIST-standardized post-quantum cryptography throughout |
| **Mandatory anchoring** | Protocol-native Mandate declarations (§13) make anchoring obligations independently verifiable; compliance gaps (missing entries, missing fields) are cryptographically detectable by any third party |

## Protocol Stack

| Layer | Specification |
|---|---|
| Wire format | Canonical CBOR (integer-keyed, deterministic) — see [`spec/SPEC-3CP-V2.md`](spec/SPEC-3CP-V2.md) §5 |
| Consensus | Two-phase BFT (PREPARE/COMMIT) with ECVRF leader election + `ceil(2N/3)` Dilithium3 quorum — §6 |
| State | Sparse Merkle Tree (BLAKE3, depth 256) — §9 |
| Transport (canonical) | gRPC over Protocol Buffers — §7 |
| Sub-chains | Per-service SMT + cross-chain anchors — §9 |
| Identity | UID0 soulbound tokens with contract-derived binding — §14 |
| Mandates | Signed, versioned, anchored declarations of anchoring obligations — §13 |
| Key Rotation | Protocol-native `3cp:key-rotation:v1` entries — §8 |
| Verifiability | Light Client protocol + Anchor Publishers — §12 |

## Specification

The `spec/` directory contains the normative protocol definition:

- `SPEC-3CP-V2.md` — Current normative specification (v2.0)
- `3CP.md` — Historical v1.0 specification (preserved for reference)
- `glossary.md` — Unified terminology
- `schemas/` — CDDL schemas for all protocol data structures
- `notes/` — Explanatory documents on protocol design decisions
- `examples/` — Test vectors and conformance fixtures

## Implementations

3CP is a protocol specification. Reference implementations are maintained in
separate repositories:

- **Gleipnir** (Go) — Reference implementation of the 3CP node. Available at
  [github.com/had-nu/gleipnir](https://github.com/had-nu/gleipnir) under AGPL-3.0.
- **CARCOSA** (Rust) — ZK audit framework consuming the 3CP anchoring layer. Available at
  [github.com/had-nu/carcosa](https://github.com/had-nu/carcosa).

These repositories are **not part of this specification repository**.

```
3CP (protocol) ← Gleipnir (implements) ← CARCOSA (consumes)
```

## Conformance

Implementations claiming 3CP v2.0 conformance must pass the extended test suite
defined in `spec/examples/` and `spec/notes/`, including:

- 12+ adversarial BFT test cases (TC-BFT-01 through TC-MEM-01 per §15)
- Key rotation vectors (`test-vectors-key-rotation.md`)
- Light client verification vectors (`test-vectors-light-client.md`)

## Repository Structure

```
3CP/
├── README.md            ← this file
├── README.pt-BR.md      ← Portuguese version
├── CONTRIBUTING.md      ← how to propose changes
├── PLAYBOOK.md          ← intern onboarding playbook
├── LICENSE              ← All Rights Reserved (pre-publication)
├── spec/
│   ├── SPEC-3CP-V2.md   ← normative protocol specification v2.0 (RFC 2119)
│   ├── 3CP.md           ← historical v1.0 specification
│   ├── glossary.md      ← terminology
│   ├── schemas/         ← CDDL wire format definitions
│   ├── notes/           ← explanatory notes
│   └── examples/        ← test vectors
```

## Status

- **Draft** — the specification is complete and validated by an extended
  conformance suite against the reference implementation.
- **Pre-publication** — this repository is private. The specification will be
  opened alongside the accompanying academic paper.
- **Protocol version**: v2.0 — wire-format stable. Future versions will be
  backward-compatible or explicitly versioned.

## Citation (pre-publication)

```bibtex
@techreport{3cp-protocol,
  title        = {{3CP}: Cryptographic Chain-of-Custody Protocol},
  author       = {André Ataíde},
  year         = {2026},
  note         = {Pre-publication draft}
}
```

---

<p align="center">
  <strong>3CP</strong> — Cryptographic Chain-of-Custody Protocol v2.0<br>
  All Rights Reserved © 2026
</p>