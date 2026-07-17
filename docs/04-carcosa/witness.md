# Witness

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `carcosa/air/src/trace.rs`, `carcosa/cli/src/commands/prove.rs`
> - Design: `carcosa/docs/ARCHITECTURE.md` §3.2

---

## Purpose

Document how the witness (approver signatures + authorization) is turned into
the AIR trace.

## Scope

The mapping from real inputs to trace columns. The constraint logic is in
[air](air.md); the trace builder is [trace-generation](trace-generation.md).

## Witness Inputs

| Input | Source | Used in trace |
|-------|--------|---------------|
| Approver public keys | `--approver-keys` | `is_authorized` (vs mandate list) |
| Signatures | `--signatures` | `has_valid_sig` (verify over `approval_data`) |
| Approval data | `--approval-data` | signed message |
| Policy hash / mandate | `--policy-hash` | defines authorized set + threshold |

## As-Built

The trace builder (`trace.rs`) currently **hardcodes** the witness rather than
deriving it:

- `is_authorized` is set to `1` for every row (`trace.rs:24`) — no mandate-list
  check.
- `has_valid_sig` is `1` for all rows except the last, which is `0`
  (`trace.rs:19`) — a placeholder, not a real signature verification.

No signature verification or mandate authorization is performed in the
reference code. A real prover must:

1. Verify each signature over `approval_data` with its public key.
2. Check each key against the active mandate's authorized set.
3. Populate `has_valid_sig` / `is_authorized` from those checks.

## Known Gaps

- Witness is faked in the trace builder (D11).
- No linkage from `commitments` public input to actual keys ([air](air.md)).

## References

- [air](air.md), [trace-generation](trace-generation.md), [proving](proving.md)
