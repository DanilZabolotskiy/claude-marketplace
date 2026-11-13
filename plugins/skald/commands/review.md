---
description: Manual review of iteration N.M's PR by pr-reviewer, no merge
argument-hint: "<N.M> <PR>"
allowed-tools: Agent
---

`$ARGUMENTS` is `<N.M> <PR>`, where PR is a number or URL; otherwise show usage `/skald:review <N.M> <PR>` and stop. Reply to the user in Russian.

Run agent `skald:pr-reviewer` with the prompt:

```
Iteration: <N.M>
PR: <PR>
```

Show the verdict, issues, nits and notes. The full report is in the PR comment. Merging is manual.

With notes: squash with a commit body free of those story and iteration numbers, built as in `/skald:dev` step 5.
