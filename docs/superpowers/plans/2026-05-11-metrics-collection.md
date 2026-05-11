# Metrics Collection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Track per-phase token usage and wall-clock time for every ralphish iteration, stored in `orchestration.history` entries and a top-level `metrics` summary in `progress.yaml`.

**Architecture:** The bash main loop records `$SECONDS` around each `run_phase` call and injects `{{WORKER_SECONDS}}` and `{{REVIEWER_SECONDS}}` into the orchestrator prompt. The worker and reviewer agents each run `/usage` before and after their work, write the token diff into their output YAMLs, and the orchestrator aggregates everything (plus its own self-timed seconds via `date +%s`) into `orchestration.history` and a top-level `metrics` section.

**Tech Stack:** bash, YAML, Claude Code `/usage` slash command, `date +%s`

---

## File Map

- **Modify:** `ralphish` (single file, all changes)
  - Lines ~651–684: main loop — add `$SECONDS` timing around each `run_phase` call and template substitution for timing vars
  - Lines ~305–365: `WORKER_PROMPT` — add `/usage` bookend steps and `tokens` block in `summary.yaml` schema
  - Lines ~367–416: `REVIEWER_PROMPT` — add `/usage` bookend steps and `tokens` block in `review.yaml` schema
  - Lines ~418–491: `ORCHESTRATOR_PROMPT` — add self-timing via `date +%s`, read phase tokens, write `wall_clock_seconds` + `tokens` into history, maintain top-level `metrics`
- **Modify:** `IMPROVEMENTS.md` — mark item #6 as DONE

---

## Task 1: Bash timing — wrap run_phase calls with $SECONDS

**Files:**
- Modify: `ralphish:651-684`

- [ ] **Step 1: Add timing around the worker phase**

In the main loop, find:
```bash
        # Phase 1: Worker — implement one task, write summary.yaml
        worker_prompt="${WORKER_PROMPT//\{\{ITERATION\}\}/$i}"
        worker_model=$(resolve_worker_model "$CONFIG_DIR/.ralphish/context.yaml")
        if ! run_phase "worker" "$worker_prompt" "$worker_model"; then
```

Replace with:
```bash
        # Phase 1: Worker — implement one task, write summary.yaml
        worker_prompt="${WORKER_PROMPT//\{\{ITERATION\}\}/$i}"
        worker_model=$(resolve_worker_model "$CONFIG_DIR/.ralphish/context.yaml")
        _t0=$SECONDS
        if ! run_phase "worker" "$worker_prompt" "$worker_model"; then
```

Then find the block immediately after the worker `run_phase` failure check (the `if ! sandbox_exec "test -f .ralphish/summary.yaml"` block) and add `WORKER_SECONDS=$(( SECONDS - _t0 ))` after the if-block closes:

```bash
        if ! sandbox_exec "test -f .ralphish/summary.yaml" 2>/dev/null; then
            sync_state_out
            echo "Worker phase did not produce .ralphish/summary.yaml"
            exit 1
        fi
        WORKER_SECONDS=$(( SECONDS - _t0 ))
```

- [ ] **Step 2: Add timing around the reviewer phase**

Find:
```bash
        # Phase 2: Reviewer — review the diff, write review.yaml (retry once)
        reviewer_ok=false
        for attempt in 1 2; do
```

Replace with:
```bash
        # Phase 2: Reviewer — review the diff, write review.yaml (retry once)
        reviewer_ok=false
        _t0=$SECONDS
        for attempt in 1 2; do
```

Then find where `reviewer_ok` is checked after the loop and add the timing line after:
```bash
        if [ "$reviewer_ok" = false ]; then
            sync_state_out
            echo "Reviewer phase failed after retry"
            exit 1
        fi
        REVIEWER_SECONDS=$(( SECONDS - _t0 ))
```

- [ ] **Step 3: Inject timing variables into the orchestrator prompt**

Find:
```bash
        # Phase 3: Orchestrator — update progress.yaml, write context.yaml
        orch_prompt="${ORCHESTRATOR_PROMPT//\{\{ITERATION\}\}/$i}"
        orch_rc=0
```

