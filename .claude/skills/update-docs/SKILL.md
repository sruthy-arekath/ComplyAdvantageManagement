---
name: update-docs
description: Bring docs/TECHNICAL_DOCUMENTATION.md in line with the current code changes. Use after small or medium tasks that changed behaviour, config, endpoints, Excel layouts or business rules.
disable-model-invocation: true
---

1. Run `git status` and `git diff` against the base branch named in CLAUDE.md to see what changed.
2. If nothing documentable changed, say so and stop.
3. Otherwise dispatch the `tech-writer` agent with a short summary of the change ($ARGUMENTS if given).
4. Show me the sections it changed. Don't commit.
