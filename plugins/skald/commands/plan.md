---
description: Plan iterations for story N, or one by one for every story without iterations
argument-hint: "[<N>]"
allowed-tools: Agent, Read
---

Formats: `${CLAUDE_PLUGIN_ROOT}/README.md`. Reply to the user in Russian.

`$ARGUMENTS` is empty or a story number; otherwise show usage `/skald:plan [<N>]` and stop.

1. Targets from `task.md`:
   - with N: story N; if it is missing or already has iterations, say so and stop;
   - without N: every story without iterations, in file order; none → say so and stop.
2. For each target in turn, never in parallel, run agent `skald:iteration-planner` with the prompt `Story: <N>`. Then check that the story now has iterations in `task.md`; if not, record an error and move on.

**Summary.** One line per story: number, title, number of iterations or the error, and the planner's note on a story title that names a story or iteration number. Do not commit: the edits stay in the working tree.
