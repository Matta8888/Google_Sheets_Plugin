# sheets-modelling

A Claude Code plugin for building, auditing, and restructuring Google Sheets models that look defensible — not amateur.

**Current version:** 2.0.0

## What's in it

Two skills, one plugin.

### `google-sheets-modelling` — the core

Applies to every Google Sheet interaction: building, cleaning, formatting, auditing.

- **Three-pass build workflow** — Content → **Format** (mandatory phase, not polish) → **Verify** (post-write sanity gate)
- **FAST Standard** workbook structure (About / Inputs / Data / Calcs / Outputs / Checks / Disclaimer / Claude_Log) with the corkscrew-consistency rule
- **Tidy data** (Wickham) for `Data` tabs; long-format for time series
- **Classic WSO colour convention** — blue `#0000FF` for hardcoded inputs, black for formulas, green `#006100` for cross-sheet refs, red `#C00000` for external
- **Number format spec** — explicit patterns per data class (£, £m, £bn, %, multiples, counts, ratios, dates), one unit per tab
- **Column-width + frozen-panes defaults** — pad 20 / label 260 / sub-label 180 / year 95 / notes 160
- **Pre-write discipline** — range + shape declaration, ref-map of named-range targets, `safe_mode: true` on uncertain writes, re-read inputs after writing
- **Magic-number lint** — regex scan to catch hardcoded literals in formulas
- **Post-write sanity gate** — sector-calibrated order-of-magnitude bands, directional signs, bridge ties, no-magic-numbers-remaining
- **Sign-convention assertion** — memo cell + Checks row catches P&L sign flips
- **Volatile-function avoidance** — `TODAY()`, `INDIRECT()`, `IMPORTRANGE()` patterns and their fixes
- **Google Sheets limits** — 10M hard cap, ~1.2M practical slowdown, 60 API req/min
- **Audit-first workflow** — read the existing structure, report findings, wait for explicit approval before any write
- **Mandatory `enable_claude_log`** before every write tool on the paired [Google Sheets MCP](https://github.com/Matta8888/Google_Sheets_MCP)

### `financial-modelling-extras` — for valuation + M&A models

Triggers on DCF / valuation / M&A / synergies / football-field / pro-forma keywords.

- **DCF template** — explicit FCF build, Gordon TV with `FCF_{N+1}` + discount-to-today, exit-multiple TV cross-check
- **Football-field layout** — max 4 methods, low/mid/high with source cells, no average-of-mids
- **M&A pro-forma bridge** — mandatory `Checks` row: `pro_forma_ebitda == sum_of_parts`
- **Synergy structure** — revenue vs cost split, 5-year ramp (25/65/100/100/100), `revenue_synergy_flowthrough = 80%`, integration cost below-the-line
- **Paid vs captured synergies** — standalone EV is what the seller gets; synergy PV is a buyer-side memo, never folded into EV
- **Sector order-of-magnitude bands** — EV/Rev, EV/EBITDA, synergy-PV-as-%-deal-EV

## Install (local)

From a Claude Code session:

```
/plugin marketplace add /Users/matt/Claude Code Projects/sheets-modelling-plugin
/plugin install sheets-modelling@sheets-modelling-local
```

Restart the session. Both skills auto-trigger by their descriptions.

## Upgrade from v1

```
/plugin update sheets-modelling
```

## Pairs with

The [Google Sheets MCP server](https://google-sheets-mcp-production-a4ca.up.railway.app/mcp) (v0.4.0, 34 tools). The skill's formatting phase depends on MCP v2 tools: `set_column_width`, `set_frozen`, `set_conditional_format`, `format_ranges_batch`, `set_data_validation`, `set_sheet_protection`, `merge_cells`, plus write-tool flags (`safe_mode`, `return_computed`, `ROW_COUNT_MISMATCH` with `suggested_range`).

## Iterate

- Edit `skills/<skill-name>/SKILL.md`
- Bump `version` in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`
- Log the change in `CHANGELOG.md`
- `git tag vX.Y.Z`
- `/plugin update sheets-modelling` in Claude Code to pick up the new version

See [CHANGELOG.md](CHANGELOG.md) for version history.
