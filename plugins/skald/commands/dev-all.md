---
description: Every unfinished story in task.md order until none are left or a run stops; plans a story first if it has no iterations
allowed-tools: Agent, Bash, Read, Skill
---

Formats and rules: `${CLAUDE_PLUGIN_ROOT}/README.md`. Reply to the user in Russian.

`git status --porcelain` is not empty → report a dirty working tree and stop.

Repeat:

1. `git checkout dev && git pull --ff-only`. Take the first unfinished story N in `task.md` order. None → done. N was already processed in this run → stop: its iterations are not ticked `[x]`.
2. N has no iterations → run `/skald:plan N` via the Skill tool. Still no iterations → stop. Otherwise commit `task.md` and `iteration.md` to `dev` (message per the project rules, else `plan: story N`) and `git push`. Error → stop.
3. Run `/skald:dev-story N` via the Skill tool. A stop ends the run.

**Summary.** The story lines from `/skald:dev-story` and why the run ended: all stories done, or the stop details.
