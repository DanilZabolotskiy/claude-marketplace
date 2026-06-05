---
description: Execute one or more stories from task.md end-to-end. For each iteration runs dev-executor → pr-reviewer (auto-merge) → (fix-executor + pr-reviewer up to 3x) in the main session. Optional `tests` flag chains /skald:dev-api-test-story for each successfully implemented story before moving to the next one. No nested-subagent dispatching — the per-iteration state machine lives here.
argument-hint: [<story-id>] [+] [tests]
allowed-tools: Task, Bash, Read, Grep, Glob
---

Execute stories from `task.md` end-to-end. For each iteration of each target story the command (running in the main session) drives a state machine that dispatches `dev-executor`, `pr-reviewer`, and `fix-executor` directly via the Task tool. Merging happens through `pr-reviewer` with `Auto-merge on approve: true` — no separate merge step.

There is no `story-orchestrator` subagent any more. The previous architecture (`/skald:dev-story` → `story-orchestrator` → `dev-executor`/`pr-reviewer`/`fix-executor`) is retired because nested-subagent Task-dispatching is unreliable: the orchestrator was randomly reporting «Task tool недоступен — невозможно вызвать subagent dev-executor». The main session always has Task working, so the state machine lives here.

Reply to the user in Russian. Do not enter plan mode — execute.

## Arguments

The command accepts positional arguments in three base forms, optionally followed by the literal flag `tests`:

- **0 args** — `/skald:dev-story` — process **every** story in `task.md` in document order.
- **1 arg** — `/skald:dev-story <N>` — process **only** story `<N>`. `<N>` must match `^\d+$`.
- **2 args** — `/skald:dev-story <N> +` — process every story in `task.md` in document order **starting from story `<N>`** (inclusive). `<N>` must match `^\d+$`, the second arg must be literally `+`.

The optional `tests` flag must be the **last** token. When present, after each story finishes with every iteration green and is deployed to the dev contour (Step 2d), the command immediately runs the `/skald:dev-api-test-story` per-story pipeline for that story (Step 2e) — then moves on to the next story. Combinations:

- `/skald:dev-story tests`
- `/skald:dev-story <N> tests`
- `/skald:dev-story <N> + tests`

Parse arguments as follows:

1. If the **last** token is exactly `tests`, strip it and set `tests_after: true`. Otherwise `tests_after: false`.
2. The remaining 0/1/2 tokens must match one of the three base forms above.
3. If neither check holds, print one Russian line and stop:

```
Использование: /skald:dev-story [<story-id>] [+] [tests]
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
   - **2 args `<N> +`:** the **document-order** suffix starting from the position of `<N>` in the Step-2 list, inclusive. Find the index of `<N>` and take that element plus everything after it in document order — do NOT take «every story whose id ≥ N». In the `[7, 5, 10]` example, `/skald:dev-story 7 +` dispatches `[7, 5, 10]`, and `/skald:dev-story 5 +` dispatches `[5, 10]`. If `<N>` not present, stop with the same line as above.
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

For each target story `<id>` in the filtered dispatch list (document order), do Steps 2a → 2c in order, then Step 2d (build & deploy to dev) **if this story ended `story_final_status: success`**, then Step 2e **only if `tests_after: true`, the story ended `story_final_status: success`, and the Step 2d deploy succeeded**. Between stories, re-sync the main session's working tree (so the next story sees PRs merged by the previous one):

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
    review := dispatch pr-reviewer (Task ID: <iter_id>, PR: <pr_number>, Auto-merge: true, Review scope: iteration)
    if review.status != success:
       final_status := failed; failed_stage := review
       notes := "reviewer failed: " + review.notes
       ABORT STORY

    last_review_report := <Markdown above the ===RESULT=== line, starting from the <!-- skald-pr-reviewer-report --> marker>

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
Review scope: iteration

Read `task.md` at the project root, locate section `### Iteration <iter_id>` (or the canonical equivalent), fetch PR #<pr_number> via gh, produce the review report per your instructions, then merge on approve. End your message with the RESULT block.
```

**`fix-executor` (`subagent_type: fix-executor`):**

```
Task ID: <iter_id>
PR number: <pr_number>
Branch: <branch>
Attempt: <attempts_used>/3

Review report:
<the full last_review_report verbatim, including the <!-- skald-pr-reviewer-report --> marker and the **Verdict:** line>

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

