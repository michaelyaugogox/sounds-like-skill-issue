---
name: hello-world
description: Example skill that demonstrates the SKILL.md frontmatter format. Triggers when the user asks for a "hello world" example or sanity-checks that the starter plugin is loaded.
---

# hello-world

This is a sample skill bundled with the `starter` plugin.

When invoked, respond with a short greeting that confirms the plugin is loaded, then explain that the user can duplicate this skill's directory to create a new one.

## Authoring tips

- Keep the frontmatter `description` precise — it's what Claude uses to decide when to load this skill.
- The skill body is loaded into context only when the skill is triggered, so it's fine to write at length here.
- Reference related files with relative paths from this SKILL.md.
