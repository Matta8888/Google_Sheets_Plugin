---
name: google-sheets-modelling
description: Use when building, auditing, restructuring, or formatting any Google Sheet — financial models, trackers, dashboards, raw-data logs, or forecast models — especially via the Google Sheets MCP. Covers the FAST standard, tidy data, the classic blue/black/green/red colour convention, formula hygiene, Google Sheets performance, and a three-pass build workflow (content → format → verify) with mandatory pre-write and post-write gates so models ship looking defensible, not amateur.
---

# Google Sheets modelling

## Core principle

Apply the **FAST Standard** (Flexible, Appropriate, Structured, Transparent) and **tidy data** to every sheet. Build in **three passes**:

1. **Content** — get the structure, values, and formulas right
2. **Format** — apply the formatting checklist as a mandatory phase (not "polish")
3. **Verify** — run the post-write sanity gate before declaring done

**Audit before you write**: read the current structure, produce a findings report, wait for user approval before changing anything on an existing sheet.

Violating the letter of the rules is violating the spirit. An unformatted model looks amateur and obscures bugs. A formatted model with bad numbers is worse. Both passes are mandatory.

## When to use

- Building a new Google Sheet model from scratch (DCFs, P&L projections, trackers, dashboards)
- User asks you to "clean up", "fix", "audit", "restructure", or "improve" an existing sheet
- Adding tabs, ranges, formulas, or formats via the Sheets MCP
- Converting messy wide-format data to tidy long-format
- User asks for colour conventions, formatting, named ranges, data validation, or error checks

## Audit-first workflow (MANDATORY for any existing sheet)

When the user points you at an existing spreadsheet, **do not write anything** before completing steps 1–4:

1. **Structure scan** — `get_spreadsheet_metadata` → tabs, named ranges, protected ranges
2. **Content scan** — for each non-trivial tab, `read_range` with `value_render: "FORMULA"` on ~30 rows to see how formulas are built
3. **Anti-pattern scan** — `search_in_spreadsheet` with `use_regex: true` for each of:
   - `TODAY\(|NOW\(|RAND\(|RANDBETWEEN\(` (volatile)
   - `INDIRECT\(|OFFSET\(` (break dependency tracking)
   - `IMPORTRANGE\(|IMPORTDATA\(|IMPORTHTML\(|IMPORTXML\(` (external HTTP per refresh)
   - `VLOOKUP\(|HLOOKUP\(` without `IFERROR`/`IFNA` wrapper
   - Magic numbers in formulas: `\*\s*0?\.\d+`, `[+\-*/]\s*[0-9]+\.[0-9]+`, `[+\-*/]\s*[0-9]{2,}`
4. **Findings report** — present to user grouped (a) quick wins, (b) structural, (c) risky. **Stop. Wait for approval before editing.**

Only proceed to execution after user says "go" or equivalent.

## Audit trail — `enable_claude_log` + `log_session_summary` (MANDATORY)

**Before the first write tool** on a spreadsheet, call `enable_claude_log` (idempotent; returns `already_enabled: true` if on). This creates the `Claude_Log` tab with the session-summary schema.

**Before ending the chat thread** (your final response to the user), call `log_session_summary` exactly once with:
- `user_request` — a one-line paraphrase of what the user asked you to do
- `actions_summary` — a short bullet-style list of the substantive actions you took (tool calls bundled, not each one separately)
- `output_summary` — a one-line statement of what was delivered (tabs created / ranges written / checks passing)

Exactly one summary row per chat thread. Call it even if the work failed — record the failure in `output_summary`.

## Three-pass build workflow

### Pass 1 — Content

1. `add_sheet` for each FAST tab in order (see Workbook structure below)
2. Write the `About` tab content first — title, owner, purpose, tab index, live check-status cell `=COUNTIF(Checks!D:D, FALSE)`
3. Build `Inputs` tab; `set_named_range` for each scalar you'll reference elsewhere (`tax_rate`, `growth_rate`, `err_tol = 0.0001`, `today_ref`, `sign_convention_memo`)
4. Build `Data`/`Calcs`/`Outputs` tabs with formulas
5. Build `Checks` tab — one row per invariant

