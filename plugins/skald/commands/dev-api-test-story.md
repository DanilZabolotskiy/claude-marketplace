---
description: Сгенерировать black-box тесты для одной истории или для батча историй из task.md. Для каждой истории сначала запускается story-test-orchestrator-подагент (чистый контекст) — он делает только discovery: ищет HTTP-маршруты, кросс-чекает OpenAPI/контроллеры/доки и возвращает по одному dev-test brief'у на эндпоинт. Затем сама команда (в главной сессии) на каждый эндпоинт гоняет fix-loop dev-test-author → pr-reviewer (auto-merge) → dev-test-fixer → автор-ретрай / сборка+деплой dev-контура скриптами из script/, до 3 фикс-попыток. Истории, которые ещё не завершены в iteration.md, пропускаются.
argument-hint: [<story-id>] [+]
allowed-tools: Task, Read, Bash, Grep, Glob
---

Generate dev-tests for one or more stories from `task.md`. Architecture splits work between a subagent (clean context per story) and the command itself (main session):

- **`story-test-orchestrator` subagent** — discovery only: routes, OpenAPI, controllers, docs, brief assembly. Emits ROUTE-BRIEF / BLOCKED-ROUTE blocks + STORY-DISCOVERY-RESULT. Does NOT dispatch other agents.
- **This command (main session)** — for each ROUTE-BRIEF the orchestrator returned, runs the per-route state machine inline, dispatching `dev-test-author`, `pr-reviewer`, `dev-test-fixer` via the Task tool. The state machine, the safety caps, and the per-story coverage table all live here. This is the reason the orchestrator-side `Task`-dispatch architecture was retired — sub-subagent dispatching is unreliable; the main session has Task working.

Only **completed** stories (all iterations marked `- [x]` in `iteration.md`) are dispatched — testing in-progress code is wasted effort.

Reply to the user in Russian. Do not enter plan mode — execute.

## Arguments

The command accepts zero, one, or two positional arguments:

- **0 args** — `/skald:dev-api-test-story` — process **every** story in `task.md` in document order.
- **1 arg** — `/skald:dev-api-test-story <N>` — process **only** story `<N>`. `<N>` must match `^\d+$`.
- **2 args** — `/skald:dev-api-test-story <N> +` — process every story in `task.md` in document order **starting from story `<N>`** (inclusive). `<N>` must match `^\d+$`, the second arg must be literally `+`.

If the arguments don't fit any of the three forms, print one Russian line and stop:

```
Использование: /skald:dev-api-test-story [<story-id>] [+]
```

## Step 0 — Sanity check working tree

```
git checkout dev && git pull --ff-only
```

- Dirty working copy → stop with «Грязная рабочая копия — закоммить или спрячь изменения и запусти команду снова.» Do not stash.
- Pull conflict / non-fast-forward → stop with the git error and ask the user to resolve.

Each per-story orchestrator will also do its own pull when it starts, so it picks up commits merged during the batch.

## Step 1 — Discover stories (document order)

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
3. For each story, capture the body between its H3 and the next H3 (or EOF). Look for at least one iteration heading:
   - `#### Iteration <N>.<M>:`, `#### Итерация <N>.<M>:`, `### Iteration <N>.<M>`, `### Итерация <N>.<M>`.
   Stories with zero iterations are skipped silently (nothing to discover routes from).
4. Build the **target list** from this ordered set of stories-with-iterations. In every form the list is a contiguous slice of the document-ordered list from Step 2 — never a numeric-range filter:
   - **0 args:** all of them, in document order.
   - **1 arg `<N>`:** the single entry for `<N>`. If not present, stop with «История `<N>` не найдена в `task.md` или не имеет итераций.»
   - **2 args `<N> +`:** the **document-order** suffix starting from the position of `<N>` in the Step-2 list, inclusive. Find the index of `<N>` and take that element plus everything after it in document order — do NOT take «every story whose id ≥ N». In the `[7, 5, 10]` example, `/skald:dev-api-test-story 7 +` dispatches `[7, 5, 10]`, and `/skald:dev-api-test-story 5 +` dispatches `[5, 10]`. If `<N>` not present, stop with the same line as above.
