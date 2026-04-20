---
name: financial-modelling-extras
description: Use when building or auditing DCFs, valuation models, M&A pro formas, synergy cases, football-field valuations, or any model that produces an EV / equity value / multiple. Covers DCF structure, terminal-value conventions, football-field layout, pro-forma bridge checks, revenue-vs-cost synergy split with flow-through %, paid-vs-captured synergy memo, and sign-convention assertions. Pairs with google-sheets-modelling — inherits its FAST / colour / audit-first / verification-gate rules. Also covers tab ordering for Google Sheets delivery, entity P&L section structure, Assumptions Log, Pro Forma MEMO section, and Comparables table.
---

# Financial modelling extras

## MCP connection check (MANDATORY — do this first)

Before any other action, call `ping_sheets`. If it fails or returns an error, **stop immediately** and tell the user:

> "The Google Sheets MCP is not connected. Please check your MCP server is running and try again."

Do not proceed with any spreadsheet work until `ping_sheets` succeeds.

## Core principle

Inherits every rule from `google-sheets-modelling`. Adds structure specific to **valuation** and **M&A** models, where the cost of a small methodology error is large (£100m+ EV swings are routine).

**Two rules unique to this domain:**
1. **What the seller gets ≠ what the buyer captures.** Valuation prices what the seller receives. Synergy PV retained by the buyer is shown separately as a memo — never folded into EV.
2. **Every method in a football field gets audited individually before any average is shown.** The mid of 4 methods with one bad method is ~25% off.

## When to use

- Building a DCF (5–10 year projection + terminal)
- Building or auditing a valuation (trading comps, transaction comps, precedents, DCF, LBO)
- Producing a football-field chart or table
- Building an M&A pro-forma model with standalone targets + buyer
- Modelling synergies (revenue, cost, ramp, flow-through, integration cost)
- Auditing anything that ends in `£Xm EV`, `£Xm equity`, `Xx EV/Rev`, `Xx EV/EBITDA`

## DCF template

Structure (explicit — no improvisation):

### Inputs tab additions

| Named range | Typical | Notes |
|---|---|---|
| `wacc` | 9.0%–12.0% | UK marketplace equity; sensitise ±200 bps |
| `terminal_growth` | 2.0%–2.5% | UK nominal GDP long-run; not higher |
| `forecast_years` | 5 | 5 default, 7–10 for long-duration assets |
| `tax_rate` | 25.0% | UK corporation tax |
| `valuation_date` | `=today_ref` | Anchor for discounting |

### Calcs tab — FCF build

Row-by-row, consistent across years:

```
Revenue
× (1 + growth_rate)
– COGS (as % of Revenue)
= Gross Profit
– Opex (as % of Revenue)
= EBITDA
– D&A
= EBIT
× (1 – tax_rate)
= NOPAT
+ D&A
– Capex
– Δ Working Capital
= Unlevered FCF
```

### Terminal value — explicit about convention

Use the **Gordon Growth** model by default:

```
Terminal Value at end of Year N = FCF_{N+1} / (wacc – terminal_growth)
where FCF_{N+1} = FCF_N × (1 + terminal_growth)
```

Then **discount back to today**, not to Year N:

```
PV(Terminal) = Terminal / (1 + wacc)^N
```

**Common mistake:** using `FCF_N` (not `FCF_{N+1}`) in the Gordon formula. It understates terminal by one year of growth. The `set_cell_note` on the terminal cell must say: *"Terminal uses Year N+1 FCF = FCF_N × (1 + terminal_growth). Discounted to today at (1+wacc)^N."*

### Exit-multiple cross-check

Always include an exit-multiple terminal alongside the Gordon terminal on `Checks`:

```
Exit multiple TV = EBITDA_{N} × exit_multiple
```

If Gordon TV and exit-multiple TV differ by >30%, something's wrong — flag before declaring done.

## Football-field layout

Mandate:

