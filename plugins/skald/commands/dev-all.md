---
description: Every unfinished story in task.md order until none are left or a run stops; plans a story first if it has no iterations
allowed-tools: Agent, Bash, Read, Skill
---

Formats and rules: `${CLAUDE_PLUGIN_ROOT}/README.md`. Reply to the user in Russian.

`git status --porcelain` is not empty → report a dirty working tree and stop.

Repeat:

1. `git checkout dev && git pull --ff-only`. Take the first unfinished story N in `task.md` order. None → done. N was already processed in this run → stop: its iterations are not ticked `[x]`.
2. N has no iterations → run `/skald:plan N` via the Skill tool. Still no iterations → stop. Otherwise merge the plan through a PR:
   - `git checkout -b plan/<slug>` (README, "Public text"), commit `task.md` and `iteration.md` (message as the plan PR title in the README format), `git push -u origin plan/<slug>`;
   - `gh pr create --base dev`, title as the commit message, description: the list of iteration titles;
   - if the PR has CI checks, `gh pr checks <pr> --watch`; red → stop;
   - `gh pr merge <pr> --squash --delete-branch` without `--admin`, then `git checkout dev && git pull --ff-only`.

   Error → stop.
3. Run `/skald:dev-story N` via the Skill tool. A stop ends the run.

**Summary.** The story lines from `/skald:dev-story` and why the run ended: all stories done, or the stop details.