5. If the target list is empty (0-arg mode and no stories with iterations exist) → «В `task.md` нет историй с итерациями — запускать нечего.» stop.
6. Print the target list to the user as a numbered list (`1. <id> — <title>`) before starting — the numbering is the dispatch index (1, 2, 3…), the `<id>` is the story id, and the rows must appear in document order. No confirmation prompt — Auto mode.

## Step 1b — Filter by `iteration.md` (only test completed stories)

We only run dev-tests against stories whose code has already been merged on `dev` — testing in-progress iterations would just measure the implementation gap, not real bugs.

1. Read `iteration.md` at the project root.
   - If `iteration.md` is missing → stop with «`iteration.md` не найден — не понимаю, какие истории завершены. Запусти `/skald:dev-story` или создай файл, чтобы было откуда читать статус.»
2. For each story in the target list, look up every one of its iteration IDs (collected in Step 1) in `iteration.md`:
   - For each `<story>.<N>`, search for a line that contains the bold id (e.g. `**22.1**`) AND a checkbox.
   - `- [x]` → iteration is **done**.
   - `- [ ]` → iteration is **not done**.
   - Line not found → iteration is **not done**.
3. A story is **eligible for testing** ⇔ every one of its iterations is `- [x]`. Otherwise mark it as `not-ready`.
4. **Not-ready stories are skipped** — do NOT dispatch the orchestrator. Record them in the final report as `⏭ не готово к тестам` with the list of unchecked iteration IDs in the notes column.
5. If after filtering the dispatch list is empty (zero eligible stories), print one Russian line and stop:

   ```
   Ни одна история не готова к тестам (все имеют незавершённые итерации в iteration.md).
   ```

   Skip Step 2 entirely. Go straight to Step 3 to print the not-ready summary.

6. Print the filtered dispatch list (eligible stories only) to the user as a numbered list before starting Step 2. Note separately how many were filtered out.

## Step 2 — Per-story pipeline

Run stories **sequentially** and **in the order of the filtered dispatch list from Step 1b** — that list is already in document order (see the Critical ordering rule in Step 1). Do not re-sort by id at this stage. No parallelism — `dev` is shared, `task.md` is shared, each story's merges must be visible to the next story's discovery.

For each target story `<id>` in the filtered dispatch list (document order), do Steps 2a → 2d in order. Between stories, before dispatching the next orchestrator, re-sync the main session's working tree:

```
git checkout dev && git pull --ff-only
```

so the next orchestrator sees PRs merged by the previous story.

### Step 2a — Discovery (subagent, clean context)

Launch the `story-test-orchestrator` subagent via the Task tool with this exact prompt (substitute `<id>`):

```
Story ID: <id>

Run discovery for story <id> per your instructions: sync to dev, parse the iteration bodies in task.md, extract HTTP routes, cross-check contracts/openapi.yaml, locate controllers under modules/, gather doc context, build a self-contained dev-test brief per ok route. Emit one ===ROUTE-BRIEF #N=== block per coverable route, one ===BLOCKED-ROUTE #N=== block per non-coverable route, and end your message with the ===STORY-DISCOVERY-RESULT=== block. Do not dispatch any other subagents — the command does that.
```

Parse the orchestrator's reply:

1. **`===STORY-DISCOVERY-RESULT===` block.** Capture `status`, `story_title`, `ok_count`, `blocked_count`, `no_endpoints`, `notes`.
   - `status: blocked` → record the story as `🚫 discovery blocked` with the orchestrator's `notes`. Skip Steps 2b–2c entirely; go to the next story.
   - `status: success` + `no_endpoints: true` → record the story as `⏭ нет эндпоинтов`. Skip Steps 2b–2c; go to the next story.
   - `status: success` + `no_endpoints: false` → continue to Step 2b.

2. **All `===ROUTE-BRIEF #N===` blocks.** For each block, capture: `method`, `path`, `route_slug`, `from_iteration`, `controller_file`, `controller_class`, and the verbatim Markdown between `---BRIEF-MARKDOWN---` and `---END-BRIEF-MARKDOWN---`. Preserve emission order — that is the source order the state machine will use.

3. **All `===BLOCKED-ROUTE #N===` blocks.** For each, capture `method`, `path`, `reason`. These don't run the state machine; they go straight into the per-story discovery-blocked table in Step 2d.

