---
compatibility: opencode
name: issue-orchestrator
description: "Select an issue by trigger label, run a configured pipeline of stages (skills), manage GitHub labels and loop tracking via issue comments, and report results."
disable-model-invocation: false
---

# Issue Orchestrator

## Overview

This skill selects one issue by its trigger label, runs a configured pipeline of stages (each mapping to a skill), manages GitHub labels and loop tracking via issue comments, and reports results.

It is designed to be invoked by a shell script that polls every N minutes. The skill itself has no polling knowledge — it selects one issue, processes it, and exits.

## Invocation

1. Read `config.yaml` from the skill's directory (`skills/issue-orchestrator/config.yaml` relative to the skill root).
2. Query GitHub for issues matching any label in the `labels` section of config.
3. Select one issue at random (stateless: no bias toward any issue).
4. **Guard rail:** Skip the issue if it has the `in-progress` label. This prevents picking up issues that are already being processed (e.g., if the orchestrator crashed mid-run).
5. Add the `in-progress` label to the selected issue.
6. **Branch setup:** Generate the integration branch name from `config.yaml` (`feature/issue-<n>`) and create/checkout it. Never commit to `main`, `master`, `develop`, or `trunk`.
7. Run the configured skill for that issue's label, passing the issue number and branch info as context.
8. Based on the skill's outcome, apply the appropriate transition label (see `transitions` in config).
9. Report results and exit.

## Branch Guardrail

**NEVER commit to `main`, `master`, `develop`, or `trunk`.** Every skill invocation must work on an integration branch.

```yaml
branch:
  base: "main"
  integration: "feature/issue-<n>"
  forbidden: ["main", "master", "trunk", "develop"]
```

- `base` — the target branch all work is compared against (default: `main`).
- `integration` — the pattern for the feature branch. `<n>` is replaced with the issue number (e.g., `feature/issue-42`).
- `forbidden` — branch names the orchestrator must refuse to commit to.

Every skill invocation receives:
- `issue` — the issue number (e.g., `42`)
- `branch` — the integration branch name (e.g., `feature/issue-42`)
- `base_branch` — the base branch (e.g., `main`)

Skills use this to:
- Create PRs against `base_branch`
- Comment on the correct PR (`gh pr comment <pr-number>`)
- Manage labels on the correct issue (`gh issue edit <number>`)
- Compare diffs against `base_branch` (`git diff main...feature/issue-42`)

## Config Structure

Each label in `config.yaml` is a first-class entry:

```yaml
labels:
  ready-for-hunt-bugs:
    skill: hunt-bugs
    transitions:
      - label: ready-for-hardening
        condition: success
      - label: failed-hunt-bugs
        condition: failure
    loop:
      max_iterations: 3
      tracked: true
```

- **`skill`** — which skill to run for this label.
- **`transitions`** — list of possible next labels and their conditions. Each entry has:
  - `label` — the target label to set.
  - `condition` — `success` or `failure`.
  - `when` (optional) — `loop_max_reached` means this transition only applies when the loop limit is exceeded.
- **`loop`** (optional) — only present on labels that can loop back. Contains:
  - `max_iterations` — max number of times this stage can be re-attempted.
  - `tracked` — whether loop depth is tracked via issue comment (always `true`).

## Context Passed to Skills

Before invoking a skill, the orchestrator sets up the context:

1. **Generate the integration branch** from `branch.integration` (e.g., `feature/issue-42`).
2. **Create the branch** from `branch.base` (e.g., `git checkout -b feature/issue-42 main`).
3. **Pass to the skill** via environment variables or agent context:
   ```yaml
   issue: 42
   branch: feature/issue-42
   base_branch: main
   ```

Every skill uses this context to:
- Create PRs: `gh pr create --base main --head feature/issue-42`
- Comment on PRs: `gh pr comment <pr-number> --body "..."`
- Manage labels: `gh issue edit 42 --add-label ready-for-code-simplify`
- Compare diffs: `git diff main...feature/issue-42`

## Loop Tracking

Loop depth is tracked via a YAML block comment on the issue body. The orchestrator **reads this comment from the issue body** to determine how many times the current stage has been attempted.

### Comment Format

```yaml
---
stage: hunt-bugs
loop: 2
started: 2026-11-24T10:30:00Z
on_max_reached: for-human-review
---
```