Replace with:
```bash
        # Phase 3: Orchestrator — update progress.yaml, write context.yaml
        orch_prompt="${ORCHESTRATOR_PROMPT//\{\{ITERATION\}\}/$i}"
        orch_prompt="${orch_prompt//\{\{WORKER_SECONDS\}\}/$WORKER_SECONDS}"
        orch_prompt="${orch_prompt//\{\{REVIEWER_SECONDS\}\}/$REVIEWER_SECONDS}"
        orch_rc=0
```

- [ ] **Step 4: Verify the substitution works**

Temporarily add `echo "DEBUG: worker=${WORKER_SECONDS}s reviewer=${REVIEWER_SECONDS}s" >&2` after the substitution lines, run `ralphish` against a trivial task in a test repo, confirm non-zero values appear. Then remove the debug line.

- [ ] **Step 5: Commit**

```bash
git add ralphish
git commit -m "feat: record wall-clock seconds per phase in main loop"
```

---

## Task 2: Worker prompt — /usage token tracking

**Files:**
- Modify: `ralphish:305-365` (`WORKER_PROMPT`)

- [ ] **Step 1: Add /usage snapshot as the very first instruction**

The worker prompt currently starts at step 1 with "Check if `.ralphish/context.yaml` exists." Renumber existing steps 1–8 to 2–9 and add a new step 1:

```
1. **Token baseline**: Run `/usage` and record the four token counts as your baseline:
   `input_baseline`, `output_baseline`, `cache_read_baseline`, `cache_write_baseline`.
   If `/usage` output cannot be parsed, set all baselines to 0.
```

- [ ] **Step 2: Add /usage diff and tokens block to summary.yaml step**

The final step currently says "Write `.ralphish/summary.yaml` with this exact structure". Add the `/usage` diff instruction before it and add `tokens` to the schema:

Change the step (currently step 8, now step 9) from:

```
8. Write `.ralphish/summary.yaml` with this exact structure:
   ```yaml
   task_id: <integer>
   task_title: "<string>"
   base_sha: "<40-char sha>"
   head_sha: "<40-char sha>"
   branch: "<branch-name>"
   pr_url: "<GitHub PR URL, or empty string if push/PR creation failed>"
   what_was_done: "<description of implementation>"
   findings:
     - "<finding 1>"
   suggestions:
     - "<suggestion 1>"
   ```
```

To:

```
9. **Token diff then write `.ralphish/summary.yaml`**:
   Run `/usage` again. Subtract your baselines from the current values to get the diff.
   The `/usage` output looks like: `Tokens: 5,234 input · 1,123 output · 45,230 cache read · 2,341 cache write`
   Parse the four numbers. If you cannot parse them, write `tokens: null`.
   Write `.ralphish/summary.yaml` with this exact structure:
   ```yaml
   task_id: <integer>
   task_title: "<string>"
   base_sha: "<40-char sha>"
   head_sha: "<40-char sha>"
   branch: "<branch-name>"
   pr_url: "<GitHub PR URL, or empty string if push/PR creation failed>"
   what_was_done: "<description of implementation>"
   findings:
     - "<finding 1>"
   suggestions:
     - "<suggestion 1>"
   tokens:
     input: <integer or null>
     output: <integer or null>
     cache_read: <integer or null>
     cache_write: <integer or null>
   ```
```

- [ ] **Step 3: Add tokens to the MUST list**

In the `You MUST:` block at the end of `WORKER_PROMPT`, add:
```
- Run /usage before starting work and after finishing to compute your token diff
- Include the tokens block in summary.yaml (write tokens: null if /usage cannot be parsed)
```

- [ ] **Step 4: Commit**

```bash
git add ralphish
git commit -m "feat: worker prompt records token usage via /usage diff"
```

---

## Task 3: Reviewer prompt — /usage token tracking

**Files:**
- Modify: `ralphish:367-416` (`REVIEWER_PROMPT`)

- [ ] **Step 1: Add /usage snapshot as the very first instruction**

The reviewer prompt currently starts at step 1 with "Read `.ralphish/summary.yaml`". Renumber existing steps 1–7 to 2–8 and add a new step 1:

```
1. **Token baseline**: Run `/usage` and record the four token counts as your baseline:
   `input_baseline`, `output_baseline`, `cache_read_baseline`, `cache_write_baseline`.
   If `/usage` output cannot be parsed, set all baselines to 0.
```

