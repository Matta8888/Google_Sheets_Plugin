# Changelog

## 2.0.0 — 2026-04-18

Closes the gap between "technically correct spreadsheet" and "defensible document an executive would open without wincing."

### Added — main skill (`google-sheets-modelling`)

- **Three-pass build workflow** — Content → Format → Verify. Formatting is no longer "polish"; it's a mandatory phase with a literal checklist.
- **Formatting phase checklist** — font colour, number format, column widths, frozen panes, borders, conditional format, data validation, sheet protection. All explicit, all checkboxed, all mapped to specific MCP tools.
- **Number format spec** — explicit patterns per data class (£, £m, £bn, %, multiples, counts, ratios, dates). Auto-inference is no longer acceptable. **One unit per tab** rule.
- **Column width + frozen-panes defaults** — pad 20 / label 260 / sub-label 180 / year 95 / notes 160. Any tab with >10 data rows freezes header + label column.
- **Pre-write discipline** — declare range + shape before every write; maintain a ref-map of named-range targets at the top of each turn (Ctrl-F-catchable row drift); use `safe_mode: true` on uncertain writes; re-read inputs after writing.
- **Magic-number lint** — regex scan of FORMULA-rendered content to surface hardcoded numbers in formulas; they must be extracted to `Inputs` or justified with `set_cell_note`.
- **Post-write sanity gate** — four mandatory checks before declaring done: order-of-magnitude (sector-calibrated bands), directional sign, bridge ties, no-magic-numbers-remaining.
- **Sign-convention assertion** — every model has a memo cell stating the convention + a `Checks` row asserting a representative cost cell is negative.
- **Protection policy** — `warningOnly: true` on `About` and `Checks` only; formula tabs and `Inputs` stay unprotected so Claude and user respectively can edit.
- **Expanded rationalisations table** — adds 8 new battle-tested excuse→reality pairs covering formatting-as-polish, format-inference, UI-fix-after, 4-method averaging, read-back skipping, `safe_mode`, `ROW_COUNT_MISMATCH`, sanity-gate skipping.
- **Expanded red-flag table** — adds items for missed formatting phase, skipped sanity gate, mixed units, proceeding past `ROW_COUNT_MISMATCH`.
- **MCP tool mapping (v2)** — reflects the 10 new MCP tools shipped alongside (`set_column_width`, `set_row_height`, `auto_resize_columns`, `set_frozen`, `set_conditional_format`, `merge_cells`, `format_ranges_batch`, `set_data_validation`, `set_sheet_protection`, `move_sheet`) and v2 features on existing tools (`ROW_COUNT_MISMATCH` + `suggested_range`, `safe_mode`, `return_computed`).

### Added — new sub-skill `financial-modelling-extras`

Triggers on DCF / valuation / M&A / synergies / football-field / pro-forma keywords.

- **DCF template** — explicit FCF build, named ranges for `wacc` / `terminal_growth` / `forecast_years`, Gordon TV using `FCF_{N+1}` (not `FCF_N`) with discount-to-today, exit-multiple TV cross-check on `Checks` (flag if >30% spread)
- **Football-field layout** — max 4 methods, low/mid/high with source cells, sorted by mid descending, no average-of-mids shown (collapses the range)
- **M&A pro-forma bridge** — mandatory `Checks` row: `pro_forma_ebitda == sum_of_parts ± integration`
- **Synergy structure** — revenue-vs-cost split, 5-year ramp row (25/65/100/100/100), `revenue_synergy_flowthrough = 80%` default, integration cost below-the-line
- **Paid vs captured synergies** — standalone EV is what the seller gets; synergy PV is a buyer-side memo, never folded into EV
- **Sign-convention assertions** specific to M&A (costs, integration, dis-synergies all negative)
- **Sector order-of-magnitude bands** — EV/Rev, EV/EBITDA, synergy-PV-as-%-deal-EV, integration-cost-as-%-year-1-synergies

### Tooling dependency

Main skill's formatting phase depends on MCP tools shipped in Google Sheets MCP v0.4.0 (34-tool release): `set_column_width`, `set_frozen`, `set_conditional_format`, `format_ranges_batch`, `set_data_validation`, `set_sheet_protection`, `merge_cells`, plus v2 flags on `write_range` (`safe_mode`, `return_computed`, `ROW_COUNT_MISMATCH` with `suggested_range`).

## 1.0.0 — 2026-04-17

Initial release. Packaged the `google-sheets-modelling` skill as a Claude Code plugin.

### Included
- FAST Standard workbook structure (About / Inputs / Data / Calcs / Outputs / Checks)
- Tidy data rules for `Data` tabs (long format required for time series)
- Classic WSO colour convention — blue `#0000FF` font for hardcoded inputs, black for formulas, green `#006100` for cross-sheet refs, red `#C00000` for external refs
- Mandatory `enable_claude_log` before every write tool — non-negotiable
- Audit-first workflow with 4 scan steps → findings report → wait for user approval
- Volatile-function table (`TODAY`, `NOW`, `INDIRECT`, `OFFSET`, `IMPORTRANGE`) with fixes
- `err_tol = 0.0001` named-range pattern for floating-point-safe integrity checks
- Google Sheets limits (10M cells, 5M per tab, ~1.2M practical slowdown, 60 API req/min)
- MCP tool mapping — each modelling task mapped to a specific Sheets MCP tool
- Red-flag table + rationalisations counter-table

### Verified
- RED test: baseline subagent produced an idiosyncratic yellow-fill palette and no audit-first posture
- GREEN test: skill-aware subagent produced FAST-aligned plan with correct colour convention and explicit "awaiting go before any write"
- Pressure test: skill-aware subagent refused to skip `enable_claude_log` even when user said "just do it"