### Pass 2 — Formatting phase (not "polish")

Run this checklist as a literal to-do. Skip nothing:

- [ ] Font colour per role (blue input / black formula / green cross-sheet / red external)
- [ ] Number format applied per data class (see table below) — every currency cell, every %, every count
- [ ] One unit per tab — raw £ OR £m, never both. Mixed-unit tabs fail audit.
- [ ] Column widths set via `set_column_width` (pad 20 / label 260 / sub-label 180 / year 95 / note 160)
- [ ] Frozen panes via `set_frozen` — any tab with >10 data rows freezes the header row and label column (standard P&L: `{rows: 3, cols: 3}`)
- [ ] Title row bold 14pt
- [ ] Main / section header rows: bold, **black fill `#000000` with white text `#FFFFFF`**, top + bottom border
- [ ] Sub-header rows (within a section): bold, grey fill `#D9D9D9`, black text, bottom border only
- [ ] Totals rows: bold, solid top border
- [ ] Conditional format on any TRUE/FALSE column (green TRUE / red FALSE) via `set_conditional_format`
- [ ] Borders for section separators — **never blank rows**
- [ ] **Border consistency across columns** — any border applied to a totals or section row must span every data column uniformly (e.g. `B:<last-year-col>`), not a partial sub-range. A totals row with a top border on B:D but nothing on E:L is a format bug.
- [ ] Data validation on critical Input cells — e.g. `tax_rate` between 0 and 1 — via `set_data_validation`
- [ ] Protection on `About` and `Checks` tabs with `warningOnly: true` via `set_sheet_protection`
- [ ] **Tab colour** on every tab via `set_tab_color` — see "Tab colour palette" for the role-to-colour mapping.

Prefer `format_ranges_batch` to apply many formats in one API call — faster and rate-limit-safer than 20 individual `format_range` calls.

### Pass 3 — Verification (see Post-write sanity gate below)

## Number format spec (MANDATORY — no auto-inference)

Apply explicit formats via `format_range.format.number_format.pattern`. Auto-inference is a format-leak risk: `0.918` displayed as `91.8%` is a classic bug.

| Data class | Pattern |
|---|---|
| Currency £ (raw) | `"£"#,##0;("£"#,##0)` |
| Currency £m | `"£"#,##0.0,,"m";("£"#,##0.0,,"m")` (use if typical values > £1m) |
| Currency £bn | `"£"#,##0.0,,,"bn";("£"#,##0.0,,,"bn")` (use if typical values > £1bn) |
| Percentage | `0.0%;(0.0%);-` |
| Multiple (valuation) | `0.0"x"` |
| Count / headcount | `#,##0;(#,##0);-` |
| Ratio | `0.00;(0.00);-` |
| Year | `0` (no comma) |
| Date | `yyyy-mm-dd` (data tabs) / `d mmm yyyy` (presentation) |

**One unit per tab.** A P&L tab in £m must not have one row in raw £.

## Column width + frozen panes defaults

| Column | Width (px) | Purpose |
|---|---|---|
| A (pad) | 20 | Visual margin |
| B (label) | 260 | Line item names |
| C (sub-label) | 180 | Unit / sub-category |
| D onward (year/period) | 95 | Numeric data |
| Notes / unit cols | 160 | End-of-row commentary |

Standard P&L / model tab: `set_frozen(rows: 3, cols: 3)`. Adjust if title uses 2 rows + header = `rows: 3`.

## Colour convention (classic WSO + assumption highlight — MANDATORY)

Font colour encodes cell role. Assumption cells additionally carry a yellow fill for at-a-glance identification. **Do not improvise.**

| Role | Font hex | Fill hex | Example |
|---|---|---|---|
| Hardcoded input / assumption (manual entry) | Blue `#0000FF` | Yellow `#FFF2CC` | A growth rate you type |
| Formula on the same sheet | Black `#000000` | none | `=B7*B8` |
| Cross-sheet link | Green `#006100` | none | `=Model!B7` |
| External / other-workbook link | Red `#C00000` | none | `=IMPORTRANGE(...)` |

