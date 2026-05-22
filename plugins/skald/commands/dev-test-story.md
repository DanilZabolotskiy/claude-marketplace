---
description: Сгенерировать black-box тесты для одной истории или для батча историй из task.md — каждая история уходит в свежий story-test-orchestrator (чистый контекст), внутри он по одному дёргает dev-test-author на каждый эндпоинт
argument-hint: [<story-id>] [+]
allowed-tools: Task, Read, Bash, Grep, Glob
---

Generate dev-tests for one or more stories from `task.md`. Each story is dispatched to a fresh `story-test-orchestrator` subagent — so every story starts in clean context, as if `/skald:dev-test-story` had been launched from scratch for it. The orchestrator does its own discovery (routes, OpenAPI, controllers, docs) and dispatches one `dev-test-author` per endpoint.

Reply to the user in Russian. Do not enter plan mode — execute.

## Arguments

The command accepts zero, one, or two positional arguments:

- **0 args** — `/skald:dev-test-story` — process **every** story in `task.md` in document order.
- **1 arg** — `/skald:dev-test-story <N>` — process **only** story `<N>`. `<N>` must match `^\d+$`.
- **2 args** — `/skald:dev-test-story <N> +` — process every story in `task.md` in document order **starting from story `<N>`** (inclusive). `<N>` must match `^\d+$`, the second arg must be literally `+`.

If the arguments don't fit any of the three forms, print one Russian line and stop:

```
Использование: /skald:dev-test-story [<story-id>] [+]
```

## Step 0 — Sanity check working tree

```
git checkout dev && git pull --ff-only
```

- Dirty working copy → stop with «Грязная рабочая копия — закоммить или спрячь изменения и запусти команду снова.» Do not stash.
- Pull conflict / non-fast-forward → stop with the git error and ask the user to resolve.

Each per-story orchestrator will also do its own pull when it starts, so it picks up commits merged during the batch.

## Step 1 — Discover stories (document order)

1. Read `task.md` at the project root. If missing → «`task.md` не найден.» stop.
2. Walk the file top-to-bottom and collect every H3 story heading **in the order it appears in the file**, NOT sorted by id. Accept either form:
   - `### Story <N>: <title>`
   - `### История <N>: <title>`
3. For each story, capture the body between its H3 and the next H3 (or EOF). Look for at least one iteration heading:
   - `#### Iteration <N>.<M>:`, `#### Итерация <N>.<M>:`, `### Iteration <N>.<M>`, `### Итерация <N>.<M>`.
   Stories with zero iterations are skipped silently (nothing to discover routes from).
4. Build the **target list** from this ordered set of stories-with-iterations:
   - **0 args:** all of them, in document order.
   - **1 arg `<N>`:** the single entry for `<N>`. If not present, stop with «История `<N>` не найдена в `task.md` или не имеет итераций.»
   - **2 args `<N> +`:** the suffix starting from `<N>` inclusive. If `<N>` not present, stop with the same line as above.
5. If the target list is empty (0-arg mode and no stories with iterations exist) → «В `task.md` нет историй с итерациями — запускать нечего.» stop.
6. Print the target list to the user as a numbered list (`1. <id> — <title>`) before starting. No confirmation prompt — Auto mode.

## Step 2 — Per-story dispatch

Run orchestrators **sequentially**. No parallelism — `dev` is shared, `task.md` is shared, each orchestrator's authors merge PRs that the next orchestrator must see.

For each target story `<id>` in the list:

1. Launch the `story-test-orchestrator` subagent via the Task tool with this exact prompt (substitute `<id>`):

```
Story ID: <id>

Generate dev-tests for story <id> from task.md per your instructions. Sync to dev, discover routes via task.md + OpenAPI + controllers + docs, dispatch one fresh dev-test-author per endpoint sequentially, then print the per-story report and end your message with the STORY-TEST-RESULT block.
```

2. Parse the `===STORY-TEST-RESULT===` block. Capture: `status`, `ok_count`, `merged`, `open`, `failed`, `blocked`, `skipped`, `no_endpoints`, `notes`.
3. Capture the orchestrator's per-story report (the part above STORY-TEST-RESULT) verbatim — it will be relayed in the final output for multi-story mode.
4. **Failure handling:** regardless of `status` (`success` / `partial` / `failed` / `blocked`), **always move to the next story**. Never abort the batch because one story stumbled. The orchestrator stops its own loop on internal failure; this command only orchestrates between stories.

Safety cap: at most one dispatch per story in the target list. No retries.

## Step 3 — Final output

The output depends on the number of stories in the target list:

### Single story (target list size 1)

Relay the orchestrator's per-story report verbatim. Do NOT add a top-level by-story summary — the orchestrator's report is the full answer.

### Multi-story (target list size > 1, or 0 args)

Print first the top-level summary, then per-story sections.

```
# Прогон dev-тестов по историям: итог

Всего историй: <T>. ✅ зелёные: <S>. ⚠️ частично: <P>. ❌ ошибки: <F>. 🚫 blocked: <B>.

| История | Статус | Покрыто/Эндпоинтов | Заметки |
| --- | --- | --- | --- |
| <id> — <title> | ✅ всё зелёное | <merged>/<ok_count> | — |
| <id> — <title> | ⚠️ частично | <merged>/<ok_count> | <f> failed, <b> blocked, <o> open |
| <id> — <title> | ❌ ошибка | 0/<ok_count> | <notes> |
| <id> — <title> | 🚫 blocked | — | <notes> |
| <id> — <title> | ⏭ нет эндпоинтов | — | в истории не нашлось HTTP-маршрутов |

---

## История <id> — <title>

<orchestrator's per-story report verbatim>

## История <id> — <title>

<orchestrator's per-story report verbatim>

…
```

- Map orchestrator statuses to the table:
  - `success` + `no_endpoints: true` → `⏭ нет эндпоинтов`
  - `success` + `no_endpoints: false` → `✅ всё зелёное`
  - `partial` → `⚠️ частично`
  - `failed` → `❌ ошибка`
  - `blocked` → `🚫 blocked`
- Always include the per-story section for every dispatched story, even if `no_endpoints` or `blocked` — relay whatever the orchestrator printed.
- If the target list was empty, the Step 1 short-circuit already printed the right line; do not print this section.

No preamble, no trailing commentary outside the structures above.
