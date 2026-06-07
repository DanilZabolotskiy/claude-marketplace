---
description: Execute one or more stories from task.md end-to-end. For each iteration runs dev-executor → pr-reviewer (auto-merge) → (fix-executor + pr-reviewer up to 3x) in the main session. No nested-subagent dispatching — the per-iteration state machine lives here.
argument-hint: [<story-id>] [+]
allowed-tools: Task, Bash, Read, Grep, Glob
---

Execute stories from `task.md` end-to-end. For each iteration of each target story the command (running in the main session) drives a state machine that dispatches `dev-executor`, `pr-reviewer`, and `fix-executor` directly via the Task tool. Merging happens through `pr-reviewer` with `Auto-merge on approve: true` — no separate merge step.

There is no `story-orchestrator` subagent any more. The previous architecture (`/python:dev-story` → `story-orchestrator` → `dev-executor`/`pr-reviewer`/`fix-executor`) is retired because nested-subagent Task-dispatching is unreliable: the orchestrator was randomly reporting «Task tool недоступен — невозможно вызвать subagent dev-executor». The main session always has Task working, so the state machine lives here.

Reply to the user in Russian. Do not enter plan mode — execute.

## Arguments

The command accepts zero, one, or two positional arguments:

- **0 args** — `/python:dev-story` — process **every** story in `task.md` in document order.
- **1 arg** — `/python:dev-story <N>` — process **only** story `<N>`. `<N>` must match `^\d+$`.
- **2 args** — `/python:dev-story <N> +` — process every story in `task.md` in document order **starting from story `<N>`** (inclusive). `<N>` must match `^\d+$`, the second arg must be literally `+`.

If the arguments don't fit any of the three forms, print one Russian line and stop:

```
Использование: /python:dev-story [<story-id>] [+]
```

## Step 0 — Sanity check working tree

```
git checkout dev && git pull --ff-only
```

- Dirty working copy → stop with «Грязная рабочая копия — закоммить или спрячь изменения и запусти команду снова.» Do not stash.
- Pull conflict / non-fast-forward → stop with the git error and ask the user to resolve.

## Step 1 — Discover stories and iterations (document order)

> **Critical ordering rule.** Stories are processed in the order they physically appear in `task.md`, **not** by numeric story id. Story ids do not have to be monotonic — `task.md` may legitimately list Story 7 before Story 5, and that is the order they must run in. Never sort, re-order, or compare ids numerically when building or iterating the target list. If you find yourself wanting to write `sort` or compare `<id_a> < <id_b>`, you are doing it wrong — use the index in the document-order list instead.

1. Read `task.md` at the project root. If missing → «`task.md` не найден.» stop.
2. Walk the file top-to-bottom and collect every H3 story heading **in the order it appears in the file**, NOT sorted by id. Accept either form:
   - `### Story <N>: <title>`
   - `### История <N>: <title>`

   Concrete example — if `task.md` contains headings in this physical order:
   ```
   ### Story 7: …
   ### Story 5: …
   ### Story 10: …
   ```
   the collected list is `[7, 5, 10]` and that is the dispatch order. It is NOT `[5, 7, 10]`.
3. For each story, capture the body between its H3 and the next H3 (or EOF). Scan the body for every iteration heading in document order and capture the ordered list of iteration IDs:
   - `#### Iteration <N>.<M>:`, `#### Итерация <N>.<M>:`, `### Iteration <N>.<M>`, `### Итерация <N>.<M>`.
   - De-duplicate while preserving order.
   - Stories with zero iterations are skipped silently (nothing to execute).
4. Build the **target list** from this ordered set of stories-with-iterations. In every form the list is a contiguous slice of the document-ordered list from Step 2 — never a numeric-range filter:
   - **0 args:** all of them, in document order.
   - **1 arg `<N>`:** the single entry for `<N>`. If not present, stop with «История `<N>` не найдена в `task.md` или не имеет итераций.»
   - **2 args `<N> +`:** the **document-order** suffix starting from the position of `<N>` in the Step-2 list, inclusive. Find the index of `<N>` and take that element plus everything after it in document order — do NOT take «every story whose id ≥ N». In the `[7, 5, 10]` example, `/python:dev-story 7 +` dispatches `[7, 5, 10]`, and `/python:dev-story 5 +` dispatches `[5, 10]`. If `<N>` not present, stop with the same line as above.
5. If the target list is empty (0-arg mode and no stories with iterations exist) → «В `task.md` нет историй с итерациями — запускать нечего.» stop.
6. Print the target list to the user as a numbered list (`1. <id> — <title> (итераций: <K>)`) before starting — the numbering is the dispatch index (1, 2, 3…), the `<id>` is the story id, and the rows must appear in document order. No confirmation prompt — Auto mode.

## Step 1b — Done-detection (skip already-finished stories)

Before running any story, check whether it is already done:

