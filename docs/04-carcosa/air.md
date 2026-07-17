# AIR

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `carcosa/air/src/approval_air.rs`, `carcosa/air/src/trace.rs`
> - Framework: Winterfell 0.13 (`winter-air`)

---

## Purpose

Document the Winterfell AIR that encodes the threshold-approval proof.

## Scope

`ApprovalAir` and `ApprovalTrace`. The prover/verifier that consume it are in
[proving](proving.md) / [verification](verification.md).

## Public Inputs

`PublicInputs` (`approval_air.rs:10`): `policy_hash`, `threshold: u64`,
`approval_data_hash`, `commitments: Vec<[u8;32]>`.

> **Divergence:** `ToElements` (`approval_air.rs:17`) serializes **only**
> `threshold` into the public-input elements. The `policy_hash`,
> `approval_data_hash`, and `commitments` digests are **not** bound to the
> proof's public inputs — so the AIR does not cryptographically enforce which
> policy/data the proof attests to. Known gap (see
> [divergence D11](../04-divergence/carcosa.md)).

## Trace Layout

`ApprovalTrace` (`trace.rs`): width **3**, length = number of approvers.

| Col | Meaning |
|-----|---------|
| 0 | `has_valid_sig` (1 if sig valid) |
| 1 | `is_authorized` (1 if in mandate list) |
| 2 | `cumulative` = running sum of `has_valid_sig * is_authorized` |

## Transition Constraints

Two degree-2 constraints (`approval_air.rs:46`):

```
result[0] = cumulative_curr - (cumulative_prev + valid_approver)   // monotonic accumulation
result[1] = (cumulative_curr - cumulative_prev) * (cumulative_curr - cumulative_prev - 1)
            // step is either 0 or 1
```

Where `valid_approver = has_valid_sig * is_authorized`.

## Assertions (boundary)

- `col0 @ step0 == 0` (cumulative starts at 0)
- `col0 @ last == threshold` (final cumulative equals threshold)
- `col2 @ step i == 1` for each commitment `i` (authorized approver present)

The threshold boundary assertion makes the proof attest "exactly `threshold`
valid authorized approvals" — note this proves equality, not "≥ threshold".

## Notes

- The trace builder currently hardcodes the last approver's `has_valid_sig = 0`
  (`trace.rs:19`) — a placeholder; a real prover would set it from verified
  signatures.
- `is_authorized` is hardcoded to 1 in the trace builder; real authorization
  must come from the mandate list ([witness](witness.md)).

## References

- [architecture](architecture.md), [proving](proving.md),
  [trace-generation](trace-generation.md)
- [divergence D11](../04-divergence/carcosa.md)
