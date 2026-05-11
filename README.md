# sounds-like-skill-issue

A personal collection of Claude Code plugins that work universally across projects. Hosted locally on my laptop — not published to any public marketplace.

## Layout

```
sounds-like-skill-issue/
├── .claude-plugin/
│   └── marketplace.json     # local marketplace manifest, lists all plugins
├── starter/                 # template — duplicate to create a new plugin
│   ├── .claude-plugin/plugin.json
│   ├── skills/hello-world/SKILL.md
│   └── commands/ping.md
└── outsource/               # Jira ticket → review-ready PR workflow
    ├── .claude-plugin/plugin.json
    ├── skills/outsource/SKILL.md
    └── commands/outsource.md
```

## Installation

Run these inside a Claude Code session:

```
/plugin marketplace add /Users/michaelyau/gogox/sounds-like-skill-issue
/plugin install outsource@sounds-like-skill-issue
/plugin install starter@sounds-like-skill-issue
```

The first line registers this directory as a local marketplace (one-time setup). The remaining lines install each plugin to user scope (`~/.claude/plugins/`), making them available across all projects.

## Updating

| You changed... | Run |
|---|---|
| Skill/command body content | `/reload-plugins` |
| `marketplace.json` (added a new plugin) | `/plugin marketplace update sounds-like-skill-issue` then `/plugin install <new-plugin>@sounds-like-skill-issue` |

## Useful commands

```
/plugin marketplace list                          # confirm registration
/plugin                                           # interactive UI: enable/disable installed plugins
claude --plugin-dir /path/to/plugin               # one-shot test without installing
```

## Adding a new plugin

1. Duplicate `starter/` to a new directory at the repo root.
2. Update its `.claude-plugin/plugin.json` (name, description).
3. Append an entry to `.claude-plugin/marketplace.json` under `plugins`.
4. `/plugin marketplace update sounds-like-skill-issue` then `/plugin install <new-plugin>@sounds-like-skill-issue`.
