---
name: feature
description: Full pipeline for a large or cross-cutting feature - plan with architect, wait for approval, branch, implement, build and test, review with reviewer, update docs with tech-writer, hand off.
disable-model-invocation: true
---

Run this pipeline for the feature described in the arguments: $ARGUMENTS

PHASE 1 - PLAN (no code changes allowed in this phase)
1. Dispatch the `architect` agent with the feature description, passing my request verbatim (plus any clarifications I gave in this conversation); it is the requirement.
2. Save its plan to docs/plans/<yyyy-MM-dd>-<task-number>-<short-kebab-name>.md
   (omit the task number if there is none yet).
3. Show me: a 5-10 line summary of the plan, the open questions with their owners, and the file path.
4. STOP. End your turn. Do not create, edit or delete any other file, and do not run any
   command that changes anything, until I reply "approved".
5. If I reply with changes instead, update the plan file (re-dispatch the architect if the change
   is large), show me what changed, and STOP again. Repeat until I say "approved".

PHASE 2 - BRANCH (only after "approved")
6. Follow the Git workflow in CLAUDE.md: check `git status` and the current branch; get the
   DevOps task/bug number if missing (never invent one); create or switch to the right
   `task-*` / `bug-*` branch from the latest base branch. If anything is unclear, stop and ask.

PHASE 3 - IMPLEMENT
7. Implement the approved plan step by step in this session. Follow it exactly; if you must
   deviate, stop and ask me first.
8. Validate: `dotnet build ComplyAdvantageManagement.slnx` (no errors, no new warnings) and `dotnet test ComplyAdvantageManagement.slnx`
   (all pass). Fix failures. Don't run `dotnet run`.
9. Never call a mutating ComplyAdvantage endpoint, not even to test.

PHASE 4 - REVIEW
10. Dispatch the `reviewer` agent with the plan file path.
11. Fix every HIGH finding, then build and test again. If a fix changes the design, tell me before doing it.
    Re-dispatch the reviewer once if HIGH findings were fixed.

PHASE 5 - DOCUMENT
12. Dispatch the `tech-writer` agent with the plan file path and a summary of what changed.
13. Check its changes: docs match the code, no secrets or personal data, changelog row added.

PHASE 6 - HAND-OFF
14. Show the completion summary defined in CLAUDE.md: implementation summary, changed files,
    build and test results, diff summary, current branch, documentation sections updated,
    remaining MEDIUM/LOW findings, open questions with owners, and the manual browser checks.
15. STOP and wait for a hand-off command (`commit`, `commit only`, `push`, `create PR`).
