---
name: google-sheets-modelling
description: Use when building, auditing, restructuring, or formatting any Google Sheet — financial models, trackers, dashboards, raw-data logs, or forecast models. Covers the FAST standard, tidy data, the classic blue/black/green/red colour convention, formula hygiene, and Google Sheets performance gotchas. Applies especially when using the Google Sheets MCP tools (write_range, format_range, add_sheet, etc.) or when a user asks Claude to "clean up", "fix", "model", or "restructure" a spreadsheet.
---

# Google Sheets modelling

## Core principle

Apply the **FAST Standard** (Flexible, Appropriate, Structured, Transparent) and **tidy data** to every sheet. **Audit before you write**: read the current structure, produce a findings report, and wait for explicit user approval before changing anything on an existing sheet.

Violating the audit-first rule is violating the spirit of the rule.

## When to use

- Building a new Google Sheet model from scratch
- User asks you to "clean up", "fix", "audit", "restructure", or "improve" an existing sheet
- Adding a tab, range, formula, or format via the Sheets MCP
- Converting messy wide-format data into tidy long-format
- User asks for colour conventions, formatting, named ranges, or error checks

## Audit-first workflow (MANDATORY for any existing sheet)

When the user points you at an existing spreadsheet, **do not write anything** before completing steps 1–4:

1. **Structure scan** — `get_spreadsheet_metadata` → tabs, named ranges, protected ranges
2. **Content scan** — for each non-trivial tab, `read_range` with `value_render: "FORMULA"` on ~30 rows to see how formulas are built
3. **Anti-pattern scan** — `search_in_spreadsheet` with `use_regex: true` for each of:
   - `TODAY\(|NOW\(|RAND\(|RANDBETWEEN\(` (volatile)
   - `INDIRECT\(|OFFSET\(` (break dependency tracking)
   - `IMPORTRANGE\(|IMPORTDATA\(|IMPORTHTML\(|IMPORTXML\(` (external HTTP per refresh)
   - `VLOOKUP\(|HLOOKUP\(` without an `IFERROR`/`IFNA` wrapper
4. **Findings report** — present to user grouped as (a) quick wins, (b) structural, (c) risky. **Stop. Wait for approval before editing.**

Only proceed to execution after user says "go" or equivalent.

## Audit trail — `enable_claude_log` is MANDATORY

**Before ANY `write_range`, `write_ranges_batch`, `append_rows`, `clear_range`, `format_range`, `add_sheet`, `delete_sheet`, `duplicate_sheet`, `insert_dimension`, `delete_dimension`, `set_named_range`, or `set_cell_note` call on a spreadsheet**, call `enable_claude_log` on that spreadsheet first.

This is **non-negotiable**. The `Claude_Log` tab is the user's accountability trail — who did what, when, and with what inputs. Skipping it means the user can't audit your changes after the fact.

Rules:
- Call `enable_claude_log` at the very start of the execution phase (right after user approval) — before the first material write
- The tool is idempotent — if logging is already enabled it returns `already_enabled: true`, so calling it defensively costs nothing
- Do NOT treat it as optional for "small" or "quick" edits — small edits are exactly where silent regressions hide
- If the tool errors (e.g. permission denied), STOP and tell the user rather than proceeding without logging

Read-only audit steps (1–4 above) do NOT require logging. Logging is required only for writes.

## Workbook structure (FAST)

Every non-trivial model has these tabs in this order:

| Tab | Purpose | Data shape |
|---|---|---|
| `About` | Purpose, owner, date, version, tab index, live check-status cell | n/a |
| `Inputs` | Every user-editable scalar in one place, labelled with units | wide |
| `Data` | Raw / observational data | **long format** (tidy) |
| `Calcs` | Model logic — split by theme if large | wide |
| `Outputs` | Summary numbers, charts | wide |
| `Checks` | Integrity checks — one row per check | wide |