- **Max 4 methods** visible at a time. More = cognitive load, less = under-triangulated. Typical: Trading comps / Transaction comps / DCF (WACC sens) / DCF (multiple sens).
- Each method has explicit **low / mid / high** values and a `source` cell (e.g. `"WACC +/- 200 bps"`, `"Peers 25th–75th EV/Rev"`)
- Sort by **mid value**, descending, so the visual reads top-down by implied value
- Axis limits: round to nearest £5m for small deals, £25m for mid-market, £100m for large — explicit in a `Inputs!football_axis_step` named range
- **Do not show the average of the four mids.** The user can eye-average. Forced averaging encourages methodology blindness.

On `Checks`:
- `=MAX(all mids) - MIN(all mids) < Inputs!football_max_spread` — flags implausible triangulation spreads
- `=COUNTIF(sources, "")` = 0 — every band has a documented source

## M&A pro forma bridge

Non-negotiable bridge check on `Checks`:

```
Reported EBITDA (pro forma) == Target standalone EBITDA
                             + Buyer standalone EBITDA
                             + Revenue synergy EBITDA
                             + Cost synergy
                             – Integration cost
                             – Dis-synergies (if any)
```

Represent this as a `Checks` row: `=ABS(pro_forma_ebitda - sum_of_parts) < err_tol`.

If this fails, the model's unusable — STOP before presenting.

## Tab structure and ordering (Google Sheets delivery)

`move_sheet` must be called **sequentially** (one call per tab, not batched) because each move shifts sheet indices for subsequent tabs. Parallel calls will produce the wrong final order.

Canonical tab order for an M&A delivery workbook:

| Position | Tab | Notes |
|---|---|---|
| 0 | About | First tab a reviewer opens — must be clean |
| 1 | Inputs | All blue-font hardcoded assumptions |
| 2+ | [Buyer standalone] | One tab per entity |
| 3+ | [Target standalone] | One tab per entity |
| N-4 | Synergies | Revenue + cost ramp |
| N-3 | Pro Forma | Bridge + MEMO |
| N-2 | Valuation | Football field + Comparables |
| N-1 | Checks | All assertion rows |
| N | Claude_Log | Hidden from reviewer |

**Utility tabs** (scratch, raw data imports, intermediate calcs not shown to reviewer) must be hidden via `set_sheet_visibility` with `hidden: true` before delivery. Do not delete — the data may be needed later.

`About` is always position 0. The reviewer opens it first and it sets the tone. If `About` is buried, the model looks unfinished.

**Tab colours (applied via `set_tab_color`):** About → grey `#D9D9D9`, Inputs + Assumptions Log → yellow `#FFF2CC`, Buyer/Target standalone + Synergies + Calcs → mid blue `#6FA8DC`, Pro Forma + Valuation + Outputs → green `#93C47D`, Checks → orange `#F6B26B`, Claude_Log → dark grey `#666666`. See the "Tab colour palette" section in `google-sheets-modelling` for the full mapping.

## Entity standalone P&L — four-section structure

Each entity tab (Buyer, Target) follows four named sections, separated by a blank row:

| Section | Label | Font | Fill | Content |
|---|---|---|---|---|
| A | Revenue Drivers | Blue `#0000FF` | Yellow `#FFF2CC` | Hardcoded inputs: prices, volumes, growth rates. One row per driver. Row labels describe the logic inline (e.g. "UK GMV growth rate — management case"). |
| B | Cost Drivers | Blue `#0000FF` | Yellow `#FFF2CC` | Hardcoded inputs: margin %, overhead £. Same inline-label convention. |
| C | Revenue Build | Black | none | Formulas only — references Section A and `Inputs`. No hardcoded numbers. |
| D | P&L Summary | Black | none | Formulas through to EBITDA — references Sections A, B, C and `Inputs`. |

**Totals rows** in C and D get green background `#D9EAD3` + bold font — still the only green fill in the workbook, marking "the answer row" for each section. (Yellow on A and B marks inputs; green on C/D totals marks outputs — no overlap.)

**Rule:** Sections C and D must never contain hardcoded constants. If a number appears in a formula in C or D, it must trace back to Section A, Section B, or `Inputs`. Magic-number lint catches this — fix before declaring done.

## Assumptions Log tab (mandatory deliverable)

Built **after** all entity tabs are complete. One tab, named `Assumptions Log`. Purpose: give a reviewer a single place to interrogate every assumption without opening formulas.

