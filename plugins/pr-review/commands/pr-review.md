---
description: Summarize a PR as Problem → Solution → Breaking changes. Pulls Jira context if the PR references a ticket. Pass a PR URL, a PR number, or nothing (defaults to the current branch's PR).
argument-hint: [<pr-url-or-number>]
---

Run the `pr-review` skill against:

$ARGUMENTS

Parse the argument:
- If empty, target the PR for the current branch (the skill resolves it via `gh pr view`).
- If it looks like a GitHub PR URL (`https://github.com/.../pull/<n>`), pass it through verbatim.
- If it's a bare number, pass it through — the skill scopes to the current repo.

Invoke the `pr-review` skill via the Skill tool with that input. Do not implement the review inline — let the skill drive.
