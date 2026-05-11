---
description: Install or update every plugin in the sounds-like-skill-issue marketplace. Idempotent — safe to run repeatedly.
argument-hint: (no arguments)
---

Install or update every plugin in the user's local `sounds-like-skill-issue` marketplace by invoking the `claude plugin` CLI directly via Bash. Do the work — do not just print commands for the user to copy-paste.

## Steps

1. Read `/Users/michaelyau/gogox/sounds-like-skill-issue/.claude-plugin/marketplace.json` (use the Read tool). Extract:
   - The marketplace `name` (top-level field).
   - Every entry under `plugins[]` — collect each plugin's `name`.

2. If the file is missing or the `plugins` array is empty, stop and report the problem to the user. Do not fabricate plugin names.

3. Refresh the marketplace so any newly-added plugin entries are visible:

   ```bash
   claude plugin marketplace update <marketplace-name>
   ```

4. For each plugin name from step 1, run:

   ```bash
   claude plugin install <plugin-name>@<marketplace-name>
   ```

   `claude plugin install` is idempotent — it installs if missing, no-ops if already installed at the same version, updates if a newer version is available. Run installs sequentially so the output is readable and any failure stops the chain cleanly.

5. After all installs succeed, report back to the user with:
   - One line per plugin: name + outcome (newly installed / already current / updated — inferred from the CLI output).
   - A reminder that they must run `/reload-plugins` themselves to apply changes to the current session — `/reload-plugins` has no CLI equivalent and can only be triggered by the user.

6. If any `claude plugin install` fails, stop and surface the error verbatim. Do not retry, do not guess at fixes, and do not proceed to subsequent installs.

## Constraints

- Do not edit any files. This command is read + execute only.
- Do not invoke `/plugin` as a slash command — use the `claude plugin` Bash subcommand. Slash commands cannot be triggered programmatically; the Bash CLI can.
- Do not register the marketplace (`claude plugin marketplace add ...`) automatically. If the update step fails with "marketplace not found", tell the user to run `claude plugin marketplace add /Users/michaelyau/gogox/sounds-like-skill-issue` first, then re-run `/deskill-me`.