- [ ] **Step 2: Add /usage diff and tokens block to review.yaml step**

The final step currently says "Write `.ralphish/review.yaml` with this exact structure". Change the step (currently step 7, now step 8) from:

```
7. Write `.ralphish/review.yaml` with this exact structure:
   ```yaml
   task_id: <integer>
   base_sha: "<40-char sha>"
   head_sha: "<40-char sha>"
   verdict: approve  # or: request_changes
   test_results: "<pass/fail summary, or 'no test suite found'>"
   ci_status: "<passing|failing|timed_out|no_checks>"
   ci_details: "<names of failing CI checks, or empty>"
   remarks:
     - "<specific remark — reference files and line numbers>"
   ```
```

To:

```
8. **Token diff then write `.ralphish/review.yaml`**:
   Run `/usage` again. Subtract your baselines from the current values to get the diff.
   The `/usage` output looks like: `Tokens: 5,234 input · 1,123 output · 45,230 cache read · 2,341 cache write`
   Parse the four numbers. If you cannot parse them, write `tokens: null`.
   Write `.ralphish/review.yaml` with this exact structure:
   ```yaml
   task_id: <integer>
   base_sha: "<40-char sha>"
   head_sha: "<40-char sha>"
   verdict: approve  # or: request_changes
   test_results: "<pass/fail summary, or 'no test suite found'>"
   ci_status: "<passing|failing|timed_out|no_checks>"
   ci_details: "<names of failing CI checks, or empty>"
   remarks:
     - "<specific remark — reference files and line numbers>"
   tokens:
     input: <integer or null>
     output: <integer or null>
     cache_read: <integer or null>
     cache_write: <integer or null>
   ```
```

- [ ] **Step 3: Add tokens to the MUST list**

In the `You MUST:` block at the end of `REVIEWER_PROMPT`, add:
```
- Run /usage before starting work and after finishing to compute your token diff
- Include the tokens block in review.yaml (write tokens: null if /usage cannot be parsed)
```

- [ ] **Step 4: Commit**

```bash
git add ralphish
git commit -m "feat: reviewer prompt records token usage via /usage diff"
```

---

## Task 4: Orchestrator prompt — aggregate metrics and write to progress.yaml

**Files:**
- Modify: `ralphish:418-491` (`ORCHESTRATOR_PROMPT`)

This is the largest change. The orchestrator needs to:
- Self-time using `date +%s`
- Self-report tokens via `/usage`
- Read phase tokens from `summary.yaml` and `review.yaml`
- Receive `{{WORKER_SECONDS}}` and `{{REVIEWER_SECONDS}}` from the shell
- Write extended `orchestration.history` entry
- Maintain top-level `metrics` section in `progress.yaml`

- [ ] **Step 1: Add self-timing and token baseline as the very first instruction**

The orchestrator prompt currently starts with "Current iteration: {{ITERATION}}" then "## Instructions" with step 1 being "Read `.ralphish/summary.yaml`...". Add two new steps before the existing step 1, and renumber existing steps 1–6 to 3–8:

```
## Timing and token tracking

Before doing anything else:
1. Record the current epoch: run `date +%s` and save the result as `_orch_start`.
2. **Token baseline**: Run `/usage` and record the four token counts as your baseline:
   `input_baseline`, `output_baseline`, `cache_read_baseline`, `cache_write_baseline`.
   If `/usage` output cannot be parsed, set all baselines to 0.

## Instructions
```

- [ ] **Step 2: Update the header to include timing template variables**

The orchestrator prompt header currently reads:
```
Current iteration: {{ITERATION}}
```

Change to:
```
Current iteration: {{ITERATION}}
Worker wall-clock seconds: {{WORKER_SECONDS}}
Reviewer wall-clock seconds: {{REVIEWER_SECONDS}}
```

- [ ] **Step 3: Replace the orchestration.history append step**

Find the existing history append step (currently step 4, now step 6 after renumbering):

```
4. Ensure `progress.yaml` has an `orchestration` section (create it if missing). Append to `orchestration.history`:
   ```yaml
   - iteration: {{ITERATION}}
     task_id: <id>
     verdict: <approve|request_changes>
     ci_status: <passing|failing|timed_out|no_checks>
     summary: "<what happened>"
     decision: "<what you decided: done|retry|split|blocked|skipped>"
   ```
```

