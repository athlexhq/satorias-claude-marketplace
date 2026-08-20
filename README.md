# Satorias Claude Marketplace

A Claude Code plugin marketplace for general software development and
management, maintained for use by Satorias employees and associates.

## Installing

Add this marketplace to Claude Code, then install a plugin from it:

```
/plugin marketplace add athlexhq/satorias-claude-marketplace
/plugin install toolsy
```

## Plugins

- **toolsy** — an opinionated set of skills covering the pull request and
  release workflow: opening and updating pull requests, writing changelog
  entries, and cutting GitHub releases.

## Contributing

Plugins live under `plugins/<plugin-name>/`, with a `.claude-plugin/plugin.json`
manifest and skills under `skills/<skill-name>/SKILL.md`. New plugins must be
registered in `.claude-plugin/marketplace.json` to be installable.