4. **Pre-flight summary.** The orchestrator already printed a Russian pre-flight ("К покрытию (…) / Заблокировано (…)"). Relay it verbatim to the user before starting Step 2b so they see what's about to be tested.

### Step 2b — Per-route state machine

For each ROUTE-BRIEF in source order, maintain a per-route record:

- `attempts_used` (0..3) — number of `dev-test-fixer` dispatches so far for this route
- `author_attempt` (1..3) — counter passed to `dev-test-author` as `Attempt: <n>/3`; starts at 1, increments after every fixer dispatch
- `last_author_result` — verbatim `===RESULT===` block from the most recent `dev-test-author` run (used by fixer mode A)
- `last_review_report` — verbatim Markdown report (the part above `===RESULT===`) from the most recent `pr-reviewer` run (used by fixer mode B)
- `pr_number` — test PR number, set as soon as the author opens it
- `prod_pr_number` — prod-fix PR number, set when fixer returns `fix_kind: prod`
- `final_status` ∈ {`green`, `partial`, `failed`, `blocked`}
- `notes` — one-line summary used in the per-story coverage table

State machine — `state` starts at `AUTHOR`:

```
loop:
  if state == AUTHOR:
    result := dispatch dev-test-author (Attempt: <author_attempt>/3, brief)
    record last_author_result, pr_number (if result.pr_number is non-null)

    if result.status == success:
       state := REVIEW
    elif result.status == skipped:
       final_status := green; notes := "tests already exist on dev"; break
    elif result.status == blocked AND result.failure_kind == prod:
       if attempts_used >= 3: final_status := failed; notes := "fix loop exhausted: " + result.notes; break
       state := FIX_FROM_FAIL
    elif result.status == failed AND result.failure_kind == dev:
       if attempts_used >= 3: final_status := failed; notes := "fix loop exhausted: " + result.notes; break
       state := FIX_FROM_FAIL
    elif result.status == blocked AND result.failure_kind == null:
       # dirty tree, branch missing, etc.
       if result.notes contains "dirty working tree":
          ABORT WHOLE STORY (see Story-level abort below)
       final_status := blocked; notes := result.notes; break
    else:
       final_status := failed; notes := result.notes; break

  elif state == REVIEW:
    review := dispatch pr-reviewer (Auto-merge on approve: true, Review scope: dev-test, PR: <pr_number>)
    record last_review_report (everything in the subagent's reply before the ===RESULT=== line, starting from the <!-- skald-pr-reviewer-report --> marker)

    if review.status != success:
       final_status := failed; notes := "reviewer failed: " + review.notes; break

    if review.verdict in {approve, comment}:
       if review.merged == true:
          final_status := green; notes := ""; break
       else:
          final_status := partial; notes := "merge blocked: " + review.notes; break
    elif review.verdict == request-changes:
       if attempts_used >= 3: final_status := failed; notes := "3 review rounds without approval"; break
       state := FIX_FROM_REVIEW

  elif state == FIX_FROM_FAIL:
    attempts_used += 1
    fix := dispatch dev-test-fixer (Attempt: <attempts_used>/3, mode A: LAST FAILURE = last_author_result)
    if fix.status != success:
       final_status := failed; notes := "fixer (fail) blocked at attempt " + attempts_used + ": " + fix.notes; break
    if fix.fix_kind == dev:
       author_attempt += 1
       state := AUTHOR
    elif fix.fix_kind == prod:
       prod_pr_number := fix.prod_pr_number
       state := PROD_PR_REVIEW

  elif state == FIX_FROM_REVIEW:
    attempts_used += 1
    fix := dispatch dev-test-fixer (Attempt: <attempts_used>/3, mode B: REVIEW REPORT = last_review_report)
    if fix.status != success:
       final_status := failed; notes := "fixer (review) blocked at attempt " + attempts_used + ": " + fix.notes; break
    if fix.fix_kind == dev:
       author_attempt += 1
       state := AUTHOR
    elif fix.fix_kind == prod:
       prod_pr_number := fix.prod_pr_number
       state := PROD_PR_REVIEW

  elif state == PROD_PR_REVIEW:
    review := dispatch pr-reviewer (Auto-merge on approve: true, Review scope: prod-fix, PR: <prod_pr_number>)
    if review.status != success or review.verdict == request-changes:
       final_status := blocked
       notes := "prod-fix PR #" + prod_pr_number + " not approved (verdict: " + review.verdict + ")"
       break
    if review.verdict in {approve, comment} AND review.merged == true:
       state := BUILD_DEPLOY
    else:
       # approved but merge blocked
       final_status := blocked
       notes := "prod-fix PR #" + prod_pr_number + " approved but auto-merge blocked: " + review.notes
       break

  elif state == BUILD_DEPLOY:
    deploy_ok := build & deploy to dev via scripts (see "Build & deploy" below)
    if not deploy_ok:
       final_status := failed
       notes := "dev build/deploy failed after prod-fix #" + prod_pr_number
       break
    author_attempt += 1
    state := AUTHOR
```

