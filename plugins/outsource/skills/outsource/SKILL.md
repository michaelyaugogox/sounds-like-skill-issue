---
name: outsource
description: End-to-end ticket-to-PR workflow. Trigger when the user gives a Jira ticket URL and asks to "outsource", "take", "pick up", "implement", or "ship" it. Fetches the ticket, creates a worktree + branch, implements the change, writes tests if the project has them, and opens a PR with gh.
---

# outsource

Take a Jira ticket from URL → merged-ready PR. Each step has a gate; if a gate fails, stop and report back rather than guessing.

**Plan mode:** if the caller passes `--plan` (via `/outsource`) or asks for a plan, run Step 3 and wait for approval before any file edits. Otherwise skip Step 3.

**One ticket per invocation.** This skill handles a single Jira ticket end-to-end. If the caller has multiple tickets, they should spawn one subagent per ticket in parallel (the `/outsource` command does this automatically when given multiple URLs). Plan mode is incompatible with parallel multi-ticket runs because it requires interactive approval.

## Execution model — subagents within one ticket

Even for a single ticket, this skill is multi-agent. You (the orchestrator running in the main session) coordinate the flow and delegate three phases to subagents:

- **Step 3 (Plan mode only):** a parallel **Explorer** subagent maps the codebase while you draft the plan from the Jira details.
- **Step 5.5 (always):** a **Reviewer** subagent reads the final diff and returns a verdict before you open the PR. Runs in parallel with the test suite executing.
- **Steps 1, 2, 4, 5, 6, 7** stay in the main session — they either need user interaction (Step 1 ambiguity check, Step 3 approval) or benefit from iterative feedback (Step 4 implementation reacting to type/lint errors).

Subagents inside this skill operate in the **same worktree** as the main session (no `isolation: "worktree"` — that flag is for parallel *ticket* fan-out, where each ticket needs its own filesystem). They read and may edit files in the worktree directly.

When invoking a subagent, pass it a self-contained prompt: the ticket key, the worktree path, the specific question or task, and the shape of the response you want back. The subagent has no access to this conversation.

## Inputs

- A Jira ticket URL (required), e.g. `https://gogox.atlassian.net/browse/ABC-1234`.
- Extract the ticket key (`ABC-1234`) from the URL — you'll need it for the branch name and PR title.

## Step 1 — Understand the ticket

Use the Atlassian MCP tool to fetch the ticket:

- `mcp__claude_ai_Atlassian_Rovo__getJiraIssue` with the ticket key.
- Read summary, description, acceptance criteria, comments, linked issues, and attachments.
- Note the issue type (Story / Bug / Task / Chore) — this maps to the conventional-commit `{type}` in step 2.

Summarize back to the user in 3-5 bullets: what's being asked, the acceptance criteria, and any ambiguity you spotted. **Do not proceed if the ticket is ambiguous** — ask the user a clarifying question first.

### Type mapping (Conventional Commits v1.0.0)

The `{type}` MUST come from the Conventional Commits spec. Per the spec, `feat` and `fix` have specific required meanings; other types follow the widely-adopted Angular convention.

| Jira issue type | `{type}` | Notes |
|---|---|---|
| Story / New Feature | `feat` | MUST be used when adding a new feature (correlates with SemVer MINOR). |
| Bug | `fix` | MUST be used when patching a bug (correlates with SemVer PATCH). |
| Refactor / Tech debt | `refactor` | Code change that neither fixes a bug nor adds a feature. |
| Performance | `perf` | Performance improvement. |
| Docs | `docs` | Documentation only. |
| Test | `test` | Adding or correcting tests. |
| Build / Tooling | `build` | Changes to build system or dependencies. |
| CI | `ci` | Changes to CI configuration. |
| Style | `style` | Formatting, whitespace — no code-meaning change. |
| Task / Chore | `chore` | Anything else that doesn't fit above. |

If the ticket implies a breaking change (API removal, behavior change, schema change), use the `!` syntax (see step 5) and confirm with the user before proceeding.

If unclear, default to `chore` and flag the choice.

## Step 2 — Worktree + branch

Derive a brief slug from the ticket summary: lowercase, alphanumeric + hyphens, ~3-6 words.

Branch name format: `{type}/{ticket-key-lowercased}-{brief-description}`
Example: `feat/abc-1234-add-csv-export-to-orders`

