---
description: One iteration N.M — acceptance tests for a critical story, developer, pr-reviewer, up to 3 fix rounds, nits, merge into dev
argument-hint: "<N.M>"
allowed-tools: Agent, Bash, Read
---

Run the cycle for iteration `$ARGUMENTS` yourself, in the main session. Formats and rules: `${CLAUDE_PLUGIN_ROOT}/README.md`. Reply to the user in Russian.

Argument not of the form `<N>.<M>` → show usage `/skald:dev <N.M>` and stop. Iteration already ticked `[x]` in `iteration.md` → say it is done and stop.

0. **Acceptance tests.** Only for a critical story (its body has `- **Acceptance tests:** first`), and only if `origin` has no `feature/iteration-<N.M>` yet. Run agent `skald:acceptance-tester` with the prompt `Iteration: <N.M>`. `status` is `ok` or `skipped` → step 1; otherwise stop.
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
   No `VERDICT` block → stop; the reason is the reviewer's reply. `APPROVE` with no nits → step 5, `APPROVE` with nits → step 4, `CHANGES_REQUESTED` → step 3.
3. **Fix.** 3 fix rounds already done → stop. Otherwise run `skald:developer`:
   ```
   Mode: fix
   Iteration: <N.M>
   PR: <pr>
   Issues:
   <issues from the latest VERDICT, verbatim>
   Nits:
   <nits from the latest VERDICT, verbatim; omit the field if there are none>
   ```
   `status` is not `ok` → stop. Otherwise go to step 2.
4. **Nits.** Note the PR head, title and description (`gh pr view <pr> --json headRefOid,title,body`) and whether that head has CI checks (`gh pr checks <pr>`), then run `skald:developer`:
   ```
   Mode: fix
   Iteration: <N.M>
   PR: <pr>
   Nits:
   <nits from the latest VERDICT, verbatim>
   ```
   `status` is not `ok` → stop. Then `git fetch` and compare with the noted state: `git diff <noted head> origin/feature/iteration-<N.M>`, and the PR title and description.
   - Every change is a nit edit (README, "Review comment") in a file, the title or the description that the nits cite: no new commits, or the noted head had no CI checks → step 5. Otherwise wait for the new head's checks (`gh pr checks <pr> --watch`); none appear within 10 minutes, or all green → step 5; a red check → as below.
   - Anything else is a fix round. 3 fix rounds were done before it → stop and name the unreviewed commits in the Summary; otherwise step 2.
5. **Merge.** `gh pr merge <pr> --squash --delete-branch` without `--admin`, then `git checkout dev && git pull --ff-only`. Error → stop.

**Summary.** Success: one line with the iteration, the PR, the number of fix rounds, whether nits were fixed and, for a critical story, the acceptance tests status. Stop: the iteration, the step, the PR link if any, the reason and the unresolved issues and nits from the latest `VERDICT`.
