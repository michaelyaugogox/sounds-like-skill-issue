---
name: pr-review
description: Summarize a pull request as Problem → Solution → Breaking changes. Trigger when the user asks to "review", "summarize", "explain", or "look at" a PR — either by URL, by PR number, or for the current branch. Pulls Jira ticket context via the Atlassian MCP when the PR references a ticket key (e.g. ABC-1234).
---

# pr-review

Read a PR and tell the user three things, in this order:

1. **Problem** — what is this PR trying to solve, in plain language. Pull from the linked Jira ticket if there is one; otherwise from the PR description and commit messages.
2. **Solution** — how the PR solves it. A short narrative of the approach, then the concrete file-level changes.
3. **Breaking changes** — anything that would force a caller, consumer, or operator to change behavior. If there are none, say so explicitly — don't omit the section.

This skill is **read-only**. Do not edit files, push commits, or comment on the PR. Output goes to the user as a chat response.

## Audience assumption — write for someone who has never seen this codebase

The reader is reviewing a PR in a repo they don't work in day-to-day. They don't know what `OrderRouter`, `tenant_scope_middleware`, `useFlagBag`, or `apps/ingest-worker` is. They don't know which service is upstream or downstream. They don't know the team's vocabulary.

Concretely, this means:

- **Never use a project-specific noun without a one-clause gloss the first time it appears.** Write `OrderRouter (the service that fans out order-created events to fulfillment, billing, and notifications)` — not just `OrderRouter`. Glosses come from reading the surrounding code, package READMEs, top-of-file comments, or the file/directory name itself. If you can't find a plausible gloss in 30 seconds of grepping, say so explicitly: `OrderRouter (purpose unclear from this PR — appears to handle order routing based on file path)`.
- **Orient before you describe.** Before diving into "this PR changes X", spend one sentence on where in the system X lives: is it a backend service, a frontend component, a shared library, a CLI tool, a migration, a config file? The reader needs the map before the marker.
- **Translate jargon and acronyms** that aren't industry-standard. `tenant scope` → `tenant scope (the row-level filter that isolates each customer's data)`. `feature flag` doesn't need explaining; `GrowthBook gate` does.
- **Show, don't just name.** If a function's role matters to the change, quote its signature or its first few lines once. A 3-line snippet is worth more than a paragraph of prose.
- **Prefer concrete examples over abstract description.** "Renames `getUser` to `fetchUser`" is fine. "Refactors the user-fetching abstraction" is not.

If a section would read as obvious to the PR author but baffling to someone who joined yesterday, rewrite it for the newcomer. The PR author isn't your audience — they already know.

## Inputs

The caller passes one of:

- A GitHub PR URL: `https://github.com/{owner}/{repo}/pull/{number}`
- A bare PR number (operates against the current repo): `1234`
- Nothing — in which case, target the PR for the **current branch**. Resolve via `gh pr view --json number,url` (errors if the branch has no PR; surface that to the user and stop).

## Step 1 — Fetch the PR

Run a single `gh pr view` to get everything you need in one shot:

```bash
gh pr view <number-or-url> --json number,url,title,body,headRefName,baseRefName,author,state,isDraft,additions,deletions,changedFiles,files,commits,labels
```

From the JSON:
- Capture `title`, `body`, `headRefName`, `baseRefName`.
- Capture `files[].path` and the per-file `additions`/`deletions` for the change footprint.
- Capture `commits[].messageHeadline` and `commits[].messageBody` — useful when the PR body is thin.

Then fetch the diff itself:

```bash
gh pr diff <number-or-url>
```

If the diff is enormous (more than ~2000 lines), don't paste it whole into your reasoning. Skim it for the patterns called out in Step 4 and quote only the relevant hunks.

## Step 2 — Find the Jira ticket key

Search **title, body, branch name, and commit headlines** for a Jira-style key matching the regex `[A-Z][A-Z0-9]+-\d+` (e.g. `ABC-1234`, `PLAT-42`).

