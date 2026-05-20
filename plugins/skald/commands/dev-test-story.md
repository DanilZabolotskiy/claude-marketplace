---
description: Сгенерировать black-box тесты для всей истории task.md — команда собирает список API и по одному делегирует агенту, каждый эндпоинт = свой PR
argument-hint: <story-id>
allowed-tools: Task, Read, Bash, Grep, Glob
---

Generate dev-tests for story `$1` from `task.md`. The command discovers the HTTP surface of the story, then dispatches **one fresh `dev-test-author` per endpoint** — each agent runs in clean context, covers exactly one route, and ships its own PR into `dev`.

Reply to the user in Russian. Do not enter plan mode — execute.

## Validation

- `$1` must match `^\d+$` (e.g. `12`, `17`).
- If missing or invalid → print «Использование: `/skald:dev-test-story <story-id>`.» and stop.

## Step 1 — Sync working tree to dev

```
git checkout dev && git pull --ff-only
```

- Dirty working copy → stop with «Грязная рабочая копия — закоммить или спрячь изменения и запусти команду снова.» Do not stash.
- Pull conflict / non-fast-forward → stop with the git error and ask user to resolve.

The command reads files in the working tree, so the tree must reflect `dev` to give a deterministic surface.

## Step 2 — Discover iteration bodies

1. Read `task.md` at the project root. If missing → «`task.md` не найден.» stop.
2. Locate every iteration heading for story `$1` in document order:
   - `#### Iteration $1.<N>:`, `#### Итерация $1.<N>:`, `### Iteration $1.<N>`, `### Итерация $1.<N>`.
3. For each iteration capture the body — text between this heading and the next iteration heading (any story) or EOF.
4. Capture the story heading itself for the title.
5. If no iterations → «В `task.md` не найдено итераций истории `$1`.» stop.
6. **Ignore `iteration.md` entirely.** Done-markers are not a precondition.

## Step 3 — Extract candidate routes

Concatenate all iteration bodies and scan for HTTP-route mentions. Union the matches:

- Plain HTTP-mention: `(GET|POST|PATCH|PUT|DELETE)\s+/api/v1/[^\s`)\]]+`
- Cyrillic controller-line: lines containing `Контроллер` or `REST-эндпоинт` followed by a method+path on the same line.
- Public special endpoints if mentioned literally (e.g. `GET /.well-known/jwks.json`).

Deduplicate by `(method, path)` — normalize the path by stripping trailing punctuation `,`/`;`/`:`/`.`/backticks. For each unique route capture the first iteration whose body contained it (`fromIteration`).

If candidate list is empty → print final report «У истории `$1` нет упоминаний HTTP-эндпоинтов в `task.md` — тесты не нужны.» and stop without launching any subagent.

## Step 4 — Cross-check contracts/openapi.yaml

Read `contracts/openapi.yaml`. For each candidate `{method, path}`:

- Path + method present → status `ok`. Capture `requestBody` schema name (or `null`) and `responses` map (`{ status: schema-or-Problem }`).
- Path / method missing → status `blocked_no_openapi`.

## Step 5 — Locate controller files

For each `ok` candidate, Grep across:

- `modules/*/adapter/in/web/**`
- `modules/*/*/adapter/in/web/**`
- `app/src/main/kotlin/**/adapter/in/web/**`

Search for the path literal AND the matching method annotation (`@PostMapping`, `@GetMapping`, `@PatchMapping`, `@PutMapping`, `@DeleteMapping`, or `@RequestMapping(method = ...)`).

- Exactly one matching controller → capture `controllerFile` and `controllerClass`.
- Multiple → pick the file matching the method.
- Not found → `blocked_no_controller`.

## Step 6 — Pull docs context per route

Group `ok` routes by owning module (derive from controller path: `modules/<module>/...`). For each route:

1. Read `docs/tech_spec/api/<module>.md` if it exists.
2. Always read `docs/tech_spec/api/_auth.md` and `docs/tech_spec/api/_conventions.md` — cross-cutting rules (rate-limits, password policy, RFC 7807 problem+json, idempotency).
3. Extract for the brief:
   - error codes mentioned in connection with this path (`weak_password`, `email_taken`, `validation_error`, `invalid_token`, `invalid_credentials`, `oauth_*`, …);
   - rate-limit bullets explicitly tied to the path;
   - header invariants (`Accept-Language` seeding, `ETag`, `If-Match`, `Authorization`, `Set-Cookie` for refresh);
   - acceptance bullets from the iteration body that name the path or method.

Best-effort: if a doc is missing, leave fields empty for that route.

## Step 7 — Pre-flight summary

Before dispatching agents, print a Russian summary so the user (and you) see exactly what will be tested and what will be skipped:

```
# История `$1`: dev-тесты — план

Story: <heading>

К покрытию (<ok-count> эндпоинтов):
- POST /api/v1/auth/register — identity/.../AuthRegisterController.kt (источник: 12.7)
- ...

Заблокировано (<blocked-count>):
- POST /api/v1/auth/login — controller not found
- ... (или «нет»)

Запускаю агентов по одному, последовательно.
```

If zero `ok` routes (all blocked or none found) → print the summary, then write the final report (Step 9) with synthetic skipped result and stop.

## Step 8 — Dispatch agents per endpoint

For each `ok` route in source order:

1. Compute the route-slug: strip leading `/`, strip `api/v1/`, replace remaining `/` with `_`, replace remaining non-alphanumeric (except `_` and `-`) with `_`, lowercase. Examples:
   - `/api/v1/auth/register` → `authregister` (note: also collapse the first segment if single-word — `auth_register` is acceptable; pick `auth_register` for consistency, see below)
   - Use the simpler rule: drop `api/v1/` and lowercase everything else; replace `/` with `_`. So `/api/v1/auth/register` → `auth_register`. `/api/v1/auth/email/verify-confirm` → `auth_email_verify-confirm`. `/.well-known/jwks.json` → `wellknown_jwks`.
2. Build a single-route brief (Markdown). Plain text, easier than JSON for the agent:

   ```
   # Dev-test brief

   Story: <story-id> — <story title>
   Source iteration: <fromIteration>
   Method: <METHOD>
   Path: <path>
   Route slug: <route-slug>

   ## Controller
   <controllerFile>:<controllerClass>

   ## OpenAPI
   - Request body schema: <schema name or "none">
   - Responses:
     - 201 → <schema>
     - 400 → Problem (codes: weak_password, validation_error)
     - 409 → Problem (codes: email_taken)
     - ...

   ## Headers / invariants
   - <Accept-Language seeds users.locale on first authed request>
   - <refresh-cookie httpOnly+Secure+SameSite=Strict, path /api/v1/auth/refresh>
   - (или «нет»)

   ## Rate-limit
   <e.g. 5/IP/час> (or "не задан")

   ## Acceptance (from iteration body)
   - <bullet>
   - <bullet>

   ## Doc references
   - docs/tech_spec/api/<module>.md
   - docs/tech_spec/api/_auth.md § <section>
   - docs/tech_spec/api/_conventions.md § <section>
   ```

3. Launch a **fresh** `dev-test-author` via the Task tool with this prompt (substitute placeholders verbatim, keep the rest as-is):

   ```
   Story ID: <story-id>
   Route: <METHOD> <path>
   Route slug: <route-slug>

   Implement the dev-test class for the single endpoint described in the brief below. Branch off `dev` as `feature/devtest-<route-slug>`. Place tests under `app/src/devTest/kotlin/so/skald/app/devtest/<route-slug>/<Feature>DevApiTest.kt`. Run via `./script/dev-test.sh --tests "..."`. Open one PR into `dev` and squash-merge it. End your message with the RESULT block.

   === BRIEF ===
   <brief markdown>
   === END BRIEF ===
   ```

   Always pass `subagent_type: dev-test-author`. **One agent at a time, sequentially** — do not parallelise (each agent runs `git checkout dev && git pull` and opens/merges a PR; parallel runs would race on the working tree and on `dev`).

4. Parse the agent's `===RESULT===` block (`status`, `pr_number`, `branch`, `merged`, `test_count`, `passed`, `failed`, `notes`). Capture all fields.

5. Stop conditions:
   - `failed` or `blocked` on one route → continue with the next route (one bad endpoint must not stop the rest); flag it in the final report. Exception: if the agent reports a `dirty working tree` blocker, stop the whole loop with that message.
   - `success` / `skipped` → continue.

6. Move to the next route.

## Step 9 — Final report (Russian)

After every `ok` route is processed (or step 7 short-circuited), print exactly:

```
# История `<story-id>`: dev-тесты

Story: <heading>
Эндпоинтов в покрытии: <ok-count>. Заблокировано: <blocked-count>.

## Покрытие
| Метод+путь | PR | Прогон | Статус | Заметки |
| --- | --- | --- | --- | --- |
| POST /api/v1/auth/register | #<pr> ✅ merged | <passed>/<test_count> | ✅ | — |
| GET /api/v1/me/oauth | #<pr> ⚠️ open | 3/3 | ⚠️ merge blocked | branch protection |
| ... |

## Блокировки (тесты не писали)
| Метод+путь | Причина |
| --- | --- |
| POST /api/v1/auth/login | controller not found |
| ... |

(если блокировок нет — таблицу опустить и написать «Блокировок нет.»)
```

Append:
- Все `ok`-маршруты merged и зелёные → «Dev-тесты истории `<id>` зелёные.»
- Хотя бы один failed/blocked/не merged → «Проверь блокировки и/или PR перед промоушеном на prod.»
- Skipped (нет кандидатов или все blocked) → «Тесты не писали.»

No preamble, no trailing commentary outside this report.
