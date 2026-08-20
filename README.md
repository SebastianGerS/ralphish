# ralphish

A headless agentic task runner built on [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) which can run either Claude Code or Codex in a persistent sandbox.

## Installation

```
./install.sh
```

This builds the Docker image and installs `ralphish` to `~/.local/bin`.

## Setup

Claude mode uses a Claude Code OAuth token:

```
claude setup-token
export CC_TOKEN=<token>
```

Codex mode uses either a local Codex login or an API key:

```
codex login
```

or:

```
export OPENAI_API_KEY=<token>
```

## ralphish

```
ralphish [OPTIONS] [WORKSPACE]
```

Runs the selected agent headlessly in a sandbox, iterating on tasks defined in a `progress.yaml` file in the workspace. Claude is the default; use `--codex` or `--agent codex` to switch.

### Architecture: three-phase iteration loop

Each iteration runs three agent phases on the same persistent sandbox:

```
for each iteration:
    Sync         → fetch upstream, update base branch
    Worker       → implement one task, write summary.yaml
    Reviewer     → review the diff, write review.yaml
    Orchestrator → update progress.yaml, write context.yaml
    check COMPLETE or PAUSE
```

**Worker** picks the next eligible task (or the task assigned by the orchestrator via `context.yaml`), implements it on a feature branch, and writes `.ralphish/summary.yaml` with the task ID, base/head SHAs, branch name, and a description of what was done.

**Reviewer** reads the summary, examines the exact diff (`git diff base_sha..head_sha`), and writes `.ralphish/review.yaml` with a verdict (`approve` or `request_changes`) and specific remarks.

**Orchestrator** reads both artifacts, updates `progress.yaml` (marks tasks done, tracks retries, splits/blocks tasks), appends to the orchestration history log, and writes `.ralphish/context.yaml` with feedback for the next worker iteration. It can signal `COMPLETE` when all tasks are done or `PAUSE` when external action is needed.

### Inter-phase communication

Phase communication uses structured YAML files in `.ralphish/` (transient, wiped per invocation):

| File | Written by | Read by | Content |
|---|---|---|---|
| `summary.yaml` | Worker | Reviewer, Orchestrator | Task ID, base/head SHAs, branch, what was done, findings |
| `review.yaml` | Reviewer | Orchestrator | Verdict, task ID, SHAs, remarks |
| `context.yaml` | Orchestrator | Worker (next iteration) | Assigned task ID, feedback, reviewer remarks |

All durable state lives in `progress.yaml` (task statuses, retry counts, orchestration history).

Per-iteration cost and timing is recorded by the harness itself — not by the agents — as one JSON
line per iteration in `.ralphish-metrics.jsonl` next to `progress.yaml`. Token counts come from the
final `result` (Claude) / `turn.completed` (Codex) event of each phase's stream, so they are exact
and survive a phase that never wrote its YAML artifact.

### Retry and escalation

The orchestrator tracks consecutive review rejections per task via `retry_count` in `progress.yaml`. After 3 consecutive rejections, it escalates:

1. **Split** the task into smaller subtasks (preferred)
2. **Block** the task with an explanation
3. **Skip** the task as a last resort

### Pause and resume

The orchestrator can pause the loop when external action is needed (PR merge, deployment, manual approval). On pause:

1. All durable state is written to `progress.yaml`
2. `ralphish` exits cleanly (exit code 0)
3. The user performs the external action
4. The user runs `ralphish` again to resume

On resume, the sync phase fetches upstream and updates the local base branch. The `.ralphish/` directory is wiped (fresh transient state), but `progress.yaml` picks up where it left off.

### Error handling

All phases are fail-closed:

| Phase | On failure | Detail |
|---|---|---|
| Sync | Continue (best-effort) | Network issues should not block local work |
| Worker | Exit with error | No summary means nothing to review |
| Reviewer | Retry once, then exit | Transient failures get one retry; missing review never silently skipped |
| Orchestrator | Exit with error | State must be updated before next iteration |

### Options

| Flag                   | Description                                                       |
| ---------------------- | ----------------------------------------------------------------- |
| --claude               | Run Claude (default)                                              |
| --codex                | Run Codex                                                         |
| --agent AGENT          | Select `claude` or `codex`                                        |
| --new                  | Remove and recreate sandbox before running                        |
| --rm                   | Remove the sandbox                                                |
| --max-iterations N     | Maximum number of iterations to run (default: 1)                  |
| --env KEY=VALUE        | Pass environment variable to sandbox (can be used multiple times) |
| --env-file FILE        | Load environment variables from a file                            |
| --include-local-config | Copy local agent plugins, skills, and settings into sandbox       |

### Examples

