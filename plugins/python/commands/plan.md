---
description: Plan iterations for stories in task.md that don't have iterations yet — one story per subagent in clean context, the planner writes back to task.md
argument-hint: [<story-id>]
allowed-tools: Task, Read
---

Dispatch fresh `iteration-planner` subagents in clean context to plan iterations for stories in `task.md`. Each subagent reads the project, generates iterations, and writes them back to `task.md`. Reply to the user in Russian.

## Arguments

Optional positional argument `$1` — a single story id (digits only, e.g. `3`).

- If `$1` is empty: process **every** story in `task.md` that has no iterations yet, in document order.
- If `$1` is provided: process **only** that one story.
  - Validate `$1` matches `^\d+$`. If not, print one Russian line: «Неверный формат аргумента. Использование: `/python:plan [<story-id>]`.» and stop.

## Discover stories

1. Read `task.md` at the project root. If it is missing, stop with one Russian line: «`task.md` не найден.» and exit.
2. Find every H3 story heading in document order:
   - `### Story <N>: <title>`
   - `### История <N>: <title>`
3. For each story, decide whether iterations already exist between its H3 and the next H3 (or EOF). An iteration is an H4 of the form:
   - `#### Iteration <N>.<M>:` or
   - `#### Итерация <N>.<M>:`
4. Build the target list:
   - **No argument:** `targets` = ordered list of story ids without iterations.
   - **Argument `$1`:** locate story `$1`. If it does not exist, stop with «История `$1` не найдена в `task.md`.» If it already has iterations, stop with «История `$1` уже спланирована — итерации существуют.» Otherwise `targets` = `[$1]`.
5. If `targets` is empty (no-argument mode only), print the single line «Историй без итераций нет.» and stop.
6. Print `targets` to the user (one line per story: `<id> — <title>`) and proceed without confirmation (Auto mode).

## Per-story loop

Run subagents **sequentially**. No parallelism — `task.md` is shared, and each planner reads fresh state between runs.

For each story `<id>` in `targets`, in document order:

1. Launch the `iteration-planner` subagent via the Task tool. Build the prompt by substituting `<id>` into the template below — keep the rest of the wording verbatim:

```
Целевая история: <id>

Изучи проект. Есть список историй в task.md. История это атомарная функциональность, например какая-то фича или регистрация. История состоит из Итераций - отдельных небольших задач. Минимум одна итерация в истории. Тебе надо изучить документацию и составить итерации для истории <id>, записать их в task.md. Работаем строго над одной этой историей — другие истории не трогаем.

Формат итерации:

#### Iteration <id>.1: Iteration name
- **Goal:**
- **What we do:**
- **Definition of done:** `pytest` зелёный; новая опция работает end-to-end и описана в README.

Правила:

1. Истории и итерации должны быть максимально лаконичными — без воды, без примеров кода, без «введений» и «выводов», — но при этом содержать всю информацию, необходимую исполнителю, чтобы понять, что делать и какой результат ожидается. Если фразу можно убрать без потери смысла — убирай.
2. Эталон формата и уровня детализации — История 1 из `task.md`. Перед тем как писать итерации для истории `<id>`, прочитай Историю 1 целиком (её H3 и все её H4-итерации) и подражай её стилю: длина пунктов, формулировки `Goal` / `What we do` / `Definition of done`, степень детализации, отсутствие реализационных подробностей. Любое отклонение от этого образца должно быть осознанным и оправданным самим характером истории.
3. Новый функционал должен быть покрыт тестами — юнит или интеграционными (pytest), в зависимости от функционала. Это надо явно указывать в итерации, если она предполагает тесты.
4. В истории может быть от одной до 10 итераций.
5. В итерации НЕ должно быть примеров кода, сигнатур функций/классов, схем БД, JSON/YAML, описаний полей сущностей и других внутренних подробностей реализации. Только краткое описание задачи: что нужно сделать и какой результат. Решение принимает исполнитель — не предписывай ему структуры данных и именование.
```

2. Wait for the subagent to finish. Capture its short confirmation (story id + iteration count + list) for the final report.
3. After each run, re-read `task.md` and verify story `<id>` now has at least one H4 `#### Iteration <id>.<M>:` or `#### Итерация <id>.<M>:`. If not, mark the story as `failed` and continue with the next one. Do NOT auto-retry the same story.
4. Move to the next story.

Safety cap: at most one planner run per story from the original `targets`.

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
- If `targets` was empty from the start, do not print the table — the line from Discover step 5 is enough.