### Step 2d — Build & deploy to the dev contour

Run **only if** this story ended `story_final_status: success` (every iteration green). For stories that ended `partial`, `failed`, or were `skipped` in Step 1b, skip this step — the dev contour keeps the previously deployed version; initialise `deploy_status := "—"` for the story record. This step runs **regardless of the `tests` flag** — the dev contour must always reflect the latest merged story.

There is **no CI on GitHub** — build and deploy are driven by scripts in `script/` at the project root:

1. Re-sync the working tree so the build includes this story's freshly merged PRs:

   ```
   git checkout dev && git pull --ff-only
   ```

   If this fails, stop the whole batch with the git error (same global-blocker rule as elsewhere).

2. Check that both `script/build-push.sh` and `script/pull-up.sh` exist at the project root. If either is missing, this is a hard error — print one Russian line:

   ```
   Скрипты сборки/деплоя не найдены — ожидаю script/build-push.sh и script/pull-up.sh в корне проекта.
   ```

   Set `deploy_status := "❌ скрипты не найдены"` for this story, do NOT dispatch any remaining stories, and jump to Step 3 (final output).

3. Build the image and push it to the registry (Bash `timeout` parameter: `1200000` ms — 20 minutes):

   ```
   ./script/build-push.sh
   ```

4. Deploy the fresh image to the dev contour (Bash `timeout` parameter: `600000` ms — 10 minutes):

   ```
   ./script/pull-up.sh dev
   ```

Outcomes:

- Both scripts exit 0 → `deploy_status := "✅"`. Continue to Step 2e (when `tests_after: true`) or to the next story.
- Either script exits non-zero or times out → `deploy_status := "❌"`, record the failing script name and the tail of its output on the story record. Skip Step 2e for this story (testing a stale contour is pointless) and stop the whole batch — the next story's deploy would fail the same way. Jump to Step 3.

### Step 2e — Per-story dev-tests (only with `tests`)

Run **only if all three**:

- the user passed the `tests` flag (`tests_after: true`), AND
- this story ended `story_final_status: success` (every iteration green), AND
- the Step 2d deploy succeeded (`deploy_status == "✅"`).

For stories that ended `partial`, `failed`, were `skipped` in Step 1b, or whose deploy failed, skip this step entirely — initialise `tests_report := null` and `tests_status := "—"` for the story record, and move on. (Stories skipped as already-done were presumably tested in their original implementing run; the user can always invoke `/skald:dev-api-test-story <id>` manually.)

Procedure:

1. The working tree is already synced and the story is deployed to the dev contour (Step 2d) — no extra git or deploy work is needed here.

2. Follow `commands/dev-api-test-story.md` **Steps 2a → 2d** for story `<id>` as the sole target. Reuse the exact subagents, prompts, state machine, and safety caps defined there — do not re-derive them here. Specifically:
   - Step 2a — dispatch `story-test-orchestrator` and parse ROUTE-BRIEF / BLOCKED-ROUTE / STORY-DISCOVERY-RESULT blocks.
   - Step 2b — per-route state machine (author → reviewer → fixer → optional prod-PR review → build+deploy via `script/`).
   - Step 2c — build the per-story dev-test coverage report.
   - Step 2d — derive per-story dev-test counters (`final_status_per_story`, `merged`, `open`, `failed`, `blocked`, `skipped`, `ok_count`).

3. Cache the per-story dev-test coverage report (from dev-api-test-story Step 2c) on the story record as `tests_report`. Cache the per-story dev-test final status as `tests_status` mapped to a single emoji+label for the multi-story summary table:
   - `success` + `no_endpoints: true` → `⏭ нет эндпоинтов`
   - `success` + `no_endpoints: false` → `✅ зелёные`
   - `partial` → `⚠️ частично`
   - `failed` → `❌ ошибка`
   - `blocked` → `🚫 blocked`

4. Skip dev-api-test-story's own Step 1b (the `iteration.md` eligibility filter): we just merged every iteration of this story and `dev-executor` flips each checkbox to `- [x]` on success, so the story is eligible by construction. Do not re-evaluate.

5. Skip dev-api-test-story's Step 0 (working tree sanity check): the outer `dev-story` already owns the dev branch and the tree was synced in Step 2d.

