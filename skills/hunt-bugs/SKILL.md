---
name: hunt-bugs
description: "Systematically hunt bugs, regressions, and code quality issues in the integration branch's changes against base_branch. Uses gh CLI to diff files, then performs targeted analysis. Configurable effort level: Low, Medium (default), High."
disable-model-invocation: false
---

# Hunt Bugs

Systematically hunt bugs, regressions, and code quality issues in the integration branch's changes against the base branch.

## Invocation

This skill is invoked by the issue-orchestrator after `code-simplify` completes. It receives:
- `issue` — the issue number (e.g., `42`)
- `branch` — the integration branch (e.g., `feature/issue-42`)
- `base_branch` — the base branch (e.g., `main`)

## Effort Levels

The orchestrator passes an `effort` value. Choose your approach accordingly:

| Level | What you do |
|-------|-------------|
| **Low** | Quick scan — diff files, check obvious issues (typos, missing null checks, unhandled promises). Report findings as a comment on the PR. |
| **Medium (default)** | Thorough review — read changed files end-to-end, trace data flow, check edge cases, verify error handling, test coverage, and type safety. |
| **High** | Deep audit — manual code walkthrough of every changed file, re-verify logic against the issue spec, check for race conditions, resource leaks, and subtle regressions. |

## Process

### 1. Discover Changed Files

Use `gh` to get the diff between `base_branch` and `branch`:

```bash
# Get list of changed files
gh pr diff <pr-number> --stat
# or for the branch directly:
git diff <base_branch>...<branch> --stat
```

Get the full diff of each changed file:

```bash
git diff <base_branch>...<branch> -- <file>
```

### 2. Analyze by Effort Level

**Low effort — scan for:**
- Missing null/undefined checks
- Unhandled promises (no `.catch()` or `try/catch`)
- Typos in variable names (e.g., `resut` vs `result`)
- Obvious logic errors (e.g., `==` instead of `===`)
- Missing `await` on async calls
- Off-by-one errors in loops

**Medium effort — everything above plus:**
- Trace data flow: does input reach all code paths?
- Check edge cases: empty arrays, zero, null, undefined, max values
- Verify error handling: are all branches handled?
- Check type safety: are types consistent?
- Review test coverage: do tests match the changed logic?
- Check for race conditions: async operations without proper sequencing
- Verify resource cleanup: file handles, event listeners, timers
- Check for subtle regressions: does the change break existing contracts?

**High effort — everything above plus:**
- Manual line-by-line walkthrough of every changed file
- Re-verify logic against the original issue specification
- Check for race conditions in complex async scenarios
- Look for resource leaks (memory, file descriptors, connections)
- Verify security implications of the changes
- Check for subtle type narrowing issues
- Review for potential denial-of-service patterns (unbounded loops, missing bounds)

### 3. Report Findings

If bugs are found:
- Comment on the PR (`gh pr comment <pr-number>`) with a structured report:
  - Severity: Critical / Major / Minor
  - File and line reference
  - Description of the bug
  - Suggested fix
- Add the `failed-hunt-bugs` label to the issue (the orchestrator will restart from `implement`)

If no bugs are found:
- Add a comment confirming the review passed
- Signal `success` to the orchestrator so it transitions to `ready-for-hardening`

## Bug Categories to Check

- **Logic bugs** — wrong condition, wrong operator, wrong branch
- **Null/undefined** — missing guards, wrong default value
- **Async** — missing await, race conditions, unhandled rejections
- **Resource leaks** — unclosed handles, unremoved listeners, memory leaks
- **Type errors** — wrong type, missing type guard, wrong generic
- **Security** — injection, XSS, path traversal, auth bypass
- **Performance** — O(n²) where O(n) is possible, unnecessary allocations
- **Edge cases** — empty input, boundary values, overflow

## Output

After the review:
- Low effort: leave a PR comment with findings (or "LGTM" if clean)
- Medium/High effort: leave a structured PR comment with findings and severity
- If bugs found → signal `failure` (orchestrator adds `failed-hunt-bugs` label)
- If clean → signal `success` (orchestrator transitions to `ready-for-hardening`)
