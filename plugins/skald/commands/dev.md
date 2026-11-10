---
description: One iteration N.M — developer, pr-reviewer, up to 3 fix rounds, merge into dev
argument-hint: "<N.M>"
allowed-tools: Agent, Bash, Read
---

Run the cycle for iteration `$ARGUMENTS` yourself, in the main session. Formats and rules: `${CLAUDE_PLUGIN_ROOT}/README.md`. Reply to the user in Russian.

Argument not of the form `<N>.<M>` → show usage `/skald:dev <N.M>` and stop. Iteration already ticked `[x]` in `iteration.md` → say it is done and stop.

1. **Develop.** Run agent `skald:developer` with the prompt:
   ```
   Mode: new
   Iteration: <N.M>
   ```
   Take `status`, `pr`, `notes` from `RESULT`. `status` is not `ok` → stop.
2. **Review.** Run `skald:pr-reviewer`:
   ```
   Iteration: <N.M>
   PR: <pr>
   ```
   No `VERDICT` block → stop; the reason is the reviewer's reply. `APPROVE` → step 4, `CHANGES_REQUESTED` → step 3.
3. **Fix.** 3 fix rounds already done → stop. Otherwise run `skald:developer`:
   ```
   Mode: fix
   Iteration: <N.M>
   PR: <pr>
   Issues:
   <issues from the latest VERDICT, verbatim>
   ```
   `status` is not `ok` → stop. Otherwise go to step 2.
4. **Merge.** `gh pr merge <pr> --squash --delete-branch` without `--admin`, then `git checkout dev && git pull --ff-only`. Error → stop.

**Summary.** Success: one line with the iteration, the PR and the number of fix rounds. Stop: the iteration, the step, the PR link if any, the reason and the unresolved issues from the latest `VERDICT`.