1. If `iteration.md` does not exist at the project root, treat every story as **not done** (no marker to consult). Proceed to dispatch.
2. Otherwise, for every iteration of this story collected in Step 1 (e.g. `22.1`, `22.2`, …), look in `iteration.md` for a line that contains the bold id (e.g. `**22.1**`) AND a checkbox:
   - `- [x]` → that iteration is done.
   - `- [ ]` → that iteration is NOT done.
   - line not found → that iteration is NOT done.
3. If **every** iteration of the story is marked `- [x]`, the story is **done**. Record it for the final report as `⏭ пропущено (уже готово)`. Do NOT execute it.
4. Otherwise mark it for dispatch.

Note: a story with `+` mode reaches done-detection in document order; skipping it does NOT abort the batch — keep going to the next target.

If after filtering the dispatch list is empty (every targeted story is already done), print one Russian line and skip Step 2:

```
Все целевые истории уже завершены — все итерации помечены в `iteration.md`.
```

Then jump to Step 3 to print the skip-summary.

Print the filtered dispatch list (eligible stories only) as a numbered list before starting Step 2.

## Step 2 — Per-story pipeline

Run stories **sequentially** and **in the order of the filtered dispatch list from Step 1b** — that list is already in document order (see the Critical ordering rule in Step 1). Do not re-sort by id at this stage. No parallelism — `dev` is shared, `task.md` is shared, each story's merges must be visible to the next story's discovery.

For each target story `<id>` in the filtered dispatch list (document order), do Steps 2a → 2c in order. Between stories, re-sync the main session's working tree (so the next story sees PRs merged by the previous one):

```
git checkout dev && git pull --ff-only
```

### Step 2a — Per-iteration state machine

Process the story's iterations in the order captured in Step 1 (`<story>.1`, `<story>.2`, …).

Maintain a per-iteration record:
- `iter_id` — e.g. `22.3`.
- `final_status` ∈ {`green`, `failed`, `not-run`}. Starts `null`; assigned exactly once.
- `failed_stage` ∈ {`dev`, `review`, `fix`, `merge`, `null`} — only set when `final_status: failed`.
- `pr_number` — set as soon as `dev-executor` opens the PR.
- `attempts_used` (0..3) — number of `fix-executor` dispatches for this iteration.
- `last_review_report` — verbatim Markdown report (the part above `===RESULT===`) from the most recent `pr-reviewer` run for this iteration; passed to `fix-executor`.
- `notes` — one short line for the per-story coverage row.

Maintain a story-level flag `story_aborted: false` — once any iteration fails, set to `true` and mark every remaining iteration as `not-run`. Do NOT continue to the next iteration in the same story after an abort.

State machine for one iteration — `state` starts at `DEV`:

```
loop:
  if state == DEV:
    result := dispatch dev-executor (Task ID: <iter_id>)
    if result.status == success:
       pr_number := result.pr_number
       branch    := result.branch
       state := REVIEW
    else:
       # blocked or failed
       final_status := failed; failed_stage := dev
       notes := result.notes
       ABORT STORY (mark remaining iterations as not-run, jump to Step 2b)

  elif state == REVIEW:
    review := dispatch pr-reviewer (Task ID: <iter_id>, PR: <pr_number>, Auto-merge: true)
    if review.status != success:
       final_status := failed; failed_stage := review
       notes := "reviewer failed: " + review.notes
       ABORT STORY

    last_review_report := <Markdown above the ===RESULT=== line, starting from the <!-- python-pr-reviewer-report --> marker>

    if review.verdict in {approve, comment}:
       if review.merged == true:
          final_status := green
          notes := (review.verdict == comment) ? "approved с комментариями" : ""
          break   # done with this iteration, go to next
       else:
          # approved but merge blocked (branch protection, required checks, etc.)
          final_status := failed; failed_stage := merge
          notes := "merge blocked: " + review.notes
          ABORT STORY
    elif review.verdict == request-changes:
       state := FIX

  elif state == FIX:
    if attempts_used >= 3:
       final_status := failed; failed_stage := fix
       notes := "3 попытки исправлений без approve. PR #" + pr_number + " остаётся открытым."
       ABORT STORY

    attempts_used += 1
    fix := dispatch fix-executor (Task ID: <iter_id>, PR: <pr_number>, Branch: <branch>, Attempt: <attempts_used>/3, review report)
    if fix.status != success:
       final_status := failed; failed_stage := fix
       notes := "fixer attempt " + attempts_used + ": " + fix.notes
       ABORT STORY

    # fix pushed — re-review
    state := REVIEW
```

#### Dispatch templates

Each subagent prompt is self-contained — they all start in clean context. Use these exact templates (substitute placeholders verbatim):

**`dev-executor` (`subagent_type: dev-executor`):**

```
Task ID: <iter_id>

Implement iteration <iter_id> from task.md per your instructions. Create the feature branch off `dev`, implement only what the iteration section describes, run build and tests, open a PR into `dev`. End your message with the RESULT block.
```

**`pr-reviewer` (`subagent_type: pr-reviewer`):**

