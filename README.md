# graphed-checkpoint

Content-addressed checkpoint store, deterministic resume, and error harvesting for
[`graphed`](https://github.com/graphed-org/graphed-project-mvp) — **milestone M8** (the durable
half). The other half — the serializable, versioned, byte-identical `DurablePlan` and its
content-addressed `task_id` — lives in `graphed-core`. This package consumes that plan and makes a
long run **killable and resumable** without redoing work or double-counting.

Local filesystem + single machine only (the M8 guardrail). Analysis *preservation* is M9, not here.

## Why content addressing

Every unit of work is keyed by *what it computes*, never by when or where it ran. A `DurablePlan`'s
`task_id(partition)` is a SHA-256 over the canonical analysis IR, the process spec, and the
partition. Two things fall out for free:

- **Resume is a lookup, not a protocol.** On a fresh run every task id is missing, so everything
  executes and each partial is committed as it finishes. After a crash the same plan recomputes the
  same task ids; whatever was already committed is loaded instead of recomputed. There is no resume
  token to reconcile — the store *is* the state.
- **Cache poisoning is structurally hard.** Change the analysis (different IR bytes), the processing
  function, or the partition and the key changes. A stale store can never satisfy a different
  computation; at worst it is ignored. The blob name *is* its SHA-256.

## The pieces

- **`Store`** (`store.py`) — a local-filesystem store with three durable parts: `objects/`
  content-addressed blobs written **atomically** (temp file + `fsync` + `rename`, so a torn write is
  never visible and re-running is idempotent); an append-only **`journal.log`** manifest of completed
  tasks (replayed on resume — a torn trailing line is ignored, and an entry whose blob is missing is
  not honored); and an append-only **`dead_letter.log`**.
- **`run_resumable`** (`runner.py`) — for each partition, compute its `task_id`; if it is already
  journaled with a present blob, **skip** `process` and reuse the partial. The final reduction
  recombines **per-task partials in deterministic task order** (never a persisted running
  accumulator), so a resumed run matches an uninterrupted one **bit-for-bit** — no double-count, no
  lost partition. The `ResumeReport` is the audit trail (`executed` / `skipped` / `dead` / `stopped`
  / `dead_letters`, plus `did_less_work`).
- **Retry policies** (`retry.py`) — `RetryN` (transient hiccup), `RetrySmallerChunk` (splits a
  too-big partition and recombines, recovering a chunk that OOMed whole), `RetryElsewhere` (re-run on
  a fresh worker context), and `Quarantine` (send straight to the dead-letter set). An **error
  budget** is a stopping condition (`ResumeReport.stopped == "error_budget"`).
- **Error harvesting** (`errors.py`) — a failed partition becomes a reproducible, JSON-serializable
  `dead_letter_descriptor`. If the failure is a `graphed_debug.StageError` (M6) its user-source
  provenance is captured too (duck-typed, so graphed-debug stays a soft dependency), so the dead
  letter points at the user's analysis line, not an opaque worker string.
- **Codecs** (`codec.py`) — deterministic value↔bytes: `PickleCodec` (default, protocol pinned) and
  `NumpyCodec` (a fixed `.npy` layout for the common array-partial case).

## A resumed run, end to end

```python
from graphed_core import DurablePlan, OpSpec, Partition
from graphed_checkpoint import Store, run_resumable

plan = DurablePlan(
    ir=analysis_ir_bytes,                       # session.serialized_ir(out) in real use
    process=OpSpec.from_ref("myanalysis:hist_chunk"),
    combine=OpSpec.from_ref("myanalysis:hist_add"),
    empty=OpSpec.from_ref("myanalysis:hist_empty"),
    partitions=tuple(Partition("toy", "", i*1000, (i+1)*1000) for i in range(6)),
    read_columns=("x",),
)

store = Store("checkpoints/")
res   = run_resumable(plan, store)
res.report.executed                  # 6 — everything ran, six partials committed

again = run_resumable(plan, store)   # same plan, same store
again.report.skipped                 # 6 — the run is a pure lookup; again.value == res.value
```

Because `process`/`combine`/`empty` are import refs and the IR is canonical bytes, a serialized plan
runs in a fresh interpreter with only installed packages present — no analysis source files needed.

See [`docs/design.rst`](docs/design.rst) for the engineering walkthrough, `CONTRIBUTING.md` for the
local gate panel, and `CLAUDE.md` for the milestone digest.
