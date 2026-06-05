# CLAUDE.md — graphed-checkpoint

Defers to the root **`graphed-project/CLAUDE.md`**; the **project plan
(`graphed-project-plan-gated.md`) always wins.** This file distills **milestone M8** (the durable
half) and its guardrails.

## What this repo is

`graphed-checkpoint`: the **content-addressed checkpoint Store + deterministic resume + error
harvesting** for `graphed`. It is the durable side of **M8**; the other half (the serializable,
versioned, byte-identical `DurablePlan` + content-addressed `task_id`) lives in `graphed-core`. This
package consumes that plan and makes a long run **killable and resumable** without redoing work or
double-counting.

> Hard guardrails (plan M8): **local filesystem store only** (no distributed store in MVP) ·
> **single machine** · M8 is checkpoint/resume — analysis **preservation** is M9, not here · the
> canonical durable form is the **serializable IR**, never cloudpickle except for `opaque=True` nodes.

## The pieces

- **`Store`** (`store.py`): `objects/` content-addressed blobs written **atomically** (temp +
  `fsync` + `rename`, so a torn write is never visible); an append-only **`journal.log`** manifest
  (replayed on resume; a torn trailing line is ignored, an entry whose blob is missing is not
  honored); an append-only **`dead_letter.log`**. The blob name *is* its SHA-256 → idempotent,
  cache-poisoning-safe.
- **`run_resumable`** (`runner.py`): for each partition, compute the plan's content-addressed
  `task_id`; if it is already journaled with a present blob, **skip** `process` and reuse the
  partial (logged in `ResumeReport.skipped`). The final reduction recombines **per-task partials in
  deterministic task order** (never a persisted running accumulator) → a resumed run matches an
  uninterrupted one **bit-for-bit**, with **no double-count and no lost partition**.
- **Retry policies** (`retry.py`): `RetryN`, `RetrySmallerChunk` (splits a partition and recombines
  — recovers a chunk too big to process whole), `RetryElsewhere`, `Quarantine`. An **error budget**
  is a stopping condition (`ResumeReport.stopped == "error_budget"`).
- **Error harvesting** (`errors.py`): a failed partition becomes a reproducible dead-letter
  descriptor; a `graphed_debug.StageError` (M6) contributes its user-source provenance (duck-typed,
  so graphed-debug stays a soft dependency).
- **Codecs** (`codec.py`): deterministic value↔bytes (`PickleCodec` default; `NumpyCodec` `.npy`).

## Layout

```
src/graphed_checkpoint/store.py     content-addressed Store (atomic blobs + journal + dead-letter)
src/graphed_checkpoint/runner.py    run_resumable + ResumeReport/ResumeResult
src/graphed_checkpoint/retry.py     retry_n / retry_smaller_chunk / retry_elsewhere / quarantine
src/graphed_checkpoint/errors.py    dead_letter_descriptor (harvests StageError provenance)
src/graphed_checkpoint/codec.py     PickleCodec + NumpyCodec
tests/frozen/m8/                    the M8 acceptance suite (kill/resume, dead-letter, retry, no-source)
```

## Gates (run before pushing)

`ruff check .` + `ruff format --check .` · `mypy` (strict) · `pytest tests/frozen --cov=graphed_checkpoint
--cov-branch` (≥90%) · `sphinx-build -W docs docs/_build/html`. CI installs the sibling packages from
`git+…@main`, so push `graphed-core` first when the plan/IR contract changes.

Status: see `.graphed/state.json`.
