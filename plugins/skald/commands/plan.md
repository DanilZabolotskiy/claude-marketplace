---
description: Plan iterations for stories in task.md that don't have iterations yet — one story per subagent in clean context, the planner writes back to task.md
allowed-tools: Task, Read
---

For every story in `task.md` that has no iterations yet, dispatch a fresh `iteration-planner` subagent in clean context. Each subagent reads the project, generates iterations, and writes them back to `task.md` itself. Reply to the user in Russian.

## Arguments

No arguments. If the user passed anything, ignore it and proceed.

## Discover stories

1. Read `task.md` at the project root. If it is missing, stop with one Russian line: «`task.md` не найден.» and exit.
2. Find every H3 story heading in document order:
   - `### Story <N>: <title>`
   - `### История <N>: <title>`
3. For each story, decide whether iterations already exist between its H3 and the next H3 (or EOF). An iteration is an H4 of the form:
   - `#### Iteration <N>.<M>:` or
   - `#### Итерация <N>.<M>:`
4. Build `pending` — the ordered list of story ids without iterations.
5. If `pending` is empty, print the single line «Историй без итераций нет.» and stop.
6. Print `pending` to the user (one line per story: `<id> — <title>`) and proceed without confirmation (Auto mode).

## Per-story loop

Run subagents **sequentially**. No parallelism — `task.md` is shared, and each planner reads fresh state between runs.

For each story `<id>` in `pending`, in document order:

1. Launch the `iteration-planner` subagent via the Task tool. The prompt is the following text, **verbatim** — no substitutions, no edits, no surrounding context:

```
Изучи проект. Есть список историй в task.md. История это атомарная функциональность, например какая-то фича или регистрация.  История состоит из Итераций - отдельных небольших задач. Минимум одна итерация в истории.Тебе надо изучить всю документацию и составить итерации для всех историй по порядку, записать их в task.md. Будем делать по одной истории.

Формат итерации:

#### Iteration 1.1: Iteration name
- **Goal:**
- **What we do:**
- **Definition of done:** Working Platega API integration; the port is ready to be invoked from the bot.

Правила:

1. Описывай максимально кратко, без воды, но содержательно
2. Новый функционал должен быть покрыт тестам - юнит, интеграционнными или e2e, в зависимости от функцонала. Это надо явно указывать в истории, если она предполагает тесты.
3.  В истории может быть от одной до 10 итераций
```

2. Wait for the subagent to finish. Capture its short confirmation (story id + iteration count + list) for the final report.
3. After each run, re-read `task.md` and verify story `<id>` now has at least one H4 `#### Iteration <id>.<M>:` or `#### Итерация <id>.<M>:`. If not, mark the story as `failed` and continue with the next one. Do NOT auto-retry the same story.
4. Move to the next story.

Safety cap: at most one planner run per story from the original `pending`. If `task.md` mutates unexpectedly mid-loop and `pending` shifts, trust the planner (it picks the first unplanned story itself) — do not re-plan its targeting.

## Final report

After all stories are processed (or the list is exhausted), print one Russian table:

```
| История | Статус | Итераций |
| --- | --- | --- |
| <id> — <title> | ✅ готово | <N> |
| <id> — <title> | ❌ failed | — |
```

- No preamble, no postscript.
- If every story passed, append one line: «Все истории обработаны.»
- If `pending` was empty from the start, do not print the table — the line from Discover step 5 is enough.