Record the per-route outcome and move to the next route.

#### Build & deploy (after merging a prod-fix PR)

After `gh pr merge` of a `fix/<slug>-prod` PR succeeds (the `pr-reviewer` did this with auto-merge), the dev contour must be rebuilt and redeployed before re-running the test. There is **no CI on GitHub** — build and deploy are driven by scripts in `script/` at the project root. From the main session:

1. Sync the working tree so the build includes the just-merged fix:

   ```
   git checkout dev && git pull --ff-only
   ```

2. Check that both `script/build-push.sh` and `script/pull-up.sh` exist at the project root. If either is missing, this is a hard error — print one Russian line:

   ```
   Скрипты сборки/деплоя не найдены — ожидаю script/build-push.sh и script/pull-up.sh в корне проекта.
   ```

   Mark this route `failed` with `notes: deploy scripts missing`, mark every remaining ROUTE-BRIEF of this story as `blocked` with `notes: skipped — deploy scripts missing`, do NOT dispatch any remaining stories, and jump to Step 3 (final output).

3. Build the image and push it to the registry (Bash `timeout` parameter: `1200000` ms — 20 minutes):

   ```
   ./script/build-push.sh
   ```

4. Deploy the fresh image to the dev contour (Bash `timeout` parameter: `600000` ms — 10 minutes):

   ```
   ./script/pull-up.sh dev
   ```

- Both scripts exit 0 → deploy succeeded (`deploy_ok: true`), proceed to AUTHOR retry.
- Either script exits non-zero or times out → deploy failed (`deploy_ok: false`) → `final_status: failed` for this route; put the failing script name and the tail of its output into `notes`.

#### Story-level abort

If `dev-test-author` returns `blocked` with `notes` containing `dirty working tree`, stop the whole story loop. Mark the currently-processing route as `blocked` with `notes: dirty tree during dispatch`, and mark every remaining (not-yet-processed) ROUTE-BRIEF as `blocked` with `notes: skipped — dirty tree on earlier route`. Then jump to Step 2c for this story.

#### Safety cap

For each route, total subagent dispatches are bounded: 1 initial author + up to 3 fixer + up to 3 author retries + up to 4 reviewer (one per code state) + up to 1 prod-PR reviewer + up to 1 build+deploy run. The `attempts_used` counter caps fixer entries at 3; once exhausted, the route ends as `failed`.

Routes within a story are processed **sequentially** — never parallelise.

#### Dispatch templates

All subagent prompts must be self-contained — each subagent starts in clean context. Use these exact templates (substitute placeholders verbatim):

**`dev-test-author` (`subagent_type: dev-test-author`):**

```
Story ID: <story-id>
Route: <METHOD> <path>
Route slug: <route-slug>
Attempt: <author_attempt>/3

Implement (or, for Attempt > 1, verify) the dev-test class for the single endpoint described in the brief below. Branch `feature/devtest-<route-slug>`. Place tests under `app/src/devTest/kotlin/so/skald/app/devtest/<route-slug>/<Feature>DevApiTest.kt`. Run via `./script/dev-test.sh --tests "..."`. Open the PR into `dev` (do NOT merge — pr-reviewer will). End your message with the RESULT block.

=== BRIEF ===
<brief markdown captured verbatim between ---BRIEF-MARKDOWN--- fences in Step 2a>
=== END BRIEF ===
```

**`dev-test-fixer` mode A — after author failure (`subagent_type: dev-test-fixer`):**