Structural signals (on top of cell-role coding):
- **Totals** — bold
- **Section / main headings** — **black fill `#000000`, white text `#FFFFFF`**, bold
- **Sub-headers** — light-grey fill `#D9D9D9`, bold, black text
- **Negative numbers** — bracket notation `(1,234)` in red (via number format, not cell colour)
- **Section separators** — borders only, never blank rows. Borders must span every data column in the row uniformly (see "Border consistency" below).

Max 6–8 distinct colours across the whole model (yellow fill counts as one of those).

## Workbook structure (FAST)

Tab order, left to right. M&A / valuation tabs in parentheses — add when relevant:

| Tab | Purpose | Data shape |
|---|---|---|
| `About` | Purpose, owner, date, version, tab index, live check-status cell, **sign-convention memo** | n/a |
| `Inputs` | Every user-editable scalar in one place, labelled with units | wide |
| (`Target standalone` / `Buyer standalone`) | M&A only — separate standalone operating models | wide |
| (`Synergies`) | M&A only — revenue / cost split, ramp, flow-through %, integration cost | wide |
| `Data` | Raw / observational data | **long format** (tidy) |
| `Calcs` | Model logic — split by theme if large | wide |
| (`Pro Forma`) | M&A only — combined, post-synergy | wide |
| `Outputs` | Summary numbers, charts, (`football field` for valuation) | wide |
| `Checks` | Integrity checks — one row per check | wide |
| `Disclaimer` | Legal / caveats | n/a |
| `Claude_Log` | Audit trail (created by `enable_claude_log`) | auto |

`About!B5` displays `=COUNTIF(Checks!D:D, FALSE)` — non-zero at the top of the cover tab means something's broken.

## Tab colour palette (MANDATORY — role-coded)

Set each sheet's tab colour with `set_tab_color` so the tab bar at the bottom carries the same semantic signal as the rest of the format system. Tabs of the same role share a colour.

| Tab role | Tabs | Tab colour |
|---|---|---|
| Cover / narrative | `About`, `Disclaimer` | Light grey `#D9D9D9` |
| Assumptions | `Inputs`, `Assumptions Log` | Yellow `#FFF2CC` |
| Raw data | `Data` | Light blue `#CFE2F3` |
| Workings | `Calcs`, `Synergies`, `Target standalone`, `Buyer standalone` | Mid blue `#6FA8DC` |
| Answer / output | `Outputs`, `Pro Forma`, `Valuation`, `Football field` | Green `#93C47D` |
| Integrity | `Checks` | Orange `#F6B26B` |
| Audit trail | `Claude_Log` | Dark grey `#666666` |

Hidden utility / scratch tabs don't need a colour. Apply colours as part of Pass 2, immediately after tab creation.

For financial-modelling specifics (DCF templates, football-field layouts, pro-forma bridges, synergy structures, paid-vs-captured splits), see the companion skill `financial-modelling-extras`.

## Tidy data rules (MANDATORY for `Data` tabs)

Wickham's three rules:
1. Each variable has its own column
2. Each observation has its own row
3. Each cell has one value

**Long format for time series** — never `2023 | 2024 | 2025` as columns. Use a `year` (or `period`) column. `QUERY` / `FILTER` / pivot tables all assume long format.

**Never** in a data zone: merged cells, blank separator rows, subtotals mixed with raw rows. Use the `Outputs` tab for subtotals.

## Formula construction rules

- **Consistency across rows** — if `K10` is `=K9*(1+K$5)`, every cell `L10:Z10` uses the same structure. FAST calls this "corkscrew consistency"; #1 auditor checkpoint.
- **One logical operation per cell** — break complex `IF(AND(...))` into helper columns
- **Named ranges for named constants** — `tax_rate`, `growth_rate`, `err_tol`, `today_ref`. NOT for ranges that grow in shape.
- **Absolute refs for constants** — `$B$7`
- **Closed ranges over open** — `SUM(A1:A10000)` not `SUM(A:A)`