`About!B5` shows `=COUNTIF(Checks!D:D, FALSE)` so a non-zero at the top of the cover tab means something's broken.

## Colour convention (classic WSO — MANDATORY)

Font colour encodes cell role. **This is the convention. Do not improvise.**

| Role | Font hex | Example |
|---|---|---|
| Hardcoded input (manual entry) | Blue `#0000FF` | A growth rate you type in |
| Formula on the same sheet | Black `#000000` | `=B7*B8` |
| Cross-sheet link | Green `#006100` | `=Model!B7` |
| External / other-workbook link | Red `#C00000` | `=IMPORTRANGE(...)` or `=[X.xlsx]Y!A1` |

Additional layer (structural signals only, not replacements for the font-colour code):

- **Totals** → bold
- **Headers** → light-grey fill `#D9D9D9`, bold, black text
- **Negative numbers** → bracket notation `(1,234)`, red
- **Section separators** → borders, never blank rows

Max 6–8 distinct colours across the whole model. Beyond that readers stop parsing them.

### How to apply via MCP

```
format_range(
  range: "Inputs!B2:B20",
  format: {
    text_format: { foreground_color: "#0000FF" }
  }
)
```

Apply in one `format_range` call per colour-zone rather than per-cell.

## Tidy data rules (MANDATORY for `Data` tabs)

Hadley Wickham's three rules:

1. Each variable has its own column
2. Each observation has its own row
3. Each cell has one value

**Long format for time series** — never `2023 | 2024 | 2025` as columns. Use a `year` (or `period`) column. `QUERY` / `FILTER` / pivot tables all assume long format.

**Never** in a data zone: merged cells, blank separator rows, subtotals mixed with raw rows. Use the `Outputs` tab for subtotals.

## Formula construction rules

- **Consistency across rows** — if `K10` is `=K9*(1+K$5)`, every cell `L10:Z10` uses the same structure. FAST calls this "corkscrew consistency" and it's the #1 auditor checkpoint.
- **One logical operation per cell** — break complex `IF(AND(...))` into helper columns
- **Named ranges for named constants** — `tax_rate`, `growth_rate`, `err_tol`, `today_ref`. NOT for ranges that grow in shape.
- **Absolute refs for constants** — `$B$7`
- **Closed ranges over open** — `SUM(A1:A10000)` not `SUM(A:A)`. Open ranges are slower and unsafe.

### Magic numbers → named inputs

Any literal number inside a formula (e.g. `=A1*0.20`) must be extracted to `Inputs` and referenced by named range. `=A1*tax_rate` is the correct form.

## Volatile functions — avoid

| Function | Why | Fix |
|---|---|---|
| `TODAY()`, `NOW()` | Cascade to every dependent on every edit | One `=TODAY()` in `Inputs!B1`, named range `today_ref`, reference everywhere |
| `RAND()`, `RANDBETWEEN()` | Same as above | Copy-paste-values once; don't live-recalc |
| `INDIRECT()`, `OFFSET()` | Break dependency analysis, harder to audit | Use direct refs; if truly dynamic, document with `set_cell_note` |
| `IMPORTRANGE()` et al. | External HTTP per refresh; rate-limit hazard | Snapshot pattern — copy values on a schedule |

## Integrity checks

Every non-trivial model has a `Checks` tab. One row per invariant:

| Check | Expected | Actual | Pass? |
|---|---|---|---|
| Balance sheet | 0 | `=Model!B30-Model!C30-Model!D30` | `=ABS(C2)<err_tol` |
| Raw vs summary total | 0 | `=SUM(Data!E:E)-Outputs!B5` | `=ABS(C3)<err_tol` |

- Define `err_tol = 0.0001` as a named range — this kills floating-point noise in equality checks
- Wrap each `VLOOKUP` / `XLOOKUP` with a flag: `=IF(ISERROR(XLOOKUP(...)), "MISSING", XLOOKUP(...))`. Do NOT use a bare `IFERROR(..., "")` — silent failures are the worst failures.
- The `Checks` tab's `Pass?` column is what `About!B5` counts.