```
Story ID: <story-id>
Route: <METHOD> <path>
Route slug: <route-slug>
Branch: feature/devtest-<route-slug>
Attempt: <attempts_used>/3

Apply the minimum fix needed for this endpoint's dev-test pipeline based on the last author failure below. If failure_kind is dev, fix the test under app/src/devTest/**. If prod, open a prod-fix PR on fix/<route-slug>-prod. Sanity-compile, push, end with the RESULT block.

=== BRIEF ===
<brief markdown>
=== END BRIEF ===

=== LAST FAILURE ===
<verbatim ===RESULT=== block from the last dev-test-author run, including the ===END=== line>
=== END ===
```

**`dev-test-fixer` mode B — after review request-changes (`subagent_type: dev-test-fixer`):**

```
Story ID: <story-id>
Route: <METHOD> <path>
Route slug: <route-slug>
Branch: feature/devtest-<route-slug>
Attempt: <attempts_used>/3

Address every non-OK finding in the pr-reviewer report below. Categorise as fix_kind dev or prod per your instructions. Sanity-compile, push, end with the RESULT block.

=== BRIEF ===
<brief markdown>
=== END BRIEF ===

=== REVIEW REPORT ===
<verbatim Markdown report from pr-reviewer, starting with the <!-- skald-pr-reviewer-report --> marker and including the **Verdict:** line and all sections>
=== END ===
```

**`pr-reviewer` for the test PR (`subagent_type: pr-reviewer`):**

```
Task ID: <story-id>.0
PR number: <pr_number>
Auto-merge on approve: true
Review scope: dev-test

Anchor "Task fit" on the dev-test brief embedded in the PR body and on app/src/devTest/CLAUDE.md. Fetch PR #<pr_number> via gh, produce the review report per your instructions, then merge on approve. End your message with the RESULT block.
```

**`pr-reviewer` for the prod-fix PR (`subagent_type: pr-reviewer`):**

```
Task ID: <story-id>.0
PR number: <prod_pr_number>
Auto-merge on approve: true
Review scope: prod-fix

Anchor "Task fit" on the bug description in the PR body and the linked dev-test PR. Fetch PR #<prod_pr_number> via gh, produce the review report per your instructions, then merge on approve. End your message with the RESULT block.
```

### Step 2c — Per-story coverage report

After every ROUTE-BRIEF for this story has been processed (or the loop short-circuited via Story-level abort), build the per-story report in Russian:

```
# История `<id>`: dev-тесты

Story: <story_title>
Эндпоинтов в покрытии: <ok_count>. Заблокировано (discovery): <blocked_count>.

## Покрытие
| Метод+путь | PR | Прогон | Попыток | Статус | Заметки |
| --- | --- | --- | --- | --- | --- |
| POST /api/v1/auth/register | #<pr> ✅ merged | <passed>/<test_count> | 1/3 | ✅ | — |
| POST /api/v1/auth/login    | #<pr> ✅ merged | 4/4 | 2/3 | ✅ | auto-fixed после первого падения |
| PATCH /api/v1/me           | #<pr> + prod #<X> | — | 1/3 | 🚫 | prod-fix #<X> pending review |
| GET /api/v1/me/oauth       | #<pr> ⚠️ open | 3/3 | 1/3 | ⚠️ | merge blocked: branch protection |
| DELETE /api/v1/x           | — | — | 3/3 | ❌ | fix loop exhausted: <last notes> |
```

Column legend:
- **PR** — test PR number + ✅ merged / ⚠️ open. For prod-fix scenarios, append `+ prod #<X>` so the prod-PR is visible.
- **Прогон** — `<passed>/<test_count>` from the last successful author run. `—` if no author run ever succeeded.
- **Попыток** — `<attempts_used>/3`, where `attempts_used` is the number of fixer dispatches for this route.
- **Статус**:
  - ✅ — `final_status: green`.
  - ⚠️ — `final_status: partial` (tests pass, merge blocked).
  - 🚫 — `final_status: blocked` (e.g. prod-fix awaiting human review, or other principled stop).
  - ❌ — `final_status: failed` (compile error, fix loop exhausted, deploy failed).

Then the discovery-blocked table (if any):

```
## Блокировки на этапе discovery (тесты не писали)
| Метод+путь | Причина |
| --- | --- |
| POST /api/v1/auth/login | controller not found |
```

