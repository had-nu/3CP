<p align="center">
  <h1>3CP</h1>
</p>

<p align="center">
  <em>Cryptographic Chain-of-Custody Protocol</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-draft-yellow" alt="Status: Draft">
  <img src="https://img.shields.io/badge/protocol--version-v1-blue" alt="Protocol Version: v1">
  <img src="https://img.shields.io/badge/conformance-33%2F33%20passing-brightgreen" alt="Conformance: 33/33">
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
| Wire format | Canonical CBOR (integer-keyed, deterministic) — see [`spec/3CP.md`](spec/3CP.md) §4 |
| Consensus | ECVRF leader election (RFC 9381) + M-of-N Dilithium3 quorum — §6 |
| State | Sparse Merkle Tree (Blake3, depth 256) — §5 |
| Transport (canonical) | gRPC over Protocol Buffers — §7 |
| Sub-chains | Per-service SMT + cross-chain anchors — §9 |
| Identity | UID0 soulbound tokens with contract-derived binding — §8 |
| Mandates | Signed, versioned, anchored declarations of anchoring obligations — §13 |

## Repository Structure

```
3CP/
├── README.md            ← this file
├── CONTRIBUTING.md      ← how to propose changes
├── LICENSE              ← All Rights Reserved (pre-publication)
├── spec/
│   ├── 3CP.md           ← normative protocol specification (RFC 2119)
│   ├── glossary.md       ← terminology
│   ├── schemas/          ← CDDL wire format definitions
│   └── examples/         ← test vectors
```

## Status

- **Draft** — the specification is complete and validated by a 33-test
  conformance suite against the reference implementation.
- **Pre-publication** — this repository is private. The specification will be
  opened alongside the accompanying academic paper.
- **Protocol version**: v1 — wire-format stable. Future versions will be
  backward-compatible or explicitly versioned.

## Reference Implementation

**Gleipnir** — a Go reference implementation of the 3CP protocol. Available at
[github.com/had-nu/gleipnir](https://github.com/had-nu/gleipnir) under AGPL-3.0.
The Gleipnir conformance test suite (33 tests) validates the protocol at the
gRPC boundary.

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
  <strong>3CP</strong> — Cryptographic Chain-of-Custody Protocol v1<br>
  All Rights Reserved © 2026
</p>