Fields:
- `stage` — the current label name (e.g., `hunt-bugs`).
- `loop` — current iteration count (starts at 0, increments on each re-attempt).
- `started` — timestamp when the first attempt began.
- `on_max_reached` — the label to apply when `loop >= max_iterations` (e.g., `for-human-review`).

### How the Orchestrator Interprets Loop Config

When the orchestrator finds an issue with a label that has `loop.max_iterations` in config:

1. **Read the issue body** using `gh issue view <number> --json body`.
2. **Parse the YAML block comment** (the text between `---` delimiters).
3. **If no comment exists** — create one with `loop: 0`, `stage: <current-label>`, `started: <now>`, `on_max_reached: <on_max_reached value from config>`.
4. **If comment exists** — read the `loop` value.
5. **Run the skill** for the current label.
6. **On skill outcome:**
   - **Success** → apply the `success` transition label, update the comment with the new `stage` and reset `loop: 0`.
   - **Failure** → increment `loop` by 1, check if `loop >= max_iterations`:
     - If **not exceeded** → apply the `failure` transition label, update the comment with the new `stage` and incremented `loop`.
     - If **exceeded** → apply the `on_max_reached` transition (e.g., `for-human-review`), update the comment with `on_max_reached` label.
7. **Update the issue body** with the new comment using `gh issue edit <number> --body <new-body>`.

### Example: `ready-for-hunt-bugs` Flow

```
Iteration 1: label=ready-for-hunt-bugs, loop=0 → hunt-bugs fails → label=failed-hunt-bugs, loop=1
Iteration 2: label=failed-hunt-bugs, loop=1 → /implement → label=ready-for-code-simplify, loop=0
Iteration 3: label=ready-for-code-simplify → ... → label=ready-for-hunt-bugs, loop=0
Iteration 4: label=ready-for-hunt-bugs, loop=0 → hunt-bugs fails → label=failed-hunt-bugs, loop=1
Iteration 5: label=failed-hunt-bugs, loop=1 → /implement → label=ready-for-code-simplify, loop=0
...
Iteration N: label=ready-for-hunt-bugs, loop=2 → hunt-bugs fails → loop=3 >= 3 → label=for-human-review
```

## Label Management

- **Trigger labels** — make an issue eligible for the orchestrator (e.g., `ready-for-agent`, `failed-hunt-bugs`).
- **In-progress label** — added when the orchestrator starts processing (e.g., `in-progress`).
- **Transition labels** — map between stages (e.g., `ready-for-code-simplify`).
- **Done label** — marks completion (e.g., `ready-for-human`).
- **Human-review label** — marks escalation (e.g., `for-human-review`).
- **Guard rail label** — `in-progress` prevents re-pickup of the same issue.

## GitHub Operations

Uses `gh` CLI for all GitHub interactions:

- `gh issue list --label <trigger> --json number,title,body,labels` — find eligible issues
- `gh issue edit <number> --add-label <label>` — add labels
- `gh issue edit <number> --remove-label <label>` — remove labels
- `gh issue view <number> --json body` — read issue body for loop tracking
- `gh issue edit <number> --body <body>` — update issue body

## Report Format

On completion, report:

```
Issue #42: "Implement feature X"
  Label: ready-for-agent
  Skill: implement
  Next label: ready-for-code-simplify
  Loop depth: 1
  Result: SUCCESS
  Branch: feature/issue-42
```

## Error Handling

- If no eligible issues exist: report "No issues to pick up" and exit.
- If the issue body is corrupted (no loop comment): create one with `loop: 0`.
- If `gh` is not available: report error and exit.
- If a stage skill is not found: escalate to `for-human-review`.
- **`failed-hunt-bugs` retry loop:** If `hunt-bugs` fails, the issue is labeled `failed-hunt-bugs`. The orchestrator picks this up and re-runs `/implement` (the full pipeline restarts). This repeats up to 3 times. If all 3 attempts fail, the issue is labeled `for-human-review`.
- **`failed-spec-review` retry loop:** If `code-review` fails, the issue is labeled `failed-spec-review`. The orchestrator picks this up and re-runs `/implement` (the full pipeline restarts). This repeats up to 3 times. If all 3 attempts fail, the issue is labeled `for-human-review`.
