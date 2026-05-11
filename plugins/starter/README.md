# starter

A template plugin. Duplicate this directory to create a new plugin in this marketplace.

## Structure

```
starter/
├── .claude-plugin/
│   └── plugin.json       # plugin manifest
├── skills/
│   └── hello-world/
│       └── SKILL.md      # example skill
└── commands/
    └── ping.md           # example slash command (/ping)
```

## Adding new plugins to this repo

1. Copy `starter/` to a new directory at the repo root, e.g. `my-plugin/`.
2. Update `my-plugin/.claude-plugin/plugin.json` with the new name and description.
3. Add an entry under `plugins` in `../.claude-plugin/marketplace.json`.
4. Run `/plugin install my-plugin@sounds-like-skill-issue` in Claude Code, or `/reload-plugins` if already installed.