- Prefer the first key found in the PR title, then body, then branch, then commits.
- If multiple distinct keys appear, capture all of them — the primary is the one in the title (or, if none, the first one in body/branch). The rest are "also referenced".
- If no key is found, skip Step 3.

## Step 3 — Fetch the Jira ticket (if a key was found)

Use `mcp__claude_ai_Atlassian_Rovo__getJiraIssue` with the primary key. Read:

- **Summary** and **description** — these are the problem statement.
- **Acceptance criteria** (usually in the description or a custom field).
- **Issue type** — Story / Bug / Task / Chore tells you what kind of problem this is.
- **Linked issues** — if there's a parent epic or a "blocks/blocked by", note it.
- **Comments** — only skim for late-breaking scope changes or decisions; don't quote routine status updates.

If the MCP call fails (auth, ticket doesn't exist, wrong tenant), don't retry indefinitely — note "Jira lookup failed for {KEY}: {reason}" in the Problem section and fall back to the PR body.

For any "also referenced" keys, do **not** fetch them by default — just list them. Fetch only if the primary ticket is missing or unhelpful and a secondary key looks load-bearing.

## Step 4 — Detect breaking changes

Walk the diff and the file list. A change is breaking if a reasonable caller or operator would have to do work to keep things running after this PR merges. Concrete signals to grep for, by category:

**API surface (HTTP / RPC / GraphQL):**
- Removed or renamed route, endpoint, RPC method, GraphQL field/type.
- Required request parameter added, or response field removed/renamed.
- Status code or error shape changed for an existing endpoint.
- Version bump in a URL path or header (`/v1/` → `/v2/`).

**Public code surface (library / SDK / shared module):**
- Exported function/class/type/constant removed or renamed.
- Function signature change: parameter added without a default, parameter removed, return type narrowed.
- Default value changed in a way that flips behavior.
- A `@deprecated` symbol actually deleted.

**Data layer:**
- Migration that drops or renames a column/table, changes a type, or adds a NOT NULL column without a default + backfill.
- Index removed on a hot path.
- Schema file (`*.proto`, `*.graphql`, `*.openapi.yaml`, `*.sql`) edits that aren't pure additions.

**Configuration / environment:**
- New required env var (no default, code throws if missing).
- Renamed env var, config key, or CLI flag.
- Changed default that affects production behavior (timeouts, retries, feature flags flipped on).

**Build / deploy:**
- Bumped Node/Python/runtime version.
- Removed peer dependency, or a `package.json`/`pyproject.toml` engine constraint tightened.
- Dockerfile base image change that drops bundled tools.

**Commit / PR self-declaration:**
- Conventional Commits `!` marker in a commit subject (`feat!:`, `refactor(api)!:`).
- A `BREAKING CHANGE:` footer in any commit body.
- A `breaking` / `breaking-change` label on the PR.

For each suspected breaking change, quote the file path and the smallest hunk that proves it. If you're not sure whether something is breaking (e.g. removed function that's only used internally), say so and explain the uncertainty — don't bury it.

If you find none of the above, say "**None detected.**" and briefly list what you checked (e.g. "no migrations, no public API edits, no env var changes, no Conventional Commits `!` markers"). The negative case is informative — it tells the reviewer where you looked.

## Step 5 — Report

Output a single chat message with three sections. No preamble, no trailing summary.

**Before you write anything**, gather codebase orientation so you can gloss unfamiliar terms (per the Audience rule above):

- Identify the repo's *type* (monorepo? single service? library?) from the root layout — look at the top-level dirs and the root `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml`.
- For each changed file, find its containing package/service and skim that package's README or top-level docstring. If a `CLAUDE.md`, `README.md`, or `ARCHITECTURE.md` sits next to the file or in its parent dir, read it.
- For each non-trivial symbol the diff touches (class, exported function, route handler, migration), read the surrounding file enough to write a one-clause gloss. Don't paraphrase the diff — explain what the *surrounding code* does so the reader understands the stage the change is happening on.

