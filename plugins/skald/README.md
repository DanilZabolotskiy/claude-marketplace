# skald

Claude Code plugin: stories in `task.md` → iterations → PRs into `dev` → review → merge, with test gates. Language-agnostic, built around `git` and GitHub PRs.

Requires `git`, an authenticated `gh`, a `dev` branch on `origin` and `script/test` (see "Test contract").

If `dev` is protected, require PRs and checks but not approvals: the reviewer posts a comment, because GitHub does not let you approve your own PR.

## Commands

| Command | What it does |
| --- | --- |
| `/skald:plan [<N>]` | Plans iterations for story N, or one by one for every story without iterations. Edits stay in the working tree. |
| `/skald:dev <N.M>` | One iteration: `acceptance-tester` for a critical story → `developer` → `pr-reviewer` → up to 3 fix rounds → nits → merge. |
| `/skald:dev-story <N> [<N> …]` | Every unfinished iteration of each story in order, then the story gate. |
| `/skald:dev-all` | Every unfinished story in `task.md` order until none are left or a run stops. A story without iterations is planned first; the plan is merged through a PR. |
| `/skald:review <N.M> <PR>` | Manual PR review, no merge. |
| `/skald:fix <N.M> <PR>` | Manual PR fix from the latest review. |

## Agents

- `iteration-planner` — writes one story's iterations to `task.md` and their lines to `iteration.md`; marks a critical story.
- `acceptance-tester` — for a critical story, writes an iteration's acceptance tests before its code.
- `developer` — mode `new`: implements an iteration and opens a PR into `dev`; mode `fix`: addresses review issues or nits on that PR.
- `pr-reviewer` — reviews the PR, runs the tests itself, posts its report on the PR, returns a verdict. Never merges.

Orchestration runs in the main session, inside the commands. Agents never call other agents.

## Formats

`task.md`:

```markdown
### Story <N>: <title>

<story body in the project's own format>

#### Iteration <N>.<M>: <title>
- **Goal:** …
- **What we do:**
  - …
- **Definition of done:** …
```

- A story runs from its heading to the next heading of level `###` or higher. An iteration runs to the next heading of level `####` or higher.
- Stories and iterations go in file order, not by number.
- A story whose body has the line `- **Acceptance tests:** first` is critical. The planner adds the line when the project rules define critical stories and the story fits; without such a rule, never.

`iteration.md`:

```markdown
# Iterations

## Story <N>: <title>
- [ ] **<N>.<M>:** <iteration title>
```

- The planner creates the lines; the developer ticks `[x]` in its iteration's PR.
- An iteration is done when its line is `[x]`. A story is done when all its iterations are done.

## PR

Title — Conventional Commits, unless the project rules set another format: `<type>(<scope>): <summary>`.
- `type`: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `build`, `ci`, `chore`; a breaking change adds `!` before the colon.
- `scope`: a module of the project; optional.
- Summary: imperative, lowercase, up to 72 characters, about the result. No iteration number: it is in the description and the branch name.
- Plan PR: `chore(plan): plan story <N>`.

Description:

```markdown
Iteration <N>.<M> — <iteration title> (story <N>)

## Changes
- <1–5 bullets on behavior, not on files>

## Decisions
- <non-obvious decisions>

## Tests
- <test type: what is covered>

## Notes
- <flaky tests, known limitations>
```

- `Decisions` and `Notes` only when there is something to say; otherwise omit them.
- Do not restate the task or write "This PR…". `script/test` is green by definition.

After a fix round or a nits fix the developer comments on the PR: `Fixes for review <k>: 1 — fixed (<sha>); N1 — fixed (description)`. A nit whose fix needed more than a nit edit: `N2 — fixed (<sha>|title|description, not a nit edit)`.

## Review comment

Changes requested:

```markdown
<!-- skald-review -->
## ❌ Changes requested — iteration <N>.<M>, review <k>/4

1. **<Category>** `<path:line>`[, `<path:line>` …] — <problem>. → <what to do>.

**Nits:**
N1. **<Category>** `<path:line>`[, `<path:line>` …] — <problem>. → <what to do>.

**Previous review:** <per issue and nit number: fixed or still open>
**Checks:** `script/test` ✅|❌ · CI ✅|❌ <failed check>|none
<details><summary><check> output</summary>

<failures only>
</details>
```

Approved:

