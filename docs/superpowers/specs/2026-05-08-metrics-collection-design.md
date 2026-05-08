# Metrics Collection Design

**Date:** 2026-05-08
**Feature:** IMPROVEMENTS.md item #6 — Metrics Collection

---

## Goal

Track per-task and per-iteration metrics (token usage, wall-clock time, retry depth, model tier, reviewer verdict) so that prompt changes and model tier decisions can be evaluated against real data rather than intuition.

---

## Approach

**Option B selected:** Bash script captures wall-clock time via `$SECONDS` at each `run_phase` call site. Each agent (worker, reviewer, orchestrator) self-reports token usage by running `/usage` at the start and end of its work and writing the diff into its output YAML. The orchestrator aggregates all three agents' data into `orchestration.history` and maintains a top-level `metrics` summary in `progress.yaml`.

Since agents share a Claude Code session across phases, the before/after `/usage` diff is necessary to isolate each phase's token consumption.

---

## Data Flow

1. Bash records `$SECONDS` before each `run_phase` call, computes duration after it returns, stores in `WORKER_SECONDS`, `REVIEWER_SECONDS`, `ORCHESTRATOR_SECONDS`.
2. Worker runs `/usage` as its first action (baseline), does its work, runs `/usage` again at the end, writes diff to `summary.yaml` as a `tokens` block.
3. Reviewer does the same, writes diff to `review.yaml`.
4. Orchestrator does the same for itself, then reads tokens from `summary.yaml` and `review.yaml`, receives timing via `{{WORKER_SECONDS}}` / `{{REVIEWER_SECONDS}}` / `{{ORCHESTRATOR_SECONDS}}` template variables, and writes the full metrics into `orchestration.history` and the top-level `metrics` section.

---

## Schema

### `summary.yaml` (worker adds)

```yaml
tokens:
  input: 1234
  output: 567
  cache_read: 8901
  cache_write: 234
```

### `review.yaml` (reviewer adds)

```yaml
tokens:
  input: 890
  output: 123
  cache_read: 4567
  cache_write: 89
```

### `orchestration.history` entry (extended)

```yaml
- iteration: 3
  task_id: 2
  verdict: approve
  ci_status: passing
  summary: "..."
  decision: done
  wall_clock_seconds:
    worker: 142
    reviewer: 67
    orchestrator: 45
    total: 254
  tokens:
    worker:      { input: 1234, output: 567,  cache_read: 8901,  cache_write: 234 }
    reviewer:    { input: 890,  output: 123,  cache_read: 4567,  cache_write: 89  }
    orchestrator: { input: 450, output: 89,   cache_read: 2100,  cache_write: 45  }
    total:       { input: 2574, output: 779,  cache_read: 15568, cache_write: 368 }
```

### Top-level `metrics` section in `progress.yaml`

```yaml
metrics:
  total_iterations: 5
  total_wall_clock_seconds: 1240
  total_tokens:
    input: 12500
    output: 3800
    cache_read: 78000
    cache_write: 1800
  per_task:
    - task_id: 1
      iterations: 1
      wall_clock_seconds: 254
      tokens: { input: 2574, output: 779, cache_read: 15568, cache_write: 368 }
```

`per_task` entries are upserted by the orchestrator — if a task is retried, the entry is updated (not appended) to reflect cumulative totals for that task.

---

## Bash Changes

At the three `run_phase` call sites in the main loop:

```bash
_t0=$SECONDS
run_phase "worker" ...
WORKER_SECONDS=$(( SECONDS - _t0 ))

_t0=$SECONDS
run_phase "reviewer" ...
REVIEWER_SECONDS=$(( SECONDS - _t0 ))

_t0=$SECONDS
run_phase "orchestrator" ...
ORCHESTRATOR_SECONDS=$(( SECONDS - _t0 ))
```

The three variables are injected into the orchestrator prompt via template substitution alongside `{{ITERATION}}`.

---

## Prompt Changes

### Worker prompt

- **Step 0 (new, before all other steps):** Run `/usage`. Record the four values (`input`, `output`, `cache_read`, `cache_write`) as the baseline.
- **Final step (before writing `summary.yaml`):** Run `/usage` again. Subtract baseline from current values. Write the diff as a `tokens` block in `summary.yaml`.

### Reviewer prompt

- Same pattern as worker: `/usage` snapshot at the very start, `/usage` diff written into `review.yaml` as a `tokens` block at the very end.

### Orchestrator prompt

- Same `/usage` bookends for its own token tracking.
- Reads `tokens` from `summary.yaml` and `review.yaml`.
- Receives `{{WORKER_SECONDS}}`, `{{REVIEWER_SECONDS}}`, `{{ORCHESTRATOR_SECONDS}}` from the shell.
- Writes `wall_clock_seconds` and `tokens` (per-phase + total) into the `orchestration.history` entry.
- Upserts the `per_task` entry for the current task in the top-level `metrics` section, and updates `total_iterations`, `total_wall_clock_seconds`, and `total_tokens` accumulators.

---

## Error Handling

- If `/usage` output cannot be parsed (unexpected format), the agent writes `tokens: null` and continues — metrics are best-effort and must not block task completion.
- If `summary.yaml` or `review.yaml` lacks a `tokens` block, the orchestrator writes `null` for that phase's entry rather than failing.
- Wall-clock timing is always available from the bash layer regardless of agent behaviour.
