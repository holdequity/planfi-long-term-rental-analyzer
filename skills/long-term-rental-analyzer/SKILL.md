---
name: long-term-rental-analyzer
version: 1.0.0
description: Long-term-rental after-tax analyzer for buy-and-hold landlords by orchestrating the public planfi MCP. Use whenever someone wants the AFTER-TAX return on a long-term rental — e.g. "analyze a long-term rental", "buy-and-hold landlord after-tax return", "rental depreciation / recapture / 1031", "Schedule E P&L on a $500k rental", "what's my depreciation tax shelter", "how much tax do I owe when I sell my rental", or "should I do a 1031 exchange". The headline is the year-1 Schedule E P&L + depreciation shelter and the after-tax sale proceeds (with §1250 recapture, LTCG, and NIIT).
---

# long-term-rental-analyzer

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All math + financial logic live server-side. This skill only gathers inputs and calls the tools —
it does **not** compute anything locally, carries no business logic, math, or thresholds, and is
read-only (it never changes the user's data). The server is the source of truth.

This is the **long-term (buy-and-hold) rental** complement to **str-investment-analyzer** (nightly
Airbnb/VRBO short-term rentals) and the **after-tax** layer over `analyze_property_return` (which is
pre-tax). For an own-vs-rent decision on a primary residence, see **rent-vs-buy**.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_rental_property`):
- **`analyze_rental_property`** (PRIMARY) — the after-tax buy-and-hold rental engine: year-1
  Schedule E P&L + depreciation shelter and the full sale-event tax breakdown (§1250 recapture,
  LTCG, NIIT, or a 1031 deferral).
- optional: **`analyze_property_return`** (pre-tax IRR / cash-on-cash on a specific purchase),
  **`analyze_rent_vs_buy`** (own-vs-rent on a primary residence), **`generate_financial_plan`**
  (to derive household tax context — marginal rate, filing status, MAGI — and mint a `plan_id`).

Use whichever name your environment exposes (bare or `mcp__planfi__`-prefixed); below they are
written bare. If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — Gather inputs

Every input has a sensible server default, so `analyze_rental_property` runs cold (no inputs). Ask
only for what the user's question needs:

**Purchase + financing:** `purchase_price`, `land_value_pct` (excluded from depreciation),
`closing_costs_percent` (included in the depreciable basis), `capital_improvements`,
`down_payment_percent`, `mortgage_rate`, `mortgage_term_years`.

**Schedule E P&L (year 1):** `monthly_rent`, `vacancy_pct`, `annual_property_tax`,
`annual_insurance`, `annual_maintenance`, `hoa_annual`, `management_fee_pct` (% of EFFECTIVE,
post-vacancy rent).

**Sale / exit:** `hold_years` — **pass it for a real exit; OMIT it for an indefinite buy-and-hold**
and the sale tax becomes a clearly-labeled illustrative hypothetical. `annual_appreciation`,
`sale_price_override`, `selling_cost_pct`, `do_1031_exchange` (1031 like-kind exchange that DEFERS
the gain).

**Tax context (drives the shelter rate, LTCG stacking, and NIIT):** EITHER a `plan_id` (from
`generate_financial_plan` — plan-derived marginal rate / filing status / taxable income take
precedence), OR pass `filing_status`, `ordinary_taxable_income`, and `magi` directly. Optional
`state_flat_rate` layers a flat state tax on recapture + LTCG.

> **Engine facts to bake in (server-enforced — never restate as your own math):** all dollars are
> **today's (real) dollars**; all decimals are **fractions** (5% → `0.05`); depreciation is
> straight-line over **27.5 years on the building only** (land excluded); §1250 recapture is capped
> at **25%**; a 1031 exchange **defers** the gain, it doesn't eliminate it; tax brackets/limits are
> **~2026** values.

## Step 2 — Run the analysis

Call **`analyze_rental_property`** (PRIMARY). Pass `plan_id` when you have it so the server derives
tax context and can emit a `share_url`:

```
// FICTIONAL example — illustrate the call shape, not a real user's numbers
analyze_rental_property({
  purchase_price: 500000,
  land_value_pct: 0.20,
  closing_costs_percent: 0.03,
  monthly_rent: 3000,
  vacancy_pct: 0.05,
  annual_property_tax: 4000,
  annual_insurance: 1500,
  annual_maintenance: 2000,
  management_fee_pct: 0.08,
  down_payment_percent: 0.20,
  mortgage_rate: 0.065,
  mortgage_term_years: 30,
  hold_years: 10,
  annual_appreciation: 0.03,
  filing_status: "married_joint",
  ordinary_taxable_income: 200000,
  magi: 250000
})
// already have a plan? pass { plan_id } instead of the tax-context fields
// considering a tax-deferred exit? add: do_1031_exchange: true
```

**Route adjacent intents to the sibling tools:**
- "what's my pre-tax IRR / cash-on-cash on this purchase" → **`analyze_property_return`**.
- "is this an Airbnb / nightly short-term rental" → use the **str-investment-analyzer** skill
  (`analyze_str_property`).
- "should I buy a home to live in or keep renting" → use the **rent-vs-buy** skill.

## Step 3 — Present the results

- **Lead with the headline:** the year-1 `summary` — `effective_rent`, `schedule_e_taxable_income`
  (often a passive loss), `pre_tax_cash_flow`, and the `depreciation_shelter` value — then, when a
  sale is in scope, `after_tax_sale_proceeds` and `total_sale_tax`.
- **Schedule E P&L (`schedule_e`):** gross vs effective rent, `total_opex` + `management_fee`,
  `mortgage_interest_year1` vs `mortgage_principal_year1`, `annual_depreciation` +
  `depreciable_basis`, `taxable_income`, `pre_tax_cash_flow`, `depreciation_shelter_value`.
- **Hold horizon (`hold_horizon`):** report `mode` (`fixed` vs `indefinite`); when indefinite, relay
  the note that the sale tax is an illustrative "if you sold at year N" hypothetical, not advice.
- **Sale event (`sale`):** `sale_price`, `selling_costs`, `amount_realized`,
  `original_cost_basis` / `accumulated_depreciation` / `adjusted_basis`, `total_gain`,
  `remaining_mortgage_at_sale`, `after_tax_sale_proceeds`.
- **Tax at sale (`tax_at_sale`):** `unrecaptured_section_1250_gain` + `recapture_tax`,
  `long_term_capital_gain` + `capital_gains_tax`, `niit_tax`, `total_sale_tax`. If
  `do_1031_exchange` is true, all of those are zero and the entire `deferred_gain` carries over.
- **Surface `assumed_defaults[]`** — read back each `{ field, assumed_value, note }` so the user can
  correct any silent assumption (e.g. a missing `magi` means NIIT wasn't triggered; an omitted
  `hold_years` means the sale figures are a hypothetical). The server is the source of truth for what
  it defaulted — don't enumerate defaults from memory.
- **Disclosures:** relay `disclosures.assumptions` (land excluded from depreciation; closing costs
  in basis; 25% §1250 cap; §469 passive-loss limits NOT modeled so the shelter is an estimate; 1031
  defers, not eliminates; real dollars) and that this is a planning estimate, not financial advice.
- **`share_url`** — present only when returned (i.e. you passed a `plan_id`) so the user can open the
  full interactive plan on planfi.app.

## Next-actions hints

The tool returns a `next_actions[]` array (each `{ tool, why, prefilled_args }`, with
`prefilled_args` carrying `{ plan_id }` when available). Follow those server-defined edges rather
than guessing — they chain `analyze_rental_property` into the broader plan / related tools.

## Notes

- All decimals are fractions; all dollars are today's (real) dollars.
- Depreciation is straight-line over 27.5 years on the building only; the §1250 recapture cap is 25%;
  the depreciation shelter ignores §469 passive-activity limits (estimate only); a 1031 exchange
  defers — it does not eliminate — the gain. These are server facts surfaced via `disclosures`.
- Pass `plan_id` to derive household tax context (marginal rate / filing status / MAGI) and to get a
  `share_url`; the explicit `filing_status` + `ordinary_taxable_income` + `magi` are the fallback.
- **Related skills:** **str-investment-analyzer** (nightly STR / Airbnb), **rent-vs-buy** (own a
  primary residence vs rent), **financial-forecast** (fold the rental into the full FIRE timeline).
- Not financial advice. Planning estimates only (approximate ~2026 brackets/limits).
