# graphed-checkpoint

Content-addressed checkpoint store, deterministic resume, and error harvesting for
[`graphed`](https://github.com/graphed-org/graphed-project-mvp) — **milestone M8** (the durable half;
the serializable `DurablePlan` + content-addressed `task_id` live in `graphed-core`).

- **Crash-safe `Store`** — content-addressed blobs (atomic temp+fsync+rename), an append-only
  manifest/journal, and a dead-letter set, all on the local filesystem.
- **`run_resumable`** — kill a run and resume it: already-completed tasks are skipped (measurably
  less work), and per-task partials are recombined in deterministic order, so the resumed result
  equals an uninterrupted run **bit-for-bit** — no double-count, no lost partition.
- **Error harvesting** — failed partitions become reproducible dead-letter descriptors (with M6
  `StageError` provenance); retry policies `retry_n` / `retry_smaller_chunk` / `retry_elsewhere` /
  `quarantine`; an error budget is a stopping condition.

Local filesystem + single machine only (the M8 guardrail). Analysis *preservation* is M9.

See `CONTRIBUTING.md` for the local gate panel and `CLAUDE.md` for the design digest.