```markdown
<!-- skald-review -->
## ✅ Approved — iteration <N>.<M>, review <k>/4

- Iteration: all of "What we do" and "Definition of done" are done.
- Tests: <new tests, briefly> · `script/test` ✅ · CI ✅|none

**Nits:**
N1. **<Category>** `<path:line>`[, `<path:line>` …] — <problem>. → <what to do>.
```

- Categories: `Iteration`, `Bug`, `Tests`, `Security`, `Rules`.
- No matters of taste or praise.
- One issue or nit per rule broken, listing every place that breaks it. The developer fixes exactly the listed places.
- A nit, from review 2 only: a problem whose fix is a nit edit — it rewords, reformats or removes repeated prose in the files the nit cites (docs, code comments no tool reads, the PR title or description) and changes no fact, requirement or behavior. Anything else is an issue. Nits do not block. With changes requested, the fix round fixes them too; after an approval, `/skald:dev` has them fixed and checks the edit itself instead of another review.
- `<k>`: earlier `<!-- skald-review -->` comments on the PR plus one. At most 4: the first review and 3 fix rounds.
- `Previous review` only from review 2; `Nits` only when there are some; `<details>` only when a check failed.

## Test contract

- The project root has an executable `script/test`: no arguments, non-interactive, builds the project and runs all tests. Exit code 0 only if everything passed. The working tree stays clean.
- The plugin knows no language and has no build or test commands. What to test and what infrastructure is needed is up to the project.
- No `script/test` yet — the developer creates it in the current iteration from the project's stack. The reviewer checks that it is a real test run, not a stub.
- The developer opens a PR only when `script/test` is green.
- The reviewer runs `script/test` on the PR branch itself. If the PR has CI checks (`gh pr checks`), it waits for them to finish. Red tests or CI → `CHANGES_REQUESTED` with the error output.
- New behavior is covered by tests in the same iteration. Tests are never deleted, skipped or weakened to get green. A flaky test is reported, not worked around.
- A story ends with a gate: after its last iteration is merged, `script/test` passes on a fresh `dev`.
- Tests must catch broken behavior. The reviewer runs `script/mutants origin/dev` if the project has it; otherwise it breaks 2–3 key behaviors of the iteration by hand, runs the tests and reverts. A breakage no test catches is a `Tests` issue. Iterations without code skip this.
- `script/mutants <base-ref>` is optional: non-interactive, mutates only the code changed since `<base-ref>`, exit code 0 only if the tests catch every mutant. The working tree stays clean.
- Long commands (`script/test`, `script/mutants`, `gh pr checks --watch`) run with a Bash timeout of 600000 ms.

## Acceptance tests

For each iteration of a critical story, before any code:
- `acceptance-tester` writes tests of every item of the iteration's `Definition of done` and of the behavior the specs fix for it. Black box: only through interfaces a user or another component sees (API, CLI, protocol, a module's public API). Names come from the specs; a missing name the tester chooses and lists in the commit body.
- It commits only the tests as `test(acceptance): iteration <N>.<M>` on `feature/iteration-<N>.<M>` and pushes without a PR. The new tests fail because the code is missing; everything else stays green.
- An iteration with nothing to test, such as docs only → `skipped`, and it runs as usual.
- The developer builds on that branch. It changes an acceptance test only to fix a defect in the test, and lists each change with its reason under `Decisions` in the PR.
- The reviewer checks every change to those tests (`git diff <acceptance commit> HEAD -- <its files>`). A change that is unlisted or weakens a check is a `Tests` issue.

## Agent rules

- Before starting, read the project rules: `AGENTS.md`, `CLAUDE.md`, `rules.md` in the root, whichever exist. Style, tests and the format of commits and PR titles follow them.
- Everything written to the repository or GitHub is English: iterations, `iteration.md`, code, commits, PRs, review comments. Only chat replies are Russian.
- The base branch is `dev`; the iteration branch is `feature/iteration-<N>.<M>`.
- Changes reach `dev` only through a PR merged by the orchestrator. Never commit or push to `dev` directly.
- `gh pr comment` runs with `dangerouslyDisableSandbox: true`: the sandbox silently drops this write.
- A dirty working tree or a git error means stop. No `stash`, `reset`, force-push, or bypassing hooks or signing (`--no-verify`).
- Touch only your own iteration or story.
- A `RESULT` or `VERDICT` block uses its keys verbatim and comes last in the reply.
