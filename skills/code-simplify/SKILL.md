---
name: code-simplify
description: "Simplifies and refines code for clarity, consistency, and maintainability while preserving all functionality. Focuses on recently modified code unless instructed otherwise."
disable-model-invocation: false
---

# Code Simplify

Simplify and refine code for clarity, consistency, and maintainability while preserving exact functionality.

## Invocation

This skill is invoked by the issue-orchestrator after the `implement` stage completes. It receives:
- `issue` — the issue number (e.g., `42`)
- `branch` — the integration branch (e.g., `feature/issue-42`)
- `base_branch` — the base branch (e.g., `main`)

## Process

1. **Discover Changed Files** — First, discover which files differ between `base_branch` and `branch`:

   ```bash
   git diff <base_branch>...<branch> --stat
   ```

   This gives you the list of all changed files. If the orchestrator passes a specific scope, use that instead.

2. **Scope** — Only refine code that has been recently modified or touched in the current session, unless explicitly instructed to review a broader scope. If no specific scope is given, refine the entire branch's changes against `base_branch`.

3. **Preserve Functionality** — Never change what the code does — only how it does it. All original features, outputs, and behaviors must remain intact.

4. **Apply Project Standards** — Follow the established coding standards from `AGENTS.md` (or `CLAUDE.md` if present) including:
   - Use ES modules with proper import sorting and extensions
   - Prefer `function` keyword over arrow functions
   - Use explicit return type annotations for top-level functions
   - Follow proper React component patterns with explicit Props types
   - Use proper error handling patterns (avoid try/catch when possible)
   - Maintain consistent naming conventions

5. **Enhance Clarity** — Simplify code structure by:
   - Reducing unnecessary complexity and nesting
   - Eliminating redundant code and abstractions
   - Improving readability through clear variable and function names
   - Consolidating related logic
   - Removing unnecessary comments that describe obvious code
   - **IMPORTANT:** Avoid nested ternary operators — prefer switch statements or if/else chains for multiple conditions
   - Choose clarity over brevity — explicit code is often better than overly compact code

6. **Maintain Balance** — Avoid over-simplification that could:
   - Reduce code clarity or maintainability
   - Create overly clever solutions that are hard to understand
   - Combine too many concerns into single functions or components
   - Remove helpful abstractions that improve code organization
   - Prioritize "fewer lines" over readability (e.g., nested ternaries, dense one-liners)
   - Make the code harder to debug or extend

7. **Verify** — Ensure the refined code is simpler and more maintainable than the original.

8. **Document** — Document only significant changes that affect understanding.

## Output

After simplification, commit the changes to the integration branch (`branch`). Then signal `success` to the orchestrator so it transitions to `ready-for-hunt-bugs`.

## Key Principle

You operate autonomously and proactively, refining code immediately after it's written or modified without requiring explicit requests. Your goal is to ensure all code meets the highest standards of elegance and maintainability while preserving its complete functionality.
