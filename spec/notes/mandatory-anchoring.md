# Mandatory Event Anchoring — Paper Notes

## Context

3CP v1 established the cryptographic substrate for chain-of-custody evidence:
Dilithium3 signatures, SMT proofs, VRF consensus, sub-chains, third-party
verifiability. The protocol defined *how* to anchor but not *what must* be
anchored. An operator could silently omit events without protocol-level
consequence. The forensic indistinction problem — the inability to distinguish
a conscious decision from a non-event after the fact — merely shifted one level:
from "was this decision recorded?" to "was this event class subject to
recording?"

## Problem Statement

Existing compliance frameworks (CRA Art. 14, NIS2 Art. 21, DORA Chapter III)
require organisations to produce evidence of specific decisions — releases,
exceptions, incident responses — but provide no protocol-level mechanism for
making the *obligation* to produce that evidence independently verifiable.
Without such a mechanism, an auditor must rely on the organisation's own
declaration of what it chose to record. Accountability depends on the good
faith of the audited entity — precisely the assumption 3CP was designed to
eliminate.

## Solution: Protocol-Native Mandate Type

The Mandate extends 3CP with a signed, versioned, anchored declaration of
anchoring requirements. Key design properties:

1. **Mandates are entries.** A Mandate is submitted as a `ProvenanceEntry`
   (hash of its canonical CBOR) and inherits all 3CP guarantees: non-repudiation
   via Dilithium3, integrity via SMT, third-party verifiability.

2. **Mandates form a genealogical chain.** Each version links to the previous
   via `PrevVersion`. The full policy history is auditable from the chain.

3. **Compliance is an external function.** The protocol detects missing fields
   at submission time (when a submitter claims mandate compliance via
   `MandateRef`), but detecting *missing events* is a verification-time
   function performed by an auditor or compliance tool. This is intentional:
   the protocol cannot know about events that never reached it. What it
   provides is the cryptographic basis for comparing the chain against the
   mandate — a comparison any third party can perform independently.

## Key Design Decisions

### Decision 1: Mandate as ProvenanceEntry, not block header field

Mandates could have been a first-class block header field (like `QuorumConfig`
or `Validators`). This would have made mandate resolution implicit in every
block. Rejected because:
- It couples policy evolution to consensus parameter changes
- Mandates are authored by organisational authorities, not validators
- Versioned mandates with supersession require storage the block header
  was not designed for

Instead, mandates are submitted via the same `SubmitHash` path, labelled
`"3cp:mandate:v1"`, and resolved by querying the SMT. No consensus change.

### Decision 2: Dual enforcement — submission time and verification time

Submission-time enforcement validates structural compliance when a submitter
explicitly references a mandate. This prevents "I was complying with mandate X"
claims that fail basic field requirements.

Verification-time enforcement is the cryptographic novelty: a verifier holds
the chain and the mandate set, and can independently determine whether expected
entries exist. The protocol's guarantee is that this comparison is
deterministic, reproducible, and trustless — not that every omission is
impossible.

This is the protocol-level expression of the core thesis: accountability cannot
depend on the good faith of the audited entity.

### Decision 3: Sub-chain inheritance

Organisations with multiple services need hierarchical policy application.
A parent chain's mandates apply to sub-chains by default, with sub-chain-specific
overrides. This mirrors the organisational structure: hub-level policy sets the
baseline, service teams tighten for their domain.

## Implications

### For trust model

The trust model shifts from "trust the operator to record everything" to "trust
the operator to record the mandate, then verify the execution." The mandate
chain itself is subject to 3CP guarantees, so the operator cannot retroactively
alter what was required without detection. The auditor verifies both the
obligation and the execution from the same append-only log.

### For regulatory compliance

A mandate with `regulatory_scope: ["CRA"]` and `mandatory: true` provides a
protocol-native answer to the question "what did CRA Article 14 require this
organisation to anchor?" The answer is not a policy document — it is a signed,
versioned, anchored record that the organisation published and cannot repudiate.

### For forensic indistinction

Without a mandate: missing entry = indistinguishable from "not required."
With a mandate: missing entry = detectable compliance gap.

This closes the gap identified in the release-decisions and
cost-of-the-decision-not-taken articles. The forensic indistinction problem
does not disappear — it retreats to the question of whether the mandate itself
was correctly scoped. That question is governed by the mandate's own version
chain and the external `PolicyHash` reference.

## Open Questions

1. **Mandate discovery**: How does an auditor discover what mandates existed
   during a window without relying on the organisation's compliance team?
   `GetActiveMandates(T)` provides the mechanism; the question is whether
   an auditor independently verifies that the mandate set is complete (i.e.,
   no authorised authority issued a mandate the auditor missed).

2. **Cross-authority mandates**: Can a mandate issued by Authority A apply to
   submissions from Identity B? The current design ties each submission's
   `MandateRef` to a specific mandate; cross-authority application would
   require either a delegation mechanism or an organisational registry.
   Deferred to future work.

3. **Programmatic policy composition**: As the number of mandates grows,
   rule overlap and contradiction become governance problems. The protocol
   provides no conflict detection — that is the consuming application's
   responsibility.

## Related Work

- **SLSA Attestations**: Supply-chain levels attestation framework.
  SLSA defines *what* was produced; 3CP Mandates define *who required it to
  be attested*.
- **Sigstore / Rekor**: Transparency log for software signatures.
  Rekor records that something was signed; 3CP records the policy that
  required the signing and anchors both in the same verifiable structure.
- **in-toto**: Attestation framework for software supply chain integrity.
  In-toto focuses on the pipeline's steps; 3CP focuses on the governance
  layer that decides which steps require attestation.

## Visual Summary

```
┌─────────────────────────────────────────────────────┐
│                  External Policy                     │
│  (CRA, NIS2, DORA — PDF / legal text)              │
└──────────────────────┬──────────────────────────────┘
                       │ PolicyHash (BLAKE3-256)
                       ▼
┌─────────────────────────────────────────────────────┐
│              Mandate (anchored in 3CP)               │
│  Signed, versioned, time-bounded declaration         │
│  "release_gate + CVSS >= 9.0 + critical asset        │
│   = MUST anchor with Approver + Signature"           │
└──────────────────────┬──────────────────────────────┘
                       │ MandateRef (ProvenanceEntry key 7)
                       ▼
┌─────────────────────────────────────────────────────┐
│              Anchored Events (3CP chain)              │
│  Each entry references the mandate it satisfies       │
│  Verifier: "For every mandate rule, for every         │
│  expected event in window [T0,T1], does an entry      │
│  exist?"                                              │
└──────────────────────┬──────────────────────────────┘
                       │ Independent verification
                       ▼
┌─────────────────────────────────────────────────────┐
│              Compliance Gap Report                    │
│  gap: "release_gate @ 2026-06-15T14:23:00Z           │
│   — missing (no entry found)"                        │
│  gap: "exception_grant @ 2026-06-15T16:00:00Z        │
│   — missing_fields (no Approver)"                    │
└─────────────────────────────────────────────────────┘
```