6. A failure inside the dev-test phase does NOT abort the outer story batch. Record `tests_status` for this story and continue to the next story per the outer loop. The exception is the same global-blocker the outer loop already handles — if `git checkout dev && git pull --ff-only` itself fails, stop the whole batch with the git error.

## Step 3 — Final output

The output depends on the number of stories actually dispatched (excluding `skipped`).

### Single dispatched story (target list size 1, not skipped)

Relay the cached per-story report from Step 2b verbatim. Do NOT add a top-level by-story summary — the per-story report is the full answer.

If the story ended `story_final_status: success` (Step 2d ran or was attempted), append one line right after the report: «Деплой на dev: <deploy_status>» — `✅`, `❌` + имя упавшего скрипта, or `❌ скрипты не найдены`.

If `tests_after: true` AND `tests_report` is non-null (i.e. Step 2e ran), append a horizontal rule and a level-2 heading `## Dev-тесты` followed by the cached `tests_report` verbatim:

```
<cached per-story impl report from Step 2b>

Деплой на dev: <deploy_status>

---

## Dev-тесты

<cached tests_report from Step 2e, verbatim>
```

If `tests_after: true` but Step 2e was skipped (story did not end `success`, or the deploy failed), do not print a dev-tests section — the impl report plus the deploy line already explain why the story didn't reach the test phase.

If the only target was skipped via Step 1b done-detection, print one Russian line: «История `<id>` уже завершена — все итерации помечены в `iteration.md`.»

### Multi-story (target list size > 1, or 0 args, or any mix of dispatched/skipped)

Print first the top-level summary, then per-story sections.

When `tests_after: true`, the summary table includes the extra column **Dev-тесты** (after «Заметки»). When `tests_after: false`, omit that column entirely — do not print empty placeholders.

Top-level summary template (with `tests`):

```
# Прогон историй: итог

Всего историй в таргете: <T>. ✅ замёрджено: <S>. ⚠️ частично: <P>. ❌ ошибки: <F>. ⏭ пропущено (готово): <K>.

| История | Статус | Успешно/Всего | Заметки | Dev-тесты |
| --- | --- | --- | --- | --- |
| <id> — <title> | ✅ всё замёрджено | <succeeded>/<total> | — | <tests_status or "—"> |
| <id> — <title> | ⚠️ частично замёрджено | <succeeded>/<total> | итерация <iter> — <stage>, <notes> | — |
| <id> — <title> | ❌ ошибка | 0/<total> | итерация <iter> — <stage>, <notes> | — |
| <id> — <title> | ⏭ пропущено (готово) | — | все итерации помечены в `iteration.md` | — |
```

Without `tests`, drop the `Dev-тесты` column:

```
| История | Статус | Успешно/Всего | Заметки |
| --- | --- | --- | --- |
…
```

Then the per-story sections:

```
---

## История <id> — <title>

<cached per-story impl report from Step 2b verbatim>

<if tests_after AND tests_report is non-null:>
---

### Dev-тесты

<cached tests_report from Step 2e verbatim>
</if>

## История <id> — <title>

…
```

- Map per-story `story_final_status` to the «Статус» column:
  - `success` → `✅ всё замёрджено`
  - `partial` → `⚠️ частично замёрджено`
  - `failed` → `❌ ошибка`
- Map `tests_status` (set in Step 2e) to the «Dev-тесты» column verbatim. For stories where Step 2e was skipped (no `tests`, story not in `success`, deploy failed, or done-detected skip), use `—`.
- If a story's Step 2d deploy failed (`deploy_status` starts with `❌`), append «деплой dev: <deploy_status>» to its «Заметки» cell, and print one line right after the table: «Остановка батча: деплой на dev упал после истории `<id>` — оставшиеся истории не запускались.»
- Stories never dispatched because the batch stopped on a deploy failure appear in the table with `⏭ не запущено` and «остановка после деплой-ошибки истории `<id>`» in «Заметки».
- Skipped (done-detected) stories appear with `⏭ пропущено (готово)`. No per-story section is printed for them.
- Always include the per-story section for every **dispatched** story, even on failure — relay the report cached in Step 2b, followed by the deploy line «Деплой на dev: <deploy_status>» when the story ended `success`. Append the cached `tests_report` (Step 2e) under a level-3 `### Dev-тесты` subheading only when it exists.
- If the target list was empty after Step 1 or every story was done-detected, the short-circuit already printed the right line; do not print this section.

No preamble, no trailing commentary outside the structures above.