```
## Problem
{2–5 sentences for a reader unfamiliar with this codebase.

Open with one sentence of *system orientation*: what part of the product/service this PR touches, in plain language. Example: "This PR modifies the orders backend service, specifically the export endpoint that lets tenants download their order history as CSV."

Then state the user-visible problem from the Jira ticket. If the ticket is a Bug, state what was broken and who saw it. If it's a Story/Feature, state what was missing and who asked for it. Quote the Jira summary and one acceptance criterion if it sharpens the picture.

If there is no Jira ticket, reconstruct the problem from the PR body and commits — and say so ("No Jira ticket linked; inferred from PR body and commits:").}

Ticket: [{KEY}]({jira-url}) · type: {Story|Bug|…} · also referenced: {KEY2}, {KEY3} (omit line if no extras)

## Solution
{Start with one sentence that names the approach in plain English — what strategy did the author pick, and why does it solve the Problem above?

Then 1–3 sentences walking through the mechanism. When you mention a project-specific noun (service, module, class, middleware, hook, table), add a one-clause gloss the first time it appears. Example:
"Adds a new `/orders/export` route to the orders API (the HTTP service in `apps/orders-api/` that owns all order CRUD). The route streams rows in batches of 1000 through `csvStream` (a small helper in `packages/shared/streaming/` that wraps Node's `Readable` to emit CSV-formatted chunks), so memory stays flat regardless of tenant size."

**Changes:**
- `path/to/file.ext` — {what this file is (one clause), then what changed and why it matters}
  Example: `apps/orders-api/src/routes/export.ts` — *new file; the HTTP handler for the export endpoint.* Defines `GET /orders/export?format=csv`, validates the `format` query param, and delegates to `OrderExporter`.
- `path/to/other.ext` — {…}

{Group trivial changes ("test fixtures updated", "snapshots regenerated") into a single bullet. For huge PRs, pick the load-bearing files (the ones that *implement* the change, not the ones touched in passing) and end with "+ N other files (tests, fixtures, generated code)".}

## Breaking changes
{For each finding, write enough context that an outsider understands the blast radius. Structure each bullet as: **what changed**, **who calls/depends on this** (so the reader knows the blast radius without already knowing the codebase), **what they have to do**.}

- **{category, e.g. "API surface"}** — `path/to/file:Lstart-Lend`: {what changed}. *Callers:* {who depends on this — other services in this repo, external clients, a CLI command, etc. Grep for usages and name them concretely.} *Migration:* {one-line path forward if it's obvious, otherwise "see PR body" or "unclear — ask author"}.
- …

{Or, if none:}
**None detected.** Checked: migrations, public exports, env vars, route signatures, Conventional Commits `!` markers, `BREAKING CHANGE` footers. {Note which categories were N/A for this repo — e.g. "no schema files; no public SDK surface in this service" — so the reader knows where you looked and where you didn't bother.}
```

**Self-check before sending:** scan your draft and underline every project-specific noun (service names, module names, class names, table names, middleware names, internal jargon). For each underlined term that lacks a gloss on first use, add one. If you can't write a gloss because you don't know what the thing does, write `(unknown — appears to be {your best guess from filename/path})` rather than skipping it.

## Gotchas

- **Draft PRs are fair game** — review them the same way, but mention `state: DRAFT` once in the Problem section so the user knows.
- **Stacked PRs**: if the base branch isn't the repo default (e.g. PR targets another feature branch), call it out — the diff is relative to that base, not `main`. Run `gh pr view --json baseRefName` to confirm.
- **Force-pushes / rewritten history**: `gh pr diff` always reflects the current head, so you're fine, but if commit messages reference removed work, don't be confused.
- **Bot PRs (dependabot, renovate)**: the "Problem" is "dependency X has an update / CVE". Lean on the bot's PR body — there's usually no Jira ticket. Breaking-change detection still applies, especially major-version bumps.
- **Empty PR body + no Jira key**: say so plainly in Problem ("No PR description and no Jira ticket referenced; inferring intent from commits and diff:") and reconstruct from `commits[]` + the diff. Don't fabricate a problem statement.
- **Don't review your own code as if you wrote it.** This skill is for understanding a PR, not for defending implementation choices. Be neutral.
