# 3CP Protocol Overview

> **Document Level:** C — Protocol
>
> **Classification:** Normative (distilled from [`spec/3CP.md`](../../spec/3CP.md))
>
> **Status:** Draft
>
> **Protocol Version:** 1
>
> **Version:** 1.0
>
> **Source**
> - Specification: [`spec/3CP.md`](../../spec/3CP.md) §1–§2, §10
> - Implementation: `gleipnir-ipc/`
> - Tests: `gleipnir-ipc/cmd/conformance-test/` (33 tests)

---

RFC 2119 key words ("MUST", "SHOULD", "MAY", etc.) are used as defined in
[RFC 2119](https://tools.ietf.org/html/rfc2119). This section is the normative
protocol boundary; where the reference implementation diverges, the divergence
is flagged.

## Purpose

Provide a normative but readable map of 3CP: what it is, its layers, and how the
other Section 02 documents fit together. It is the protocol-level counterpart of
the conceptual [vision](../01-introduction/vision.md).

## Scope

The 3CP wire protocol, consensus rules, and verification algorithms at
**PROTOCOL_VERSION = 1**. Transport realisation (gRPC) and identity (UID0) are
covered in [networking](networking.md), [protobuf](protobuf.md), and referenced
where relevant.

## What 3CP Is

3CP is an **application-layer protocol** for producing, preserving, and
verifying cryptographic chain-of-custody evidence across distributed systems
(spec §1). Its distinguishing property:

> Verification of chain-of-custody evidence does not depend on the node that
> produced the evidence. Any party in possession of a chain's blocks can
> independently verify every SMT proof (§5.4), signature (§3.3), and block hash
> (§3.2) without contacting the originating network.

3CP requires no tokens, mining, or smart contracts, uses post-quantum
cryptography throughout, and is transport-agnostic (gRPC is canonical).

## Protocol Stack

| Layer | Definition | Document |
|-------|------------|----------|
| Wire format | Canonical CBOR (integer-keyed, deterministic) | [protobuf](protobuf.md), spec §3.6, §4 |
| Cryptographic primitives | BLAKE3-256, SHA-256, Dilithium3, Kyber1024, ECVRF | [cryptography](cryptography.md) |
| State commitment | Sparse Merkle Tree (BLAKE3, depth 256) | [sparse-merkle-tree](sparse-merkle-tree.md) |
| Leader election | ECVRF (RFC 9381, Ristretto255) | [vrf](vrf.md) |
| Consensus | VRF selection + M-of-N Dilithium3 quorum | [consensus](consensus.md) |
| Transport (canonical) | gRPC over Protocol Buffers | [networking](networking.md) |
| Identity | UID0 soulbound tokens | spec §8 |
| Sub-chains | Per-service SMT + cross-chain anchors | [anchoring](anchoring.md), spec §9 |
| Mandates | Signed, versioned anchoring obligations | [anchoring](anchoring.md), spec §13 |

## Normative Requirements Summary

A conformant implementation of 3CP MUST (spec §10):

- Implement the consensus cycle (§6) — see [consensus](consensus.md)
- Produce and verify VRF proofs (§3.5) — see [vrf](vrf.md)
- Produce and verify SMT proofs (§5) — see [sparse-merkle-tree](sparse-merkle-tree.md)
- Encode blocks and identity as canonical CBOR (§4, §8)
- Implement the gRPC service (§7) — see [networking](networking.md)
- Enforce entry validation (§7.2), including mandate-aware validation (§7.2.2)
- Implement `SubmitMandate`, `GetMandate`, `GetActiveMandates` (§7.5)
- Handle single-node and multi-node modes (§6.2, §6.3)
- Persist `MandateEntry` values alongside anchor proofs (§7.5.1)
- Resolve mandate inheritance for sub-chains (§9.3)

## Conformance Status

- Validated by a **33-test conformance suite** exercising every normative
  requirement at the gRPC boundary (spec §10; suite in
  `gleipnir-ipc/cmd/conformance-test/`).
- **No independent second implementation exists.** Any second implementation
  does so at its own risk of discovering untested edge cases (spec §10).

## Implementation Divergence (Gleipnir)

> The reference implementation does **not** satisfy every MUST above as
> reviewed:
>
> | MUST (spec §10) | Gleipnir status |
> |-----------------|-----------------|
> | Consensus cycle §6 | Implemented (`pkg/consensus/engine.go` `RunCycle`) |
> | VRF §3.5 | Implemented (`pkg/identity/vrf.go`); hash-to-curve diverges from RFC 9381 (see [vrf](vrf.md)) |
> | SMT §5 | Implemented (`pkg/smt/smt.go`) |
> | Canonical CBOR §4/§8 | Identity CBOR implemented (`pkg/identity/cbor.go`); block serialization is JSON in storage |
> | gRPC service §7 | Implemented (`pkg/server/`); `StreamBlocks` returns `Unimplemented` |
> | Mandate endpoints §7.5 | **Not implemented** |
> | Mandate inheritance §9.3 | **Not implemented** |
> | Entry validation §7.2 | Implemented (`pkg/validation`); codes differ from spec registry |
>
> These are load-bearing gaps. Treat the specification as the normative target
> and Gleipnir as a partial realisation. See
> [Section 03](../03-gleipnir/architecture.md).

## References

- [`spec/3CP.md`](../../spec/3CP.md) — full normative specification
- [consensus](consensus.md), [cryptography](cryptography.md),
  [sparse-merkle-tree](sparse-merkle-tree.md), [vrf](vrf.md),
  [networking](networking.md), [protobuf](protobuf.md),
  [anchoring](anchoring.md), [protocol-state-machine](protocol-state-machine.md)
