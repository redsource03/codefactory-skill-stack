---
name: implement
description: "Implement a piece of work based on a spec or set of tickets."
disable-model-invocation: false
---

Implement the work described by the user in the spec or tickets.

If the user passes a ticket reference, fetch it from the issue tracker and state its title before starting. If the reference is ambiguous, ask.

**Check for existing PR comments:** Before starting, check if an open PR already exists for this ticket. If it does, read the latest comment on the PR to understand what changes are needed. Use that comment as the working spec — the agent should implement exactly what the latest comment requests, not just the original ticket.

If a PR exists but has no comments, proceed with the original ticket.

## GitHub Operations

Uses `gh` CLI for all GitHub interactions:

- `gh pr list --json number,title,body,headRefName` — find open PRs for the current branch
- `gh pr view <number> --json comments` — read PR comments
- `gh pr comment <number> --body <body>` — add a comment to the PR
- `gh issue edit <number> --add-label <label>` — add labels
- `gh issue edit <number> --remove-label <label>` — remove labels
- `gh issue view <number> --json body` — read issue body
- `gh issue edit <number> --body <body>` — update issue body

Call the Skill tool with "tdd" where possible, at pre-agreed seams.

Run typechecking regularly, single test files regularly, and the full test suite once at the end.

Once done, call the Skill tool with "pr" to create a pull request for the work.

Commit your work to the current branch.
