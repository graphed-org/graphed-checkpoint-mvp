
## 2026-06-10 — freeze-M8-1 (user-authorized respin, graphed-core freeze-M22-1)

- mark_output was removed from graphed-core's public API (outputs are per compile request).
  The two frozen m8 helpers here (analyses.py / test_no_source.py) respun to
  serialize(outputs=[...]) — behavior and bytes unchanged for these single-output graphs.
