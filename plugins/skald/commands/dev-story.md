---
description: Execute one or more stories from task.md end-to-end via story-orchestrator subagents (each story in clean context)
argument-hint: [<story-id>] [+]
allowed-tools: Task, Bash, Read, Grep
---

Execute stories from `task.md` end-to-end. Each story is dispatched to a fresh `story-orchestrator` subagent, so every story starts in clean context — as if `/skald:dev-story` had been launched from scratch for it. Reply to the user in Russian.

## Arguments

The command accepts zero, one, or two positional arguments:

- **0 args** — `/skald:dev-story` — process **every** story in `task.md` in document order.
- **1 arg** — `/skald:dev-story <N>` — process **only** story `<N>`. `<N>` must match `^\d+$`.
- **2 args** — `/skald:dev-story <N> +` — process every story in `task.md` in document order **starting from story `<N>`** (inclusive). `<N>` must match `^\d+$`, the second arg must be literally `+`.

If the arguments don't fit any of the three forms, print one Russian line and stop:

```
Использование: /skald:dev-story [<story-id>] [+]
```

## Discover stories (document order)

1. Read `task.md` at the project root. If missing, stop with one Russian line: «`task.md` не найден.»
2. Walk the file top-to-bottom and collect every H3 story heading **in the order it appears in the file**, NOT sorted by id. Accept either form:
   - `### Story <N>: <title>`
   - `### История <N>: <title>`
3. For each story, capture the body between its H3 and the next H3 (or EOF). Look for at least one H4 iteration heading of the form `#### Iteration <N>.<M>:` or `#### Итерация <N>.<M>:`. Stories with zero iterations are skipped silently (they were not planned yet).
4. Build the **target list** from this ordered set of stories-with-iterations:
   - **0 args:** all of them, in document order.
   - **1 arg `<N>`:** the single entry for `<N>`. If not present, stop with «История `<N>` не найдена в `task.md` или не имеет итераций.»
   - **2 args `<N> +`:** the suffix starting from `<N>` inclusive. If `<N>` not present, stop with the same line as above.
5. Print the target list to the user as a numbered list (`1. <id> — <title>`) before starting. No confirmation prompt — Auto mode.

## Done-detection (skip already-finished stories)

Before dispatching a story, check whether it is already done:

1. If `iteration.md` does not exist at the project root, the story is **not done** (no marker to consult). Proceed to dispatch.
2. Otherwise, for every H4 iteration of this story collected during discovery (e.g. `22.1`, `22.2`, …), look in `iteration.md` for a line that contains the bold id (e.g. `**22.1**`) AND a checkbox:
   - `- [x]` → that iteration is done.
   - `- [ ]` → that iteration is NOT done.
   - line not found → that iteration is NOT done.
3. If **every** iteration of the story is marked `- [x]` in `iteration.md`, the story is **done**. Record it in the final report as `⏭ пропущено (уже готово)` and DO NOT launch the subagent.
4. Otherwise dispatch the subagent.

Note: a story with `+` mode reaches done-detection in document order; skipping it does NOT abort the batch — keep going to the next target.

## Per-story dispatch

Run subagents **sequentially**. No parallelism — `dev` is shared, `task.md` is shared.

For each target story `<id>` in the list:

1. If done-detection marked it as already finished, record `skipped` and continue.
2. Otherwise, launch the `story-orchestrator` subagent via the Task tool with this exact prompt (substitute `<id>`):

```
Story ID: <id>

Execute story <id> from task.md end-to-end per your instructions. For each iteration in document order: dev-executor → pr-reviewer → (fix-executor + pr-reviewer up to 3x) → gh pr merge. Print the per-iteration Russian table and end your message with the STORY-RESULT block.
```

3. Parse the `===STORY-RESULT===` block from the subagent's reply. Capture: `status`, `total`, `succeeded`, `failed_iteration`, `failed_stage`, `notes`.
4. Capture the subagent's per-iteration Russian table (the part above the STORY-RESULT block) verbatim — it will be relayed in the final output.
5. **Failure handling:** regardless of `status` (`success` / `partial` / `failed` / `blocked`), **always move to the next story**. Never abort the batch because one story stumbled. The story-orchestrator already stops the story itself on internal failure; this command only orchestrates between stories.

Safety cap: at most one dispatch per story in the target list. No retries.

## Final output

The output depends on the number of stories actually dispatched (excluding `skipped`):

### Single story (target list size 1)

Relay the orchestrator's per-iteration table verbatim. Do NOT add a top-level by-story summary — the orchestrator's own report is the full answer.

If the only target was skipped via done-detection, print one Russian line: «История `<id>` уже завершена — все итерации помечены в `iteration.md`.»

### Multi-story (target list size > 1, or 0 args)

Print first the top-level summary, then per-story sections.

```
# Прогон историй: итог

Всего историй: <T>. ✅ замёрджено: <S>. ⚠️ частично: <P>. ❌ ошибки: <F>. ⏭ пропущено (готово): <K>.

| История | Статус | Успешно/Всего | Заметки |
| --- | --- | --- | --- |
| <id> — <title> | ✅ всё замёрджено | <k>/<n> | — |
| <id> — <title> | ⚠️ частично замёрджено | <k>/<n> | итерация <iter> — <stage>, <notes> |
| <id> — <title> | ❌ ошибка | 0/<n> | итерация <iter> — <stage>, <notes> |
| <id> — <title> | 🚫 blocked | — | <notes> |
| <id> — <title> | ⏭ пропущено (готово) | — | все итерации помечены в `iteration.md` |

---

## История <id> — <title>

<orchestrator's per-iteration table verbatim>

## История <id> — <title>

<orchestrator's per-iteration table verbatim>

…
```

- Skip the per-story section for stories that were skipped via done-detection (they have no orchestrator output).
- If the target list was empty (e.g. 0-arg mode and no stories with iterations exist), print one Russian line: «В `task.md` нет историй с итерациями — запускать нечего.»

No preamble, no trailing commentary outside the structures above.
