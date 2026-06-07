---
description: Apply review fixes to an existing PR branch based on a pr-reviewer report
argument-hint: <task-id> <pr-number>
allowed-tools: Task, Bash
---

Invoke the `fix-executor` subagent to apply review fixes to a PR whose latest review verdict is `request-changes`.

Arguments received:
- Task ID: `$1` — expected format `<story>.<iteration>` (e.g. `4.1`, `22.3`).
- PR number: `$2` — positive integer (e.g. `42`).

Validate before delegating:
- `$1` must match the regex `^\d+\.\d+$`.
- `$2` must be a positive integer.
- If either is missing or invalid, do NOT launch the subagent. Print a one-line error showing usage: `/python:fix <task-id> <pr-number>` and stop.

If both arguments are valid:

1. Resolve the PR's head branch:
   - `gh pr view $2 --json headRefName -q .headRefName` → capture as `<branch>`.
2. Fetch the latest review report from PR comments:
   - `gh pr view $2 --json comments --jq '[.comments[] | select(.body | startswith("<!-- python-pr-reviewer-report"))] | last | .body'`
   - The marker `<!-- python-pr-reviewer-report -->` is injected by `pr-reviewer` at the top of every report it posts, so this filter only picks plugin-authored comments (ignoring unrelated human comments).
   - If the result is empty or null, stop with a one-line error: «На PR #$2 не найдено отчёта ревью. Сначала запустите `/python:review $1 $2`.»
   - Also check the fetched body contains `**Verdict:** request-changes`. If the latest report is `approve`/`comment`, stop with: «Последнее ревью PR #$2 — не request-changes, править нечего.»
3. Launch the `fix-executor` subagent via the Task tool with this prompt (substitute values, keep the wording otherwise):

```
Task ID: $1
PR number: $2
Branch: <branch>
Attempt: 1/1

Review report:
<the review report body verbatim>

Address every non-OK finding in the report per your instructions, run build and tests, push to the same branch. End your message with the RESULT block.
```

Return the subagent's message to the user as-is.