If there were no discovery blockers, replace the table with the one line «Блокировок дискавери нет.».

Append a one-line summary based on the aggregate:
- Все `ok`-маршруты ended in `green` → «Dev-тесты истории `<id>` зелёные.»
- Хотя бы один auto-fixed (`green` with `attempts_used > 1`) → «Зелёные. Авто-фикс отработал на <K> роутах.»
- Хотя бы один `partial` / `blocked` / `failed` → «Проверь блокировки и/или PR перед промоушеном на prod.»
- Discovery вернул `no_endpoints: true` или всё заблокировано на discovery → «Тесты не писали.»

Special cases (no Step 2b executed):
- `status: blocked` from discovery → omit «## Покрытие» and «## Блокировки на этапе discovery» entirely; print one line «Discovery упал: <orchestrator notes>».
- `no_endpoints: true` → omit «## Покрытие»; print discovery-blocked table only if non-empty (it won't be in this case); end with «Тесты не писали — в истории не нашлось HTTP-маршрутов.».

Cache this per-story report — it will be relayed in Step 3.

### Step 2d — Aggregate per-story counters

For each dispatched story, derive these counters for the final summary in Step 3:

- `final_status_per_story` — derived from per-route `final_status` values:
  - All routes `green` AND `ok_count > 0` → `success`.
  - `ok_count == 0` AND `no_endpoints: true` → `success` with `no_endpoints: true`.
  - At least one `green` AND at least one non-`green` → `partial`.
  - No `green` AND at least one route processed → `failed`.
  - Discovery `blocked` → `blocked`.
- `merged` = number of routes with `final_status: green` (excluding `skipped`).
- `open` = number of routes with `final_status: partial`.
- `failed` = number of routes with `final_status: failed`.
- `blocked` = number of routes with `final_status: blocked`.
- `skipped` = number of routes where author returned `skipped`.

## Step 3 — Final output

The output depends on the number of stories in the target list.

### Single story (target list size 1)

If the only target was filtered out at Step 1b as not-ready, print one Russian line and stop:

```
История `<id>` не готова к тестам — не завершены итерации: <list of unchecked iteration IDs>.
```

Otherwise relay the cached per-story report from Step 2c verbatim. Do NOT add a top-level by-story summary — the per-story report is the full answer.

### Multi-story (target list size > 1, or 0 args)

Print first the top-level summary, then per-story sections.

```
# Прогон dev-тестов по историям: итог

Всего историй в таргете: <T>. ✅ зелёные: <S>. ⚠️ частично: <P>. ❌ ошибки: <F>. 🚫 blocked: <B>. ⏭ не готово к тестам: <NR>.

| История | Статус | Покрыто/Эндпоинтов | Заметки |
| --- | --- | --- | --- |
| <id> — <title> | ✅ всё зелёное | <merged>/<ok_count> | — |
| <id> — <title> | ⚠️ частично | <merged>/<ok_count> | <f> failed, <b> blocked, <o> open |
| <id> — <title> | ❌ ошибка | 0/<ok_count> | <notes> |
| <id> — <title> | 🚫 blocked | — | <notes> |
| <id> — <title> | ⏭ нет эндпоинтов | — | в истории не нашлось HTTP-маршрутов |
| <id> — <title> | ⏭ не готово к тестам | — | не закрыты итерации: <list> |

---

## История <id> — <title>

<cached per-story report verbatim>

## История <id> — <title>

<cached per-story report verbatim>

…
```

- Map per-story `final_status` to the table:
  - `success` + `no_endpoints: true` → `⏭ нет эндпоинтов`
  - `success` + `no_endpoints: false` → `✅ всё зелёное`
  - `partial` → `⚠️ частично`
  - `failed` → `❌ ошибка`
  - `blocked` → `🚫 blocked`
- Not-ready stories (filtered at Step 1b) appear in the table with `⏭ не готово к тестам` and the list of unchecked iteration IDs. No per-story section is printed for them (orchestrator was never dispatched).
- Always include the per-story section for every **dispatched** story, even if `no_endpoints` or `blocked` — relay the report cached in Step 2c.
- If the target list was empty after Step 1 or Step 1b, the short-circuit already printed the right line; do not print this section.

No preamble, no trailing commentary outside the structures above.
