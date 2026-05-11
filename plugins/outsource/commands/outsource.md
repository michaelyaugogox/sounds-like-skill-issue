---
description: Take one or more Jira tickets from URL to review-ready PRs (worktree → branch → impl → tests → gh pr create). Multiple URLs run in parallel background subagents. Pass --plan to require approval of an implementation plan before any code changes. Pass --bg with a single URL to run it in the background.
argument-hint: [--plan|--bg] <jira-ticket-url> [<jira-ticket-url> ...]
---

Run the `outsource` skill end-to-end for the Jira ticket request(s):

$ARGUMENTS

Parse the arguments:
- If the arguments contain the literal token `--plan`, **plan mode is ON**. Strip the flag from the arguments; the rest is the Jira URL list.
- Otherwise plan mode is OFF.
- Collect all remaining tokens that look like Jira URLs (e.g. start with `https://` and contain `/browse/`). One or more is valid.

### Single URL

Invoke the skill directly via the Skill tool (skill name: `outsource`), passing the Jira URL and the plan-mode state. Tell the skill:
- "Plan mode: ON — run Step 3 and wait for my explicit approval before editing any files." (when `--plan` is present), OR
- "Plan mode: OFF — proceed directly to implementation after Step 2." (otherwise)

Then follow the skill's steps in order. Do not skip the gates — stop and report if the ticket is ambiguous, the plan is rejected, tests fail, or you hit a blocker.

### Multiple URLs (parallel, background)

If two or more Jira URLs are given:

- **`--plan` is not supported with multiple URLs.** Plan mode requires interactive approval per ticket, which doesn't compose with parallel execution. If `--plan` is present with multiple URLs, stop and tell the user to either drop `--plan` or run the tickets one at a time.
- Otherwise, in a **single response**, spawn one `Agent` per URL — all in parallel, all in the background — using:
  - `subagent_type: "general-purpose"`
  - `run_in_background: true`
  - `isolation: "worktree"` (each ticket gets its own isolated worktree so the parallel agents don't collide)
  - A self-contained prompt that names the Jira URL and instructs the subagent to invoke the `outsource` skill with "Plan mode: OFF" and run it end-to-end, reporting back the PR URL, worktree path, and one-line summary.
- Immediately after spawning, tell the user how many agents were launched and that you will report back as each one completes. **Do not poll or sleep** — you will be notified automatically when each background agent finishes.
- As completion notifications arrive, surface each result to the user: the PR URL (or the blocker, if the agent stopped at a gate). When all agents have returned, post a final one-line-per-ticket summary.

Do not run the tickets yourself in the main session when there are multiple URLs — the parallelism is the point.

### Single URL in background

If the user passes `--bg` (or `--background`) alongside a single Jira URL:

- Strip the flag, then spawn a single `Agent` with `run_in_background: true`, `subagent_type: "general-purpose"`, `isolation: "worktree"`, and the same self-contained prompt described above.
- `--bg` and `--plan` are mutually exclusive (plan mode needs interactive approval). If both are present, stop and ask the user which one they want.
- Report back when the agent completes — do not poll.