### Magic numbers → named inputs (lint before declaring done)

After the build, re-run the anti-pattern scan (same regex set from audit step 3) to confirm **no magic numbers** remain in formulas. Any literal number inside a formula (`=A1*0.20`) must be extracted to `Inputs` and referenced by a named range (`=A1*tax_rate`), OR explicitly justified via `set_cell_note` explaining why the literal is unavoidable.

## Volatile functions — avoid

| Function | Why | Fix |
|---|---|---|
| `TODAY()`, `NOW()` | Cascade to every dependent on every edit | One `=TODAY()` in `Inputs!B1`, named range `today_ref`, reference everywhere |
| `RAND()`, `RANDBETWEEN()` | Same | Copy-paste-values once; don't live-recalc |
| `INDIRECT()`, `OFFSET()` | Break dependency analysis | Use direct refs; if dynamic, document with `set_cell_note` |
| `IMPORTRANGE()` et al. | External HTTP per refresh; rate-limit hazard | Snapshot pattern — copy values on a schedule |

## Pre-write discipline (MANDATORY)

Before any `write_range` / `write_ranges_batch` call:

1. **Declare the range** explicitly: e.g. `Inputs!B5:B12`
2. **Declare the shape**: "values is 8×1, matches range (rows=8, cols=1)"
3. **Assert match** — if shape and range don't match, **stop**. The MCP will return `ROW_COUNT_MISMATCH` with a `suggested_range` — use it to retry rather than guessing.
4. **Maintain a ref-map** at the top of the turn — list every named-range target:
   ```
   Inputs!C5  → tax_rate
   Inputs!C13 → listings_base
   Inputs!C20 → growth_rate
   ```
   Row drift that silently breaks downstream refs is the biggest time-sink bug; the ref-map makes it Ctrl-F-catchable across the turn.
5. **Use `safe_mode: true`** on writes you're not certain are adding to empty space — it blocks overwrites of non-empty cells with an `OVERWRITE_BLOCKED` error and a preview of conflicts. Bypass only with `skip_overwrite_check: true` when the overwrite is intentional.
6. **Re-read every input cell you wrote** with `value_render: "UNFORMATTED"` to catch format-inference bugs (`'1.5'` string vs `1.5` number). Use `return_computed: true` on the write itself to get a formatted-value preview without a second call.

## Sign-convention assertion

Every model declares its sign convention up front:

- `About!A10` (or `Inputs!A1`) — memo cell: `"Sign convention: costs are negative (expenses reduce P&L rows). Revenue is positive."`
- `Checks` tab row: pick a representative cost cell and assert `=IF(<cost_cell> < 0, TRUE, FALSE)` — catches accidental positive-costs bugs.

Without this, a 50-row P&L can quietly flip sign half-way down and nobody notices until the bridge breaks.

## Protection policy

- `Inputs` tab — **no protection** (user edits here)
- All formula tabs — **no hard protection** (Claude needs to write here)
- `About` and `Checks` tabs — **`warningOnly: true` via `set_sheet_protection`** (users see a warning on edit but can proceed; stops accidental overwrites of the artefacts executives read first)
- `Claude_Log` tab — **do not protect** (logging middleware appends)

## Integrity checks

Every non-trivial model has a `Checks` tab. One row per invariant:

| Check | Expected | Actual | Pass? |
|---|---|---|---|
| Balance sheet | 0 | `=Model!B30-Model!C30-Model!D30` | `=ABS(C2)<err_tol` |
| Raw vs summary total | 0 | `=SUM(Data!E:E)-Outputs!B5` | `=ABS(C3)<err_tol` |
| Cost-sign convention | TRUE | `=Calcs!F10<0` | `=D4` |

- Define `err_tol = 0.0001` as a named range (kills floating-point noise)
- Do NOT use bare `IFERROR(..., "")`. Use `=IF(ISERROR(XLOOKUP(...)), "MISSING", XLOOKUP(...))` AND count `"MISSING"` in `Checks`.
- The `Pass?` column feeds `About!B5`.

