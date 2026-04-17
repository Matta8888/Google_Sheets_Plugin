# Changelog

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
