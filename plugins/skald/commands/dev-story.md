---
description: Stories N… — every unfinished iteration in order, then the story gate (script/test on dev)
argument-hint: "<N> [<N> …]"
allowed-tools: Agent, Bash, Read, Skill
---

Process stories `$ARGUMENTS` one by one, in argument order. Formats and rules: `${CLAUDE_PLUGIN_ROOT}/README.md`. Reply to the user in Russian.

Arguments must be one or more story numbers; otherwise show usage `/skald:dev-story <N> [<N> …]` and stop.

For each story N:

1. Find it in `task.md`. Missing → stop. No iterations → stop and suggest `/skald:plan N`.
2. For each unfinished iteration, in `task.md` order, run `/skald:dev <N.M>` via the Skill tool and wait for its outcome. A stop ends the run.
3. **Story gate.** `git checkout dev && git pull --ff-only`, then `./script/test`. Script missing or non-zero exit → the story failed: stop, include the tail of the output.

**Summary.** One line per story: number, title, iterations `<done>/<total>`, gate green, red or not run. On a stop, its details; stories after it are marked not run.