## Post-write sanity gate (MANDATORY before declaring done)

Before telling the user "done", re-read the key output cells with `return_computed: true` (on the write) or `read_range` + `value_render: "FORMATTED"`. Assert all four:

### 1. Order of magnitude (sector-calibrated)

| Metric | Plausible band (flag outside) |
|---|---|
| EV / Revenue multiple | 0.5x – 20x (marketplace) |
| EBITDA margin | -30% to 60% (tech marketplace) |
| YoY revenue growth | -50% to +200% |
| Synergy £ as % combined revenue | 0% to 20% |
| Terminal growth rate | 1% to 3% (real-world nominal) |
| WACC | 6% to 15% (marketplace equity) |
| DCF implied EV on a £10m–£100m business | £10m – £1bn |

Outside these bands = investigate before claiming done. Do NOT silently proceed.

### 2. Directional sign

- Synergy accretion > 0
- Integration cost < 0
- Growth in growth year > base year (on growing businesses)
- EBITDA > 0 on profitable target

### 3. Bridge ties

Any bridge-check cell on `Checks` must return `|actual| < err_tol`. If any fail, name the failing checks and stop.

### 4. No magic numbers remaining

Re-run the magic-number regex scan. Any hits = extract to `Inputs` or justify with `set_cell_note`.

### 5. Border consistency (format lint)

For every totals row and section header you formatted, re-read the range with `read_range` and confirm that every column in the data range carries the same border style as column B. A row where borders stop mid-row (e.g. B:D bordered, E:L blank) is a format bug — extend the border to the full data width before declaring done.

## Google Sheets limits

| Limit | Value |
|---|---|
| Cells per spreadsheet | 10,000,000 (hard) |
| Cells per tab | 5,000,000 (hard) |
| Practical slowdown threshold | ~1,200,000 cells in active use |
| Columns max | 18,278 (ZZZ) |
| MCP write payload | 2 MB |
| Google API quota | 60 reads/min, 60 writes/min per user |

Near these limits? Shard across spreadsheets and use **snapshot** `IMPORTRANGE`.

## MCP tool mapping (v2)

| Task | MCP tool(s) |
|---|---|
| Audit structure | `get_spreadsheet_metadata`, `list_named_ranges`, `get_charts_and_pivots` |
| Audit formulas | `read_range` with `value_render: "FORMULA"` |
| Hunt anti-patterns | `search_in_spreadsheet` with `use_regex: true` |
| Trace cell dependencies | `analyse_formula_dependencies` |
| Write with shape guard | `write_range` / `write_ranges_batch` (handles `ROW_COUNT_MISMATCH`, `safe_mode`, `return_computed`) |
| Bulk formatting | `format_ranges_batch` (one call for many zones) |
| Number format / colour / borders | `format_range` / `format_ranges_batch` |
| Conditional format | `set_conditional_format` |
| Merge header cells | `merge_cells` |
| Column widths | `set_column_width` / `auto_resize_columns` |
| Row heights | `set_row_height` |
| Frozen panes | `set_frozen` |
| Move tabs to correct order | `move_sheet` |
| Data validation on inputs | `set_data_validation` |
| Sheet protection (warn-only) | `set_sheet_protection` |
| Define constants | `set_named_range` |
| Document exceptions | `set_cell_note` |
| Enable audit trail | `enable_claude_log` (mandatory before any write) |
| Tab colours | `set_tab_color` |
| Write the session summary to Claude_Log | `log_session_summary` (call once at end of thread) |
| Post-write sanity read | `read_range` with `value_render: "FORMATTED"` or `return_computed: true` on the write itself |

## Red flags — STOP and re-plan