Replace with:

```
6. **Token diff and self-timing**: Run `/usage` again. Subtract your baselines to get your own token diff.
   Run `date +%s` and subtract `_orch_start` to get `orchestrator_seconds`.
   The `/usage` output looks like: `Tokens: 5,234 input · 1,123 output · 45,230 cache read · 2,341 cache write`
   If you cannot parse `/usage`, write `null` for your token fields.
   Read `tokens` from `summary.yaml` (worker) and `review.yaml` (reviewer) — use `null` if missing.

7. Ensure `progress.yaml` has an `orchestration` section (create it if missing). Append to `orchestration.history`:
   ```yaml
   - iteration: {{ITERATION}}
     task_id: <id>
     verdict: <approve|request_changes>
     ci_status: <passing|failing|timed_out|no_checks>
     summary: "<what happened>"
     decision: "<what you decided: done|retry|split|blocked|skipped>"
     wall_clock_seconds:
       worker: {{WORKER_SECONDS}}
       reviewer: {{REVIEWER_SECONDS}}
       orchestrator: <orchestrator_seconds integer>
       total: <sum of all three>
     tokens:
       worker:       { input: <int or null>, output: <int or null>, cache_read: <int or null>, cache_write: <int or null> }
       reviewer:     { input: <int or null>, output: <int or null>, cache_read: <int or null>, cache_write: <int or null> }
       orchestrator: { input: <int or null>, output: <int or null>, cache_read: <int or null>, cache_write: <int or null> }
       total:        { input: <sum or null>, output: <sum or null>, cache_read: <sum or null>, cache_write: <sum or null> }
   ```
   For `total` token fields: if ANY phase has `null` for a field, write `null` for that total field too.
```

- [ ] **Step 4: Add the metrics section maintenance step**

After the history append step, add a new step (before the "Determine next action" step):

```
8. Ensure `progress.yaml` has a top-level `metrics` section (create it if missing with all counters at 0). Update it:
   - Increment `total_iterations` by 1
   - Add this iteration's `wall_clock_seconds.total` to `total_wall_clock_seconds` (skip if null)
   - For each token field (input, output, cache_read, cache_write): add this iteration's `tokens.total.<field>` to `total_tokens.<field>` (skip if null)
   - Upsert the `per_task` entry for `task_id` (create if missing, update if exists):
     - If the task was approved this iteration: set `iterations` to its final retry_count + 1, set `wall_clock_seconds` and `tokens` to this iteration's values
     - If retrying: increment `iterations` by 1, add to cumulative `wall_clock_seconds` and `tokens`

   The `metrics` section shape:
   ```yaml
   metrics:
     total_iterations: <integer>
     total_wall_clock_seconds: <integer>
     total_tokens:
       input: <integer or null>
       output: <integer or null>
       cache_read: <integer or null>
       cache_write: <integer or null>
     per_task:
       - task_id: <integer>
         iterations: <integer>
         wall_clock_seconds: <integer>
         tokens:
           input: <integer or null>
           output: <integer or null>
           cache_read: <integer or null>
           cache_write: <integer or null>
   ```
```

- [ ] **Step 5: Add metrics obligations to the MUST list**

In the `You MUST:` block at the end of `ORCHESTRATOR_PROMPT`, add:
```
- Run /usage before starting work and after finishing to compute your own token diff
- Record date +%s at start and end to compute orchestrator_seconds
- Always write wall_clock_seconds and tokens into the orchestration.history entry (use null for unparseable values)
- Always upsert the per_task entry and update top-level accumulators in the metrics section
```

- [ ] **Step 6: Commit**

```bash
git add ralphish
git commit -m "feat: orchestrator aggregates metrics into history and progress.yaml metrics section"
```

---

## Task 5: Mark IMPROVEMENTS.md item #6 as DONE

**Files:**
- Modify: `IMPROVEMENTS.md`

- [ ] **Step 1: Update the heading**

Find:
```markdown
### 6. Metrics Collection
```

Replace with:
```markdown
### 6. Metrics Collection (DONE)
```

- [ ] **Step 2: Commit**

```bash
git add IMPROVEMENTS.md
git commit -m "docs: mark metrics collection (item #6) as done"
```