**Column structure:**

| Col | Header | Width | Notes |
|---|---|---|---|
| A | # | 20 | Auto-number (ARRAYFORMULA or manual) |
| B | Assumption | 280 | Short label — matches the row label in source tab |
| C | Value / Range | 200 | The actual number or range used |
| D | Source & Rationale | 400 | Free text, WRAP on. Where the number came from + why. |

**Layout rules:**
- Group rows under bold section headers that match the source tab (e.g. **Buyer**, **Target**, **Synergies**, **Valuation**)
- Analyst assumptions (not sourced from management or data) are prefixed with `★` in column B
- Do not formula-link column C back to the model — this is a read-only narrative log, not a live feed. Update it when assumptions change.
- **Every data row** (not headers, not section dividers) gets yellow fill `#FFF2CC` across columns B:D — matches the in-tab assumption colour convention.
- Tab colour: yellow `#FFF2CC` via `set_tab_color` (same as the `Inputs` tab — both are assumption tabs).

**Checks row:**

```
Analyst assumptions flagged | >0 | =COUNTIF(B:B,"★*")>0
```

If COUNTIF returns 0, no assumptions have been flagged as analyst-owned — that's almost certainly wrong. STOP and review.

## Pro Forma MEMO section and Valuation Comparables table

### Pro Forma MEMO

Immediately below the EBITDA bridge on the `Pro Forma` tab, add a MEMO section:

```
MEMO: INCREMENTAL VALUE VS [BUYER] STANDALONE
Incremental Revenue       | £Xm  | =pro_forma_revenue - buyer_standalone_revenue
Incremental EBITDA        | £Xm  | =pro_forma_ebitda - buyer_standalone_ebitda
Implied Revenue CAGR (combined vs buyer standalone) | X% | formula
```

Header row: bold, no fill. Values: black font formula. This is a **memo** — it does not feed the football field or the EV calculation. It answers "what does the buyer gain vs doing nothing?"

### Static Comparables table

On the `Valuation` tab, below the football field, add a static reference table:

| Column | Width | Content |
|---|---|---|
| Company / Transaction | 220 | Name |
| EV / Revenue | 100 | Multiple (e.g. `3.2x`) |
| EV / EBITDA | 100 | Multiple |
| Context | 260 | Date, geography, deal type |
| Relevance | 200 | Why included — one sentence |

Label the block: `"Reference only — not formula-linked."` Use `set_cell_note` on the header to explain why it's static (avoids future confusion about why these don't update).

## Synergy structure (MANDATORY format)

### Revenue vs cost split

Synergies always split on `Synergies` tab:

| Category | Row | Notes |
|---|---|---|
| Revenue synergies | 3 sections: cross-sell uplift, pricing uplift, retention uplift | Units in £ revenue, not margin |
| Cost synergies | 3 sections: duplicative roles, procurement, real-estate | Below the gross margin line |

### Ramp row (every synergy line)

Every synergy line has a 5-year ramp as its own row, multiplied into the steady-state number:

```
Year 1: 25%
Year 2: 65%
Year 3: 100% (steady state)
Year 4: 100%
Year 5: 100%
```

Ramp lives as a named range `synergy_ramp` on `Inputs` so you can sensitise.

### Flow-through % (revenue synergies only)

Revenue synergies don't translate 1:1 to EBITDA — gross-margin drop-through applies. Mandate a named range `revenue_synergy_flowthrough = 80%` on `Inputs` and apply to every revenue-synergy row. Sensitise ±10pp.

### Integration cost as below-the-line

Integration costs are **one-time** and sit **below** reported EBITDA — same line as restructuring. Do NOT subtract from synergy steady-state in the synergy PV calc.

## Paid vs captured synergies (CRITICAL — don't double-count)

| Concept | Goes where |
|---|---|
| **Synergy PV captured by BUYER** | Memo row on `Outputs` and on `football field` as a separate band. **Never added to valuation EV.** |
| **Synergy PV priced into offer (paid away to seller)** | Only shown if modelling a competitive bidding scenario. Separate assumption `synergy_paid_away_%` on `Inputs`. |
| **Standalone seller valuation** | This is what the football field shows — it's what the seller *gets*. |

