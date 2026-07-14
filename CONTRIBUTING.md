# Contributing to 3CP

**Status**: This repository is private and pre-publication. Contributions are
limited to invited collaborators. This document describes the process for
proposing changes to the protocol specification.

## How to Propose a Change

1. **Open an issue** describing the proposed change, its motivation, and its
   impact on the protocol.
2. **Discuss** with the maintainers. Small editorial changes may proceed
   directly. Protocol-level changes require consensus.
3. **Submit a pull request** with the proposed specification changes.

## Change Types

- **Editorial** — typos, formatting, clarifications that do not alter normative
  requirements. No consensus required beyond maintainer review.
- **Normative** — changes to MUST/SHOULD/MAY requirements, new features, wire
  format changes. Requires documented consensus and a rationale section.
- **Security-relevant** — changes that affect the threat model, cryptographic
  primitives, or adversarial assumptions. Requires a Security Considerations
  section and independent review.

## Specification Requirements

All normative changes to `spec/3CP.md` MUST:

1. Use RFC 2119 key words (MUST, SHOULD, MAY, etc.) correctly.
2. Include rationale for each normative requirement.
3. Update CDDL schemas in `spec/schemas/` if wire formats change.
4. Update test vectors in `spec/examples/` if data formats change.
5. Include a Security Considerations section if the change affects the threat
   model.

## Pull Request Template

```markdown
## Summary

[Brief description of the change]

## Motivation

[Why this change is necessary]

## Specification Changes

- `spec/3CP.md`: [summary of changes]
- `spec/schemas/`: [summary of changes, if any]
- `spec/examples/`: [summary of changes, if any]

## Rationale

[Design decisions, alternatives considered]

## Backward Compatibility

[Does this break existing implementations? If so, how?]

## Security Considerations

[Threat model implications, if any]
```

## License

By contributing to this repository, you agree that your contributions are made
under the same license as the repository (see [`LICENSE`](LICENSE)). You
represent that you have the right to grant these rights.
