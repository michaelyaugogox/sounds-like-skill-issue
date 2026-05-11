---
description: Take a Jira ticket from URL to a review-ready PR (worktree → branch → impl → tests → gh pr create). Pass --plan to require approval of an implementation plan before any code changes.
argument-hint: [--plan] <jira-ticket-url>
---

Run the `outsource` skill end-to-end for this Jira ticket request:

$ARGUMENTS

Parse the arguments:
- If the arguments contain the literal token `--plan`, **plan mode is ON**. Strip the flag from the arguments; the rest is the Jira URL.
- Otherwise plan mode is OFF.

Invoke the skill explicitly via the Skill tool (skill name: `outsource`), passing the Jira URL and the plan-mode state. Tell the skill:
- "Plan mode: ON — run Step 3 and wait for my explicit approval before editing any files." (when `--plan` is present), OR
- "Plan mode: OFF — proceed directly to implementation after Step 2." (otherwise)

Then follow the skill's steps in order. Do not skip the gates — stop and report if the ticket is ambiguous, the plan is rejected, tests fail, or you hit a blocker.
