---
description: Execute an iteration from task.md in a separate feature branch and open a PR into dev
argument-hint: <task-id>
allowed-tools: Task
---

Invoke the `dev-executor` subagent to implement one iteration from `task.md` and open a PR into `dev`.

Argument received:
- Task ID: `$1` — expected format `<story>.<iteration>` (e.g. `4.1`, `17.5`).

Validate before delegating:
- `$1` must match the regex `^\d+\.\d+$`.
- If the argument is missing or invalid, do NOT launch the subagent. Print a one-line error showing usage: `/skald:dev <task-id>` and stop.

If the argument is valid, launch the `dev-executor` subagent via the Task tool with this prompt (substitute the value, keep the text otherwise verbatim):

```
Task ID: $1

Implement iteration $1 from task.md per your instructions. Create the feature branch off `dev`, implement only what the iteration section describes, run build and tests, open a PR into `dev`. End your message with the RESULT block.
```

Return the subagent's message to the user as-is.
