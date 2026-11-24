# ADR FORMAT

Each ADR is a Markdown file at `docs/adr/<NNNN>-<slug>.md`:

```markdown
---
title: <Title>
date: YYYY-MM-DD
status: proposed | accepted | deprecated | superseded
superseded-by: <slug>  # only if status is deprecated or superseded
---

## Context

What is the issue that this decision addresses? What forces are at play?

## Decision

What is the decision we are making? State it clearly and concisely.

## Consequences

What are the consequences of this decision? What are the trade-offs?

## Alternatives Considered

What other options were considered and why were they rejected?
```

- `NNNN` is a zero-padded 4-digit sequence number (0001, 0002, ...)
- `slug` is a kebab-case description of the decision
- `status` reflects the current state of the decision
- `superseded-by` links to the slug of the ADR that replaces this one
