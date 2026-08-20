# Ralphish Improvement Ideas

Researched and ranked by expected impact relative to implementation effort.
Based on 2025-2026 research on agentic coding workflows.

---

## Tier 1 — High Impact, Low Effort

### 1. Worker Self-Review Before Summary (DONE)
Add an explicit self-review step to the worker prompt: run `git diff` and verify
changes against task requirements before writing `summary.yaml`. Fix any issues
found. Research (SELF-REFINE) shows this single-step self-check significantly
improves first-attempt pass rate with near-zero cost.

### 2. Test Execution in Reviewer Phase (DONE)
The reviewer currently only reads the diff. Have it also run the test suite and
include results in `review.yaml`. This closes the gap between "looks correct"
and "actually works" — the #1 failure mode in agentic coding (CodeScene 2025).

### 3. Implementation Plan in `context.yaml` (DONE)
When the orchestrator assigns a task, have it write a 2-4 step implementation
plan (not just the task ID and feedback). Research consistently shows that
forcing planning before execution improves first-attempt success by 20-30%.

---

## Tier 2 — High Impact, Medium Effort

### 4. Task Complexity Routing (DONE)
Classify tasks as simple/medium/complex. Route simple tasks (config changes,
dependency bumps, docs) to a lighter/faster model or shorter prompt. Anthropic's
own guidance says this can reduce cost 40-60% on easy tasks without quality loss.

### 5. Cumulative `lessons.yaml` Memory *(low priority — largely superseded by claude-mem + target repo CLAUDE.md)*
Have the orchestrator maintain a running `lessons.yaml` across iterations
capturing patterns like "this repo's tests require running `make build` first"
or "the linter enforces 80-char lines". Feed this to the worker as institutional
memory. Prevents repeated mistakes across iterations. Based on JetBrains
Research (Dec 2025) on observation compression.

### 6. Metrics Collection (DONE)
Track per-task: token usage, iteration count, retry count, wall-clock time,
reviewer verdict. Store in `progress.yaml` or a separate `metrics.yaml`. This
gives you a feedback loop to measure the impact of any prompt changes. Without
it, you're optimizing blind.

*2026-08-20 — token counts are now harness-measured.* The original design had
each phase run `/usage` and diff it; `/usage` does not exist in headless `-p`
mode, so every token field came out `null` for nine iterations. The harness now
parses the final `result` (Claude) / `turn.completed` (Codex) event from the
stream it already tees to `$TMPFILE` and appends one record per iteration to
`.ralphish-metrics.jsonl`. Wall-clock stays in `progress.yaml`; the phase
prompts no longer mention tokens at all.

### 7. Remote Push, PR Creation, and CI Validation (DONE)
After the worker commits, push the feature branch and create a draft PR (idempotent
on retry — skip creation if PR already exists). The reviewer polls `gh pr checks`
until all CI checks complete (10-minute timeout via `--watch`) and records
`ci_status` in `review.yaml`. CI failure is treated as strong evidence for
`request_changes`; the orchestrator will not mark a task done while CI is red.
Requires `GH_TOKEN` in the sandbox env (add to your `.env` file).

### 8. Smarter Retry with Alternative Approaches
Currently, a retry sends the worker the same task + reviewer feedback. Research
on MCTS-based agents (SWE-Search, RethinkMCTS) shows that backtracking to try a
fundamentally different approach beats incremental fixes. Have the orchestrator
suggest an alternative strategy on retry, not just forward the reviewer's
remarks.

---

## Tier 3 — Novel / Ambitious

### 9. Parallel Reviewer Voting
Run 2 reviewer instances in parallel with independent prompts and take the
stricter verdict. Multi-reviewer consensus significantly reduces false approvals
(Anthropic's "parallelization" pattern). Cost doubles for reviewer phase but
catches more bugs.

### 10. Worker Parallel Exploration for Hard Tasks
For tasks that have already been rejected once, run 2-3 worker instances with
different approaches in parallel and have the reviewer pick the best. This is
the "best-of-N" sampling strategy — a practical lightweight version of MCTS.

### 11. Pre-flight Validation Step
Add a Phase 0.5 (after sync, before worker) that validates the workspace state:
tests pass on base branch, dependencies resolve, linter is clean. This prevents
the worker from inheriting a broken baseline and wasting an iteration.

### 12. Dynamic Agent Selection
Currently `--claude` vs `--codex` is a global flag. Allow per-task agent
selection in `progress.yaml` — some tasks might be better suited to one model
vs another. The orchestrator could even learn this from retry patterns.

### 13. Checkpoint/Resume
On `PAUSE`, serialize enough state that `ralphish` can resume exactly where it
left off without the user needing to understand what happened. Include a
human-readable status summary.

---

## Research-Validated Anti-Patterns to Avoid

- **LLM-generated AGENTS.md** — Augment Code's research (2025) found this
  *reduces* success by ~3% and increases cost >20%. Human-curated context files
  are strictly better.
- **Huge context windows** — JetBrains found performance degrades past ~100k
  tokens as agents start repeating actions. Phase isolation (clean prompt per
  phase) is architecturally correct and validated.
- **Big multi-file rewrites** — CodeScene data shows agent failure rates spike
  with change complexity. The "one task per iteration" constraint is right.

---

## References

- Anthropic: Building Effective AI Agents (2025)
- CodeScene: Agentic AI Coding Best Practice Patterns (2025)
- Addy Osmani: The Code Agent Orchestra (2025)
- Addy Osmani: Self-Improving Coding Agents (2025)
- JetBrains Research: Efficient Context Management (Dec 2025)
- Augment Code: How to Build Your AGENTS.md (2025)
- SWE-Search: MCTS for Software Agents (OpenReview, 2025)
- RethinkMCTS: Refining Erroneous Thoughts (arXiv, 2025)
- Empirical-MCTS: Continuous Agent Evolution (arXiv, Feb 2026)
- SELF-REFINE: Iterative Refinement with Self-Feedback (arXiv, 2023)
- Docker: Sandboxes for Coding Agent Safety (2026)