## Google Sheets limits

| Limit | Value |
|---|---|
| Cells per spreadsheet | 10,000,000 (hard) |
| Cells per tab | 5,000,000 (hard) |
| Practical slowdown threshold | ~1,200,000 cells in active use |
| Columns max | 18,278 (ZZZ) |
| MCP write payload | 2 MB |
| Google API quota | 60 reads/min, 60 writes/min per user |

Near these limits? Shard across spreadsheets and use **snapshot** `IMPORTRANGE` (refresh on a schedule, not live).

## MCP tool mapping

| Task | MCP tool | Notes |
|---|---|---|
| Audit structure | `get_spreadsheet_metadata`, `list_named_ranges`, `get_charts_and_pivots` | Step 1 |
| Audit formulas | `read_range` with `value_render: "FORMULA"` | Step 2 |
| Hunt anti-patterns | `search_in_spreadsheet` with `use_regex: true` | Step 3 |
| Trace cell dependencies | `analyse_formula_dependencies` | Deep-dive |
| Apply colour convention | `format_range` with `text_format.foreground_color` | Batch per zone |
| Define constants | `set_named_range` | For inputs + tolerances |
| Document exceptions | `set_cell_note` | Only for non-obvious choices |
| Enable audit trail | `enable_claude_log` | Always on for material edits |

Before any material write, call `enable_claude_log` if it's not already on, so every tool call lands in an audit row.

## Red flags — STOP and re-plan

| You're about to... | Fix |
|---|---|
| Write to an existing sheet without first doing an audit report | Run the 4-step audit, report, wait for approval |
| Call any write tool without first calling `enable_claude_log` | Stop. Call `enable_claude_log` on this spreadsheet_id first. Every single write tool is gated on this. |
| Use yellow fill for inputs instead of blue font | Classic WSO convention is font colour, not fill |
| Put a magic number in a formula | Extract to `Inputs` as a named range |
| Merge cells inside a data zone | No. Use borders + bold for visual grouping |
| Use `IMPORTRANGE` for live data | Use a snapshot pattern instead |
| Wrap a VLOOKUP in bare `IFERROR(..., "")` | Surface the error: `IF(ISERROR(...), "MISSING", ...)` and add to `Checks` |
| Use `=TODAY()` in more than one cell | One in `Inputs`, reference via named range |
| Build a wide layout for time-series data | Pivot to long format with a `period` column |

## Common rationalisations

| Excuse | Reality |
|---|---|
| "The existing sheet is too messy to audit fully, I'll just fix one thing" | Start with the audit anyway. One targeted fix without context usually breaks something else. |
| "The user asked me to just do X, not audit" | If X touches structure or formulas, the audit protects them from a silent regression. Do it and flag. |
| "Enabling Claude_Log for a one-line change is overkill" | The accountability value of logging is highest on small changes, because they're the ones that slip through review. Call it. |
| "Logging is already on for this spreadsheet, I can skip the call" | You don't know that for certain. Call `enable_claude_log` anyway — it returns `already_enabled: true` if it's on. Zero-cost idempotency. |
| "I'll enable logging after I make the change so it starts from a clean slate" | Backwards — the change itself is what needs logging. Enable BEFORE the first write. |
| "Yellow fill for inputs is what my other spreadsheets use" | Classic WSO is font colour. Fills are for headers. Be consistent with the skill, not with other sheets. |
| "Pivoting to long format is too much work" | Do the pivot. Every downstream tool (`QUERY`, pivot tables, charts) gets better. |
| "One more `TODAY()` won't hurt" | Every added volatile function cascades through every dependent. One is the limit. |
| "It's just a scratch sheet, FAST doesn't apply" | If it's scratch, mark it `Scratch_` in the tab name and skip formalism. If it's being built on, FAST applies. |
