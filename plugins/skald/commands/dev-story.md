---
description: Execute an entire story from task.md iteration by iteration — dev → review → (fix+review up to 3x) → merge — for every iteration in order
argument-hint: <story-id>
allowed-tools: Task, Bash, Read, Grep
---

Execute story `$1` from `task.md` end-to-end. For each iteration in order: implement via `dev-executor`, review via `pr-reviewer`, run a fix-and-review loop if needed (up to 3 attempts), then merge the PR and proceed to the next iteration. Reply to the user in Russian.

## Validation

- `$1` must match the regex `^\d+$` (e.g. `22`).
- If the argument is missing or invalid, print a one-line error with usage: `Usage: /skald:dev-story <story-id>` and stop without doing anything else.

## Discover iterations

1. Read `task.md` at the project root. If the file is missing, stop with a one-line error.
2. Find every heading that matches one of these forms, in the order they appear:
   - `#### Итерация $1.<N>:` (Russian H4 — used by the bundled example)
   - `### Iteration $1.<N>` (English H3)
3. Extract the ordered list of iteration IDs (`$1.1`, `$1.2`, …). De-duplicate while preserving order.
4. If the list is empty, stop and report: «В `task.md` не найдено ни одной итерации истории `$1`».
5. Print the discovered list to the user and begin execution. Do not ask for confirmation — in Auto mode just proceed.

## Per-iteration loop

Process iterations strictly in order. Maintain a running log of outcomes per iteration (status, PR number, merge result, notes) to build the final report.

For each iteration `<iter>`:

### 1. Dev phase

Launch the `dev-executor` subagent via the Task tool with this exact prompt (substitute `<iter>`):

```
Task ID: <iter>

Implement iteration <iter> from task.md per your instructions. Create the feature branch off `dev`, implement only what the iteration section describes, run build and tests, open a PR into `dev`. End your message with the RESULT block.
```

Parse the `===RESULT===` block from the subagent's reply:

- `status: blocked` → stop the whole story. Record the iteration as failed at the dev stage. Go to the final report.
- `status: failed` → stop the whole story. Record the iteration as failed at the dev stage. Go to the final report.
- `status: success` → capture `pr_number` and `branch`, continue to the review phase.

### 2. Review phase (first pass)

Launch the `pr-reviewer` subagent via the Task tool with this exact prompt (substitute values, keep the wording otherwise):

```
Task ID: <iter>
PR number: <pr_number>

Read `task.md` at the project root, locate section `### Iteration <iter>`, fetch PR #<pr_number> via gh, and produce the review report per your instructions.
```

Parse the `**Verdict:**` line from the report:

- `approve` → go to the merge phase.
- `comment` → treat as approve. Go to the merge phase (non-blocking comments will be included in the final report).
- `request-changes` → enter the fix loop.

Keep the full review report text — you will pass it verbatim to `fix-executor` if needed.

### 3. Fix loop (up to 3 attempts)

For `attempt` in `1`, `2`, `3`:

a. Launch the `fix-executor` subagent via the Task tool with this exact prompt (substitute values):

```
Task ID: <iter>
PR number: <pr_number>
Branch: <branch>
Attempt: <attempt>/3

Review report:
<the full review report verbatim, including the Verdict line>

Address every non-OK finding in the report per your instructions, run build and tests, push to the same branch. End your message with the RESULT block.
```

Parse its `===RESULT===`:

- `blocked` or `failed` → stop the whole story. Record the iteration as failed at the fix stage with the `notes` field. Go to the final report.
- `success` → re-run the review.

b. Re-launch `pr-reviewer` with the same prompt as in step 2. Parse the new `Verdict`:

- `approve` or `comment` → break out of the fix loop, go to the merge phase.
- `request-changes` → save the new report, proceed to the next attempt.

If the fix loop exits without approval after the 3rd attempt, stop the whole story with the message: «Итерация `<iter>`: 3 попытки исправлений не прошли ревью. PR #<pr_number> остаётся открытым.» Record the iteration as failed at the fix stage, include the last review report excerpt in the final report.

### 4. Merge phase

Run: `gh pr merge <pr_number> --squash --delete-branch`

- If the command succeeds, record the iteration as approved+merged and continue to the next iteration.
- If the command fails (branch protection, required checks, network), stop the whole story. Record the iteration as failed at the merge stage. Include the stderr of `gh` in `notes`. Do NOT retry, do NOT use `--admin` unless the user explicitly asked for it.

After a successful merge, the next iteration's `dev-executor` will see the merged commit on `dev` because it starts with `git checkout dev && git pull`.

## Final report (Russian)

When the loop ends — whether by completing all iterations or by stopping early — print a single Russian summary to the user:

```
# История <story-id>: итог

Всего итераций: <n>. Успешно: <k>. Провалено: <m>. Не запущено: <rest>.

| Итерация | Статус | PR | Заметки |
| --- | --- | --- | --- |
| <iter> | ✅ approved+merged | #<pr> | <if comment-verdict, brief note> |
| <iter> | ❌ failed at <stage> | #<pr или —> | <reason from notes> |
| <iter> | ⏭ не запущено | — | — |
```

- `stage` ∈ `dev`, `review`, `fix`, `merge`.
- If the failure happened in `fix`, include a one-line hint pointing at the open PR.
- If all iterations finished green, append one line: «Все итерации истории <story-id> замёрджены в `dev`.»

No preamble, no trailing commentary.
