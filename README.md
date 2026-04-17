# sheets-modelling

A Claude Code plugin containing the **google-sheets-modelling** skill — best-practice guidance for building, auditing, and restructuring Google Sheets.

**Current version:** 1.0.0

## What's in it

One skill: `google-sheets-modelling`. It encodes:

- **The FAST Standard** (Flexible, Appropriate, Structured, Transparent) with the corkscrew-consistency rule
- **Tidy data** (Wickham's three rules) for `Data` tabs; long-format for time series
- **Classic WSO colour convention** — blue font for hardcoded inputs, black for formulas, green for cross-sheet refs, red for external
- **Formula hygiene** — named ranges for constants, closed ranges over open, no magic numbers, consistent formulas across rows
- **Volatile-function avoidance** — `TODAY()`, `INDIRECT()`, `IMPORTRANGE()` patterns and their fixes
- **Google Sheets limits** — 10M cells hard cap, ~1.2M practical slowdown, 60 API writes/min
- **Audit-first workflow** — read the existing structure, report findings, wait for explicit user approval before any write
- **Mandatory `enable_claude_log`** — before any write tool on the paired [Google Sheets MCP](https://github.com/Matta8888/Google_Sheets_MCP)

## Install (local)

From a Claude Code session, add this directory as a local marketplace and install the plugin:

```
/plugin marketplace add /Users/matt/Claude Code Projects/sheets-modelling-plugin
/plugin install sheets-modelling@sheets-modelling-local
```

The skill auto-triggers when you ask Claude to build, audit, or modify a Google Sheet.

## Pairs with

The [Google Sheets MCP server](https://google-sheets-mcp-production-a4ca.up.railway.app/mcp) — the skill's concrete workflows map directly onto the MCP's 24 tools (`write_range`, `format_range`, `set_named_range`, `set_cell_note`, `enable_claude_log`, etc.).

## Iterate

- Edit `skills/google-sheets-modelling/SKILL.md`
- Bump `version` in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`
- Log the change in `CHANGELOG.md`
- `/plugin update sheets-modelling` in Claude Code to pick up the new version

See [CHANGELOG.md](CHANGELOG.md) for version history.
