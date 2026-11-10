# skald

Claude Code plugin: stories in `task.md` → iterations → PRs into `dev` → review → merge, with test gates. Language-agnostic, built around `git` and GitHub PRs.

Requires `git`, an authenticated `gh`, a `dev` branch on `origin` and `script/test` (see "Test contract").

If `dev` is protected, require PRs and checks but not approvals: the reviewer posts a comment, because GitHub does not let you approve your own PR.

## Commands

| Command | What it does |
| --- | --- |
| `/skald:plan [<N>]` | Plans iterations for story N, or one by one for every story without iterations. Edits stay in the working tree. |
| `/skald:dev <N.M>` | One iteration: `developer` → `pr-reviewer` → up to 3 fix rounds → merge. |
| `/skald:dev-story <N> [<N> …]` | Every unfinished iteration of each story in order, then the story gate. |
| `/skald:dev-all` | Every unfinished story in `task.md` order until none are left or a run stops. A story without iterations is planned first; the plan is merged through a PR. |
| `/skald:review <N.M> <PR>` | Manual PR review, no merge. |
| `/skald:fix <N.M> <PR>` | Manual PR fix from the latest review. |

## Agents

- `iteration-planner` — writes one story's iterations to `task.md` and their lines to `iteration.md`.
- `developer` — mode `new`: implements an iteration and opens a PR into `dev`; mode `fix`: addresses review issues on that PR.
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

`iteration.md`:

```markdown
# Iterations

## Story <N>: <title>
- [ ] **<N>.<M>:** <iteration title>
```

- The planner creates the lines; the developer ticks `[x]` in its iteration's PR.
- An iteration is done when its line is `[x]`. A story is done when all its iterations are done.

## Test contract

- The project root has an executable `script/test`: no arguments, non-interactive, builds the project and runs all tests. Exit code 0 only if everything passed. The working tree stays clean.
- The plugin knows no language and has no build or test commands. What to test and what infrastructure is needed is up to the project.
- No `script/test` yet — the developer creates it in the current iteration from the project's stack. The reviewer checks that it is a real test run, not a stub.
- The developer opens a PR only when `script/test` is green.
- The reviewer runs `script/test` on the PR branch itself. If the PR has CI checks (`gh pr checks`), it waits for them to finish. Red tests or CI → `CHANGES_REQUESTED` with the error output.
- New behavior is covered by tests in the same iteration. Tests are never deleted, skipped or weakened to get green. A flaky test is reported, not worked around.
- A story ends with a gate: after its last iteration is merged, `script/test` passes on a fresh `dev`.
- Long commands (`script/test`, `gh pr checks --watch`) run with a Bash timeout of 600000 ms.

## Agent rules

- Before starting, read the project rules: `AGENTS.md`, `CLAUDE.md`, `rules.md` in the root, whichever exist. Style, tests and the format of commits and PR titles follow them.
- Everything written to the repository or GitHub is English: iterations, `iteration.md`, code, commits, PRs, review comments. Only chat replies are Russian.
- The base branch is `dev`; the iteration branch is `feature/iteration-<N>.<M>`.
- Changes reach `dev` only through a PR merged by the orchestrator. Never commit or push to `dev` directly.
- A dirty working tree or a git error means stop. No `stash`, `reset`, force-push, or bypassing hooks or signing (`--no-verify`).
- Touch only your own iteration or story.
- A `RESULT` or `VERDICT` block uses its keys verbatim and comes last in the reply.
