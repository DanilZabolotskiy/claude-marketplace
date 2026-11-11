---
description: Manual fix of iteration N.M's PR by developer, from the latest skald review on the PR
argument-hint: "<N.M> <PR>"
allowed-tools: Agent
---

`$ARGUMENTS` is `<N.M> <PR>`, where PR is a number or URL; otherwise show usage `/skald:fix <N.M> <PR>` and stop. Reply to the user in Russian.

Run agent `skald:developer` with the prompt:

```
Mode: fix
Iteration: <N.M>
PR: <PR>
```

It takes the issues and nits from the latest review report on the PR. Show its outcome. Re-review with `/skald:review`.
