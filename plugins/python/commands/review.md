---
description: Review a GitHub PR against a specific task from task.md
argument-hint: <task-id> <pr-number>
allowed-tools: Task
---

Invoke the `pr-reviewer` subagent to review a PR against one task from `task.md`.

Arguments received:
- Task ID: `$1` — expected format `<story>.<iteration>` (e.g. `4.1`, `17.5`).
- PR number: `$2` — positive integer (e.g. `42`).

Validate before delegating:
- `$1` must match the regex `^\d+\.\d+$`.
- `$2` must be a positive integer.
- If either is missing or invalid, do NOT launch the subagent. Print a one-line error showing usage: `/python:review <task-id> <pr-number>` and stop.

If both arguments are valid, launch the `pr-reviewer` subagent via the Task tool with this prompt (substitute the values, keep the text otherwise verbatim):

```
Task ID: $1
PR number: $2

Read `task.md` at the project root, locate section `### Iteration $1`, fetch PR #$2 via gh, and produce the review report per your instructions.
```

Return the subagent's report to the user as-is.