```
Task ID: <iter_id>
PR number: <pr_number>
Auto-merge on approve: true

Read `task.md` at the project root, locate section `### Iteration <iter_id>` (or the canonical equivalent), fetch PR #<pr_number> via gh, produce the review report per your instructions, then merge on approve. End your message with the RESULT block.
```

**`fix-executor` (`subagent_type: fix-executor`):**

```
Task ID: <iter_id>
PR number: <pr_number>
Branch: <branch>
Attempt: <attempts_used>/3

Review report:
<the full last_review_report verbatim, including the <!-- python-pr-reviewer-report --> marker and the **Verdict:** line>

Address every non-OK finding in the report per your instructions, run build and tests, push to the same branch. Do NOT open a new PR. End your message with the RESULT block.
```

#### Safety cap

Per iteration: 1 `dev-executor` + up to 3 `fix-executor` + up to 4 `pr-reviewer` (one initial + one per fix attempt) = at most 8 subagent dispatches. The `attempts_used` counter caps fixer entries at 3; once exhausted, the iteration ends as `failed at fix`.

Iterations within a story are processed **sequentially** — never parallelise.

### Step 2b — Per-story coverage report

After every iteration has been processed (green) or the loop short-circuited via abort, build the per-story report in Russian:

```
# История `<id>`: итог

Всего итераций: <total>. Успешно: <merged_count>. Провалено: <failed_count>. Не запущено: <not_run_count>.

| Итерация | Статус | PR | Заметки |
| --- | --- | --- | --- |
| <iter> | ✅ approved+merged | #<pr> | <comment-verdict note or empty> |
| <iter> | ❌ failed at <stage> | #<pr or —> | <notes> |
| <iter> | ⏭ не запущено | — | — |
```

Rules:
- `<stage>` ∈ {`dev`, `review`, `fix`, `merge`}.
- If `final_status: failed` with `failed_stage: fix`, include the PR number in the PR column and the open-PR hint in the notes column.
- If `final_status: failed` with `failed_stage: dev`, the PR column is `—` (no PR was opened).
- If all iterations finished green, append one line after the table: «Все итерации истории `<id>` замёрджены в `dev`.»
- If at least one iteration is green and at least one failed: «Часть итераций замёрджена, остановились на итерации `<failed_iter>` (`<stage>`).»
- If the very first iteration failed (no greens): «История остановилась на первой же итерации `<failed_iter>` (`<stage>`).»

Cache this per-story report — it will be relayed in Step 3.

### Step 2c — Aggregate per-story counters

For each dispatched story, derive these counters for the final summary in Step 3:

- `story_final_status`:
  - All iterations `green` → `success`.
  - At least one `green` AND at least one `failed` → `partial`.
  - No `green` (first iteration failed) → `failed`.
  - (`blocked` is not a per-story state here — Step 0/1 short-circuits on infra problems.)
- `succeeded` = number of iterations with `final_status: green`.
- `total` = total number of iterations in the story.
- `failed_iteration` = id of the first iteration with `final_status: failed`, or `null`.
- `failed_stage` = `failed_stage` of that iteration, or `null`.

## Step 3 — Final output

The output depends on the number of stories actually dispatched (excluding `skipped`).

### Single dispatched story (target list size 1, not skipped)

Relay the cached per-story report from Step 2b verbatim. Do NOT add a top-level by-story summary — the per-story report is the full answer.

If the only target was skipped via Step 1b done-detection, print one Russian line: «История `<id>` уже завершена — все итерации помечены в `iteration.md`.»

### Multi-story (target list size > 1, or 0 args, or any mix of dispatched/skipped)

Print first the top-level summary, then per-story sections.

```
# Прогон историй: итог

Всего историй в таргете: <T>. ✅ замёрджено: <S>. ⚠️ частично: <P>. ❌ ошибки: <F>. ⏭ пропущено (готово): <K>.

| История | Статус | Успешно/Всего | Заметки |
| --- | --- | --- | --- |
| <id> — <title> | ✅ всё замёрджено | <succeeded>/<total> | — |
| <id> — <title> | ⚠️ частично замёрджено | <succeeded>/<total> | итерация <iter> — <stage>, <notes> |
| <id> — <title> | ❌ ошибка | 0/<total> | итерация <iter> — <stage>, <notes> |
| <id> — <title> | ⏭ пропущено (готово) | — | все итерации помечены в `iteration.md` |

---

## История <id> — <title>

<cached per-story report verbatim>

## История <id> — <title>

<cached per-story report verbatim>

…
```

- Map per-story `story_final_status` to the «Статус» column:
  - `success` → `✅ всё замёрджено`
  - `partial` → `⚠️ частично замёрджено`
  - `failed` → `❌ ошибка`
- Skipped (done-detected) stories appear with `⏭ пропущено (готово)`. No per-story section is printed for them.
- Always include the per-story section for every **dispatched** story, even on failure — relay the report cached in Step 2b.
- If the target list was empty after Step 1 or every story was done-detected, the short-circuit already printed the right line; do not print this section.

No preamble, no trailing commentary outside the structures above.