| You're about to... | Fix |
|---|---|
| Write to an existing sheet without first doing an audit report | Run the 4-step audit, report, wait for approval |
| Call any write tool without first calling `enable_claude_log` | Stop. Call `enable_claude_log`. Non-negotiable. |
| End the chat thread without calling `log_session_summary` | Non-negotiable. One summary row per thread — otherwise the audit trail is silent. |
| Declare done without running the post-write sanity gate | Stop. Re-read outputs, assert the four checks, then declare. |
| Skip the formatting phase and move to verify | Formatting is not polish. It's Pass 2. |
| Use yellow fill WITHOUT blue font on an assumption | Yellow fill is additive — blue font still encodes role. Apply both. |
| Use blue font WITHOUT yellow fill on an assumption | Yellow fill `#FFF2CC` is mandatory on every hardcoded input. Apply both. |
| Main header row not black-fill white-text | Main headers use `#000000` fill + `#FFFFFF` text. Grey `#D9D9D9` is for sub-headers only. |
| Borders that stop mid-row on a totals or section line | Borders must span every data column (B:last-year-col). Extend to full width. |
| Ship a workbook without role-coded tab colours | Every tab gets a colour from the palette in "Tab colour palette". Call `set_tab_color`. |
| Put a magic number in a formula | Extract to `Inputs` as a named range |
| Merge cells inside a data zone | No. Borders + bold for visual grouping |
| Use `IMPORTRANGE` for live data | Snapshot pattern instead |
| Wrap a VLOOKUP in bare `IFERROR(..., "")` | Surface: `IF(ISERROR(...), "MISSING", ...)` and add to `Checks` |
| Use `=TODAY()` in more than one cell | One in `Inputs`, reference via named range |
| Build a wide layout for time-series data | Pivot to long format with a `period` column |
| Ship a model without column widths or frozen panes | Mandatory in Pass 2. Call `set_column_width` + `set_frozen`. |
| Mix units on one tab (raw £ and £m both present) | Pick one per tab. Mixed-unit tabs fail audit. |
| Proceed past a `ROW_COUNT_MISMATCH` error | Retry with the `suggested_range` from the error — don't guess |

## Common rationalisations

| Excuse | Reality |
|---|---|
| "Formatting is polish, I'll do it last" | Formatting is Pass 2 of a 3-pass build. An unformatted model looks amateur and obscures bugs. Do it. |
| "Number format auto-inferred, should be fine" | Auto-inference is format-leak risk. `0.918` displayed as `91.8%` is a classic. Always explicit. |
| "Blank rows are fine as separators" | No. Borders. If you hit a blank-row separator during audit, treat as a structural finding. |
| "I'll set column widths in the UI after" | LLMs don't touch UIs. Ship presentable or don't ship — the MCP has `set_column_width`, use it. |
| "The valuation mids average out even with mixed methods" | Mid of 4 methods with one bad method moves ~25%. Audit EACH method before averaging. |
| "Read-back is optional if the bridge passes" | Bridge ties don't catch methodology errors. Always read `FORMATTED` on output cells before declaring done. |
| "Enabling Claude_Log for a one-line change is overkill" | Accountability value is highest on small changes. Call it. |
| "Logging is already on for this spreadsheet, I can skip the call" | You don't know that. Call `enable_claude_log` anyway — it's idempotent. |
| "I'll enable logging after I make the change" | Backwards. Enable before the first write. |
| "I already called log_session_summary mid-thread" | Call it again at the end — the last call is the canonical one. Only one row per thread. |
| "`safe_mode` is unnecessary for a small append" | Small appends are where silent overwrites hide. Use `safe_mode` unless you can *prove* the target is empty. |
| "The existing sheet is too messy to audit fully, I'll just fix one thing" | Audit anyway. One targeted fix without context usually breaks something else. |
| "The user asked me to just do X, not audit" | If X touches structure or formulas, the audit protects them from a silent regression. Do it and flag. |
| "I got a `ROW_COUNT_MISMATCH`, I'll just force-write" | No — use the `suggested_range` from the error and retry. |
| "Sector bounds are approximate, I'll skip the sanity gate" | Approximate bounds catch an EV=£9bn result on a £30m business. That's the whole point. Run the gate. |