```
ralphish ~/project                           # Run one iteration on ~/project
ralphish --max-iterations 5 .                # Run up to 5 iterations on current directory
ralphish --new --max-iterations 3 .          # Fresh sandbox, up to 3 iterations
ralphish --codex .                           # Run one iteration with Codex instead of Claude
ralphish --agent codex --max-iterations 2 .  # Same, using the generic agent flag
ralphish --rm .                              # Remove sandbox for current directory
ralphish --env GITHUB_TOKEN=ghp_xxxx .       # Pass a secret into the sandbox
ralphish --env-file ~/.ralphish-secrets .    # Load secrets from a file
ralphish --include-local-config .            # Run with your local agent plugins and skills
```

### progress.yaml format

The `progress.yaml` file defines the tasks for `ralphish` to work through. See `example-progress.yaml` for a complete example.

```yaml
next_id: 6
tasks:
  - id: 1
    title: Add rate limiting middleware
    description: |
      Add a token bucket rate limiter as HTTP middleware.
      Configurable per-route limits via environment variables.
      Return 429 with Retry-After header when exceeded.
    status: done
    blocked_by: []
    parent_id: null
    branch: feature/rate-limiting
    head_sha: abc1234
    retry_count: 0
    last_rejection: null
  - id: 4
    title: Add request ID propagation
    description: |
      Generate a UUID request ID in middleware if X-Request-Id header is not present.
      Propagate it via context to all downstream calls and include it in responses.
    status: todo
    blocked_by: [2]
    repo: ../api-service

orchestration:
  last_sync_sha: def5678
  history:
    - iteration: 1
      task_id: 1
      verdict: approve
      summary: "Implemented rate limiter middleware"
      decision: done
```

#### Task fields

| Field            | Description                                                        |
| ---------------- | ------------------------------------------------------------------ |
| `id`             | Unique integer identifier                                                       |
| `title`          | Short name for the task                                                         |
| `description`    | Detailed description of what to implement                                       |
| `status`         | `todo`, `done`, `blocked`, or `skipped`                                         |
| `blocked_by`     | List of task IDs that must be completed before this task can start               |
| `repo`           | Path to the repo for this task (absolute or relative to progress.yaml; see below)|
| `parent_id`      | If split from another task, the parent task's ID (null otherwise)               |
| `branch`         | Feature branch name (null until work starts)                                    |
| `head_sha`       | Last known commit SHA on the feature branch (null until committed)              |
| `retry_count`    | Consecutive review rejections for current attempt (default: 0)                  |
| `last_rejection` | Summary of last rejection reason (null if none)                                 |

`next_id` tracks the next available ID for new tasks. `ralphish` picks the first `todo` task whose `blocked_by` dependencies are all `done`.

New fields (`repo`, `parent_id`, `branch`, `head_sha`, `retry_count`, `last_rejection`) are optional — existing `progress.yaml` files without them work fine (missing fields default to zero values).

#### Repo-scoped sandboxes

When a task has a `repo` field, `ralphish` mounts **only that repo** into the sandbox, limiting what the agent can see and modify. The path can be absolute or relative to the directory containing `progress.yaml`.

```yaml
tasks:
  - id: 1
    title: Add rate limiting
    repo: ../api-service         # sandbox sees only api-service/
    ...
  - id: 2
    title: Update shared types
    repo: /home/user/shared-lib  # absolute path works too
    ...
  - id: 3
    title: Fix tests
    # no repo field — sandbox sees the progress.yaml directory
    ...
```

Task selection happens on the host before creating the sandbox, so each iteration's sandbox is scoped to the correct repo. When the repo differs from the `progress.yaml` directory, `ralphish` syncs `progress.yaml` and `.ralphish/` state in and out of the mounted repo automatically.

#### Orchestration section

| Field                       | Description                                            |
| --------------------------- | ------------------------------------------------------ |
| `orchestration.last_sync_sha` | Base branch SHA at last sync                         |
| `orchestration.history[]`   | Append-only log of iteration outcomes                  |

The `orchestration` section is created automatically on first run if absent.

### Creating tasks with Claude Code skills

The [perhells/skills](https://github.com/perhells/skills) collection includes skills that work well as a pipeline for producing `progress.yaml` tasks:

1. **`/grill-me`** — Stress-test your plan or design through an interactive interview, resolving ambiguities before any code is written.
2. **`/write-a-prd`** — Turn the refined plan into a structured PRD through codebase exploration and module design.
3. **`/prd-to-local-implementation-plan`** — Convert the PRD into concrete implementation tasks in `progress.yaml` format, ready for `ralphish` to execute.

Install the skills and run them in sequence to go from idea to runnable task list.