**Common mistake:** adding synergy PV to the DCF EV and showing that number to management. That's what the buyer captures, not what the standalone business is worth.

On `Checks`, assert: `=ABS(football_mid - standalone_ev_dcf) / standalone_ev_dcf < 0.20` — ensures the football field mids cluster near the standalone DCF, within 20%.

## Sign-convention assertion (from main skill, applied here)

On M&A models, costs, integration charges, and dis-synergies are **negative**. On `Checks`:

```
Cost sign check   | TRUE | =Calcs!F15<0  | =D{row}
Integration sign  | TRUE | =Calcs!H30<0  | =D{row}
Dis-synergy sign  | TRUE | =Calcs!J40<0  | =D{row}
```

A P&L that silently flips sign half-way down breaks the bridge. These assertions catch it.

## Order-of-magnitude bands (M&A-specific, complements main skill)

| Metric | Plausible band |
|---|---|
| EV / NTM Revenue (marketplace) | 1x – 10x |
| EV / NTM EBITDA (marketplace) | 8x – 25x |
| Synergy PV as % deal EV | 5% – 40% |
| Integration cost as % Year 1 synergies | 50% – 200% |
| Pre-tax synergies as % combined revenue | 2% – 10% (most deals); 10%–20% only where explicit overlap justifies |

Outside these = investigate. A model claiming synergy PV = 80% of deal EV is wrong or the deal is extraordinary — flag it either way.

## Required Checks rows for M&A models

In addition to the main skill's `Checks`:

| Check | Expected |
|---|---|
| Pro-forma EBITDA bridge | 0 (sum of parts ties) |
| Gordon TV vs exit-multiple TV spread | < 30% |
| Football field mids cluster vs standalone DCF | within 20% |
| Every synergy line has a ramp row | no bare year-1 = 100% |
| Revenue synergy flow-through applied | `=every rev synergy row has *flowthrough` |
| Cost sign convention | costs negative |
| Integration cost sign | negative |
| No double-count of synergy in EV | standalone_ev == dcf_ev (synergy memo separate) |
| All implied EV values > £10M (catches unit errors e.g. margin % used instead of £ value in multiple calc) | TRUE for all methods |
| Blended valuation Low < Mid < High (catches copy-paste formula errors in blend) | TRUE |

## Red flags — STOP

| You're about to... | Fix |
|---|---|
| Fold synergy PV into the DCF EV and present that as "the valuation" | No. Standalone EV is the valuation. Synergy PV is a buyer-side memo. |
| Show the average of 4 football-field mids as "our number" | No. Don't collapse the range — the range IS the answer. |
| Use `FCF_N` instead of `FCF_N × (1 + terminal_growth)` in Gordon TV | Correct to `FCF_{N+1}` and add a `set_cell_note` |
| Apply 100% flow-through to revenue synergies | No. Default 80%, sensitise. Note it in `Inputs`. |
| Include integration cost *inside* synergy steady-state | No. Below the line, one-time |
| Publish without the pro-forma bridge on `Checks` | Build it. If the bridge fails, the model's unusable. |
| Skip ramp on synergies — show Year 1 = 100% | No. 3-year ramp (25/65/100) unless you have evidence otherwise. |

## Common rationalisations

| Excuse | Reality |
|---|---|
| "The buyer-captured synergy is part of the deal value, so include it in EV" | No. EV is what the standalone business is worth; synergy PV accrues to the buyer after close. Double-counting here has lost real deals. |
| "Gordon and exit-multiple TV should agree exactly" | No, but >30% spread means one of them is wrong. Reconcile before declaring done. |
| "A football field with 6 methods is more rigorous" | No, it's noisier. Pick your best 4 with clear source-of-spread. |
| "Revenue synergies are 100% flow-through" | No. Gross-margin drop-through applies. 80% default; sensitise. |
| "Integration costs are small, don't need a separate line" | No. They're always large-enough to move the synergy NPV by 20–40%. Line-item. |
| "Terminal growth can be 4% because we're high-growth" | No. Terminal growth = long-run economy. 2–3% in the UK. If you need higher, extend the explicit forecast period, don't inflate terminal. |