Create the worktree off the default branch (usually `main` — verify with `git symbolic-ref refs/remotes/origin/HEAD`):

```bash
git fetch origin
git worktree add -b {branch-name} ../{repo-name}-{ticket-key} origin/main
```

`cd` into the new worktree for all subsequent work. Confirm with `pwd` and `git status` before continuing.

## Step 3 — Plan (gated; only if requested)

**When to run this step:**
- The user passed `--plan` to `/outsource`, OR
- The user explicitly asked for a plan ("show me the plan first", "plan only", "don't implement yet"), OR
- The ticket touches risky areas (auth, migrations, billing, public APIs, anything cross-service) — in that case, *recommend* a plan and ask before skipping.

Otherwise skip to Step 4.

**How to plan:**

1. **Spawn an Explorer subagent in parallel with your own analysis.** In a single response, kick off:
   - An `Agent` call with `subagent_type: "Explore"` and a self-contained prompt: "Inside the worktree at `<path>`, map the code areas relevant to Jira ticket `<KEY>`: `<summary>`. Identify the specific files that would need to change, existing patterns/conventions in those files (test layout, error handling, naming), and any nearby code that might be affected. Return a structured report: relevant files (with one-line purpose each), conventions to follow, and surprises/risks. Do not edit anything."
   - Your own deeper read of the Jira details, linked issues, and acceptance criteria — anything that informs the *what*, not the *where*.
2. When the Explorer returns, merge its findings with your ticket analysis. Present a plan back to the user with these sections:
   - **Approach** — 2-4 sentences on the implementation strategy.
   - **Files to change** — bulleted list with one line per file describing the edit.
   - **New files** — list with purpose, or "none".
   - **Tests** — what you'll add and where.
   - **Risks / open questions** — anything ambiguous, dependencies you noticed, or decisions you want the user's input on.
   - **Out of scope** — things the ticket might imply but you're deliberately not doing.
3. **Stop and wait for explicit approval** — "approved", "go", "lgtm", "ship it", etc. Treat anything else (silence, questions, "what about X") as not-yet-approved and respond accordingly.
4. If the user requests changes to the plan, revise and re-present. Only proceed to Step 4 after approval.

Do not edit any files until the plan is approved. Reading and grepping in the worktree is fine.

## Step 4 — Implement

Now you're a normal coding agent inside the worktree:

- Explore the codebase to find the relevant files before editing.
- Match existing conventions (style, naming, error handling, logging).
- Keep the change scoped to the ticket's acceptance criteria — no drive-by refactors.
- Run the project's typecheck / lint after each meaningful change.

If you hit a blocker (missing context, ambiguous requirement, broken local setup), stop and report. Do not invent requirements.

## Step 5 — Tests

Detect whether the project has tests:

- Look for `*.test.*`, `*.spec.*`, `__tests__/`, `tests/`, `pytest.ini`, `vitest.config.*`, `jest.config.*`, etc.
- Check `package.json` / `pyproject.toml` / `Makefile` for a `test` script.

If tests exist:
- Add tests covering the new behavior and any regression risk.
- Run the full test suite. **Do not open a PR if tests fail** — fix or report.

If no test infrastructure exists, skip this step and note it in the PR description.

## Step 5.5 — Independent review (always)

Before opening the PR, get a fresh perspective on the diff. A reviewer subagent with no implementation context catches scope creep, missed edge cases, and obvious-in-hindsight bugs better than the agent that just wrote the code.

Run the review **in parallel with the test suite** to hide its latency:

1. Stage your changes mentally (you don't need to `git add` yet) and capture the diff: `git diff origin/main...HEAD`.
2. In a single response, kick off both:
   - The test suite (Bash) if you haven't already in Step 5.
   - An `Agent` call with `subagent_type: "general-purpose"` and a self-contained prompt: "You are an independent code reviewer for Jira ticket `<KEY>`: `<summary>`. Acceptance criteria: `<AC>`. Review the diff below against (a) the acceptance criteria — did the change actually deliver what was asked? (b) correctness — bugs, race conditions, error handling gaps. (c) scope — anything that doesn't belong to this ticket. (d) repo conventions visible in surrounding files. Return a verdict of `APPROVE`, `REQUEST_CHANGES`, or `BLOCK`, followed by a bulleted list of findings (most important first). Be concrete: file path, line, what's wrong. Do not edit files. Diff:\n\n<paste full diff here>"
3. When the reviewer returns:
   - `APPROVE` → proceed to Step 6.
   - `REQUEST_CHANGES` → address each finding by looping back to Step 4. Re-run Step 5 and Step 5.5 after the fix. Don't argue with the reviewer; if you genuinely disagree with a finding, surface it to the user before discarding it.
   - `BLOCK` → stop and report to the user. Do not open the PR.

Cap this loop at **two review cycles**. If the reviewer still requests changes after the second pass, stop and hand back to the user with both reviews — the disagreement probably needs human judgment.

## Step 6 — Open the PR

### Commit message — Conventional Commits v1.0.0 (strict)

Follow the spec exactly: https://www.conventionalcommits.org/en/v1.0.0/#specification

**Structure:**

```
<type>[optional scope][!]: <description>

[optional body]

[optional footer(s)]
```

**Rules to honor (verbatim from the spec):**

1. The commit message MUST be prefixed with a type from the table above, in lowercase.
2. A scope MAY be provided in parentheses immediately after the type and MUST be a noun describing a section of the codebase, e.g. `feat(parser):`. Do not put the ticket key in the scope — it goes in the footer.
3. The type/scope prefix MUST be followed by a colon and a single space, then the description.
4. The description is a short summary of the change. Use the imperative mood, lowercase first letter, no trailing period — e.g. `add csv export to orders`, not `Added CSV export.`
5. A longer body MAY follow, separated from the description by **one blank line**. The body is free-form and MAY contain multiple paragraphs.
6. Footers MAY follow the body, separated by **one blank line**. Each footer is `Token: value` or `Token #value`. Tokens MUST use `-` instead of spaces (e.g. `Reviewed-by`, `Refs`). The one exception is `BREAKING CHANGE`, which uses a space.
7. **Breaking changes** MUST be signaled by either (a) appending `!` before the `:` in the prefix, e.g. `feat!:` or `feat(api)!:`, and/or (b) adding a footer `BREAKING CHANGE: <description>`. If `!` is used, the footer MAY be omitted.

**Reference the Jira ticket via a `Refs` footer** — not in the subject line:

```
feat(orders): add csv export endpoint

Adds GET /orders/export?format=csv. Streams rows in batches of 1000 to keep
memory bounded on large tenants. Honors the existing tenant scope middleware.

Refs: ABC-1234
```

Breaking-change example:

```
refactor(auth)!: drop legacy session cookies

Removes the v1 cookie reader. Clients must re-authenticate to receive a v2
token. Migration steps are in docs/auth-v2-migration.md.

BREAKING CHANGE: v1 session cookies are no longer accepted; all clients must
upgrade to the v2 token flow.

Refs: ABC-1234
```

Make one commit unless the change is genuinely staged. If staged, each commit MUST independently satisfy the spec.

### Push and open the PR with `gh`

```bash
git push -u origin {branch-name}
gh pr create --title "{TICKET-KEY} {Brief description}" --body "$(cat <<'EOF'
## Summary
<what this PR changes, 2-4 bullets>

## Ticket
{jira-url}

## Why this is mergeable
- **Scope:** changes are confined to <area>; no unrelated edits.
- **Tests:** <added N tests covering X / no test infra in repo, see note>.
- **Verification:** <typecheck/lint/tests all green; manual steps if any>.
- **Risk:** <call out anything reviewers should pay extra attention to, or "low — isolated change">.

## Test plan
- [ ] <concrete reviewer-runnable step>
- [ ] <…>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

PR title format is exactly `{TICKET-KEY} {Brief description}` (ticket key uppercased, no conventional-commit prefix in the title — the prefix lives in the commit message).

## Step 7 — Hand back

Report to the user with:
- The PR URL.
- The worktree path (so they can `cd` in for follow-ups).
- A one-line summary of what shipped and anything reviewers should know.

## Gotchas

- **Never** push to `main` or force-push. Always work in the worktree branch.
- **Never** skip git hooks (`--no-verify`) without explicit user approval.
- If the ticket links to a parent epic or has subtasks, fetch those too — the real requirement often lives one hop away.
- If the repo has a `CLAUDE.md`, `CONTRIBUTING.md`, or `.github/pull_request_template.md`, read it before step 3 and conform to it (it overrides this skill's PR template).
