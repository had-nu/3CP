# CLI

> **Document Level:** D — Implementation
>
> **Classification:** Reference Implementation
>
> **Status:** Draft
>
> **Version:** 1.0
>
> **Source**
> - Implementation: `carcosa/cli/src/main.rs`, `carcosa/cli/src/commands/*.rs`

---

## Purpose

Document the Carcosa command-line interface and its current behavior.

## Scope

The `carcosa` binary subcommands. Build/test: `cargo build` / `cargo test` in
the `carcosa` workspace.

## Subcommands

| Subcommand | Args | Behavior |
|------------|------|----------|
| `prove` | `--approver-keys --signatures --approval-data --policy-hash --threshold --output` | **Stub** — prints args + `[OK] proof generated` |
| `verify` | `--proof-digest --blob-store --block-idx? --gleipnir-addr?` | **Stub** — prints `[OK] STARK proof verified` / `[OK] 3CP SMT inclusion verified` |
| `anchor` | `--hash --label --mandate-ref? --gleipnir-addr` | **Stub** — prints `[OK] entry anchored` |
| `audit` | `--window-start --window-end --mandate --gleipnir-addr` | **Stub** — prints hardcoded `[NO] COMPLIANCE GAP` |
| `setup` | `--output` | **Stub** — prints `[OK]` (Winterfell params) |

All five subcommands are **print-only scaffolds** — none invoke the Winterfell
prover/verifier or a Gleipnir gRPC client. See [proving](proving.md) and
[audit](audit.md).

## Example (illustrative)

```text
# Illustrative — no real proof is produced
carcosa prove --approver-keys keys.json --signatures sigs.json \
  --approval-data data.json --policy-hash a1b2c3 --threshold 2
```

## References

- [architecture](architecture.md), [proving](proving.md),
  [verification](verification.md), [audit](audit.md)
