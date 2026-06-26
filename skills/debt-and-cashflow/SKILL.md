---
name: debt-and-cashflow
version: 1.4.2
description: Pay down debt and route surplus cash optimally by orchestrating the public planfi MCP. Use whenever someone wants to know the order to pay off their debts (avalanche vs snowball), whether to consolidate high-rate revolving / credit-card debt into a personal loan or a 0% balance transfer (and whether the transfer fee is worth it), whether to prepay a mortgage or invest the difference, whether refinancing is worth it and when it breaks even, where the next best dollar of savings should go, their optimal student-loan path (which income-driven repayment plan, is PSLF forgiveness worth staying for, or refinance vs aggressive payoff), or how big their emergency fund / cash runway should be (and whether they're holding too much cash) — e.g. "what's my payoff order if I have a credit card and a car loan?", "should I consolidate my credit cards into a personal loan?", "is a 0% balance transfer worth the 3% fee?", "should I pay extra on my mortgage or invest it?", "is it worth refinancing and when do I break even?", "where should my extra $X/month go?", "which IDR plan should I be on?", "is PSLF worth staying for?", "should I refinance or pay off my student loans?", "how many months of emergency fund do I need?", "do I have too much cash sitting around?".
---

# Debt and Cashflow

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp/free).
All the math — amortization, avalanche/snowball ordering, prepay-vs-invest opportunity cost,
refinance break-even, and the next-best-dollar funding waterfall — lives server-side. This skill
only gathers inputs and calls the tools — it does **not** compute anything locally and bakes in no
defaults of its own. Read-only.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_debt_payoff`):
`analyze_debt_payoff`, `analyze_debt_consolidation`, `analyze_mortgage_prepay`, `analyze_refinance`, `analyze_funding_waterfall`,
`analyze_student_loans`, `analyze_emergency_fund`, `analyze_auto_purchase`, `analyze_cash_flow`, plus optional `generate_financial_plan` (for `plan_id` chaining + a `share_url`). Use whichever name
your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp/free.)

> **Access — free for personal use.** The planfi MCP is free to try (a small monthly allowance, no key needed). Heavy automated abuse forced us to add limits — but it stays **free for personal use**: email **kam@rateapi.dev** and we'll send you a free API key, no charge. (Companies and commercial use have paid plans.) To use a key, pass it as an `Authorization: Bearer pft_…` header in your MCP client config.


## Step 1 — (Optional but preferred) build a plan first to chain context + get a share link

> **Feed it into the forecast (not just plan_id chaining):** `generate_financial_plan` now accepts `student_loans` directly as a plan input, so it flows into net worth, FIRE %, and Monte-Carlo backtesting — the balance reduces net worth and the payment redirects into investing once paid off. Use the standalone analyze tool below for a focused what-if; pass `student_loans` into the plan to see its effect on the whole household forecast.

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. **All four** debt-and-cashflow tools accept `{ plan_id }` (plus
inline overrides), so they can resolve age, marginal tax rate, and surplus from the saved plan
instead of you re-sending every figure. `generate_financial_plan` also returns a **`share_url`**
(planfi.app). Two of the four tools below emit a `share_url` of their own — but only when called
with a `plan_id` that resolves to a plan with earners (`analyze_funding_waterfall` and
`analyze_student_loans`); `analyze_debt_payoff`, `analyze_mortgage_prepay`, and `analyze_refinance`
never emit one, so for those, run `generate_financial_plan` to give the user a share link.

This step is optional: every tool below runs from raw inputs too. Prefer the plan path when the
session already has a model, when the user is asking more than one of these questions, or when they
want a sharable artifact.

> **Engine facts to bake in:** all rates/decimals are **fractions** (22% APR → `0.22`); all dollars
> are **today's (real) dollars**; tax brackets/limits are approximate **~2026** values.

## Step 2 — Route by intent

Pick the tool that matches what the user is asking. Pass `{ plan_id }` when you have one; the raw
fields below are still REQUIRED where noted (per-debt and per-loan details can't come from a plan).

### "What order should I pay off my debts?" → `analyze_debt_payoff`
Compares avalanche (highest rate first) vs snowball (smallest balance first), returns the payoff
order, total interest paid, months to debt-free, and interest saved by directing the extra payment.
REQUIRED: `debts[]` each `{ name, balance, rate, min_payment }`, and `extra_monthly_payment`.

```
analyze_debt_payoff({
  debts: [
    { name: "Visa", balance: 9000, rate: 0.22, min_payment: 250 },
    { name: "Car loan", balance: 14000, rate: 0.06, min_payment: 320 }
  ],
  extra_monthly_payment: 400
})
```

Once you have a payoff order, if the high-rate debts are revolving (credit cards), consider
`analyze_debt_consolidation` (below) to see whether rolling them into a personal loan or a 0%
balance transfer beats the status quo.

### "Should I consolidate my high-rate debt / is a balance transfer worth it?" → `analyze_debt_consolidation`
Weighs rolling high-APR revolving (credit-card) debt into a **personal loan** or a **0% balance
transfer** (new rate + fees + intro period) against the status quo, holding the monthly budget
constant for an equal-effort compare. Returns months-to-debt-free, total interest, total fees, and
total cost per option, plus the interest-plus-fees and months saved vs status quo, and the
**recommended** option (lowest total cost). Engine aggregates the balances into a single
weighted-APR revolving leg.
REQUIRED: `balances[]` each `{ name (optional), balance, apr, min_payment (optional) }`,
`monthly_payment` (the fixed total budget reused across every option), and **at least one** of
`personal_loan` `{ rate, term_months, origination_fee_pct?, finance_fee? }` or
`balance_transfer` `{ intro_months, go_to_apr, transfer_fee_pct?, finance_fee? }`.
`{ plan_id }` optional. All rates are **fractions** (24% APR → `0.24`); all dollars are today's
(real) dollars.

```
analyze_debt_consolidation({
  balances: [{ name: "Visa", balance: 15000, apr: 0.24, min_payment: 300 }],
  monthly_payment: 500,
  personal_loan: { rate: 0.11, term_months: 36, origination_fee_pct: 0.03 },
  balance_transfer: { intro_months: 18, go_to_apr: 0.22, transfer_fee_pct: 0.03 }
})
```

Cross-links: after consolidating, run `analyze_debt_payoff` (next step — order any remaining debts)
and `analyze_funding_waterfall` (route the freed-up monthly budget to the next-best dollar).

### "Should I prepay my mortgage or invest the difference?" → `analyze_mortgage_prepay`
Compares paying extra principal (guaranteed return = the mortgage rate) against investing that money,
returning interest saved, months shaved off, and the prepay-vs-invest crossover.
REQUIRED: `principal`, `mortgage_rate`, `remaining_months`. Optional: `extra_monthly`, `lump_sum`,
`expected_investment_return`, `marginal_tax_rate`, `itemizes_deductions`, `tax_year`.

```
analyze_mortgage_prepay({ principal: 280000, mortgage_rate: 0.068, remaining_months: 312 })
```

### "Should I refinance?" → `analyze_refinance`
Returns the new vs old payment, monthly savings, lifetime interest delta, and the **break-even
months** to recoup closing costs.
REQUIRED: `principal`, `old_rate`, `old_remaining_months`, `new_rate`, `new_term_months`,
`closing_costs`.

```
analyze_refinance({
  principal: 280000, old_rate: 0.068, old_remaining_months: 312,
  new_rate: 0.059, new_term_months: 360, closing_costs: 6000
})
```

### "Should I finance, lease, or pay cash for a car?" → `analyze_auto_purchase`
Compares financing, leasing, and paying cash for a vehicle in real-dollar net cost over the holding
period (net of resale value, loan interest, and the opportunity cost of cash tied up vs invested),
and names the cheapest option. Returns the per-option net cost, the loan principal / monthly payment /
total interest, the depreciated resale value, and the cash opportunity cost. Pass `{ plan_id }` for
household context; nothing is strictly required (every field has a server-side default).

```
analyze_auto_purchase({
  vehicle_price: 42000,
  down_payment: 5000,
  loan_apr: 0.069,
  loan_term_months: 72,
  lease_monthly: 480,
  lease_term_months: 36,
  holding_period_months: 72
})
```

Cross-link: this is a natural follow-on from `analyze_debt_payoff` (before taking on a new car loan),
and its `next_actions[]` chains `generate_financial_plan` to fold the loan into the full forecast.

### "How big should my emergency fund be?" / "Do I have too much cash?" → `analyze_emergency_fund`
Sizes a months-of-runway target from job stability, dependents, single-vs-dual income, and
income variability, then checks it against current cash — flagging a shortfall, an adequate
cushion, or excess cash dragging against investing. REQUIRED: `monthly_fixed_expenses`.

```
analyze_emergency_fund({
  monthly_fixed_expenses: 4200, monthly_variable_expenses: 1500,
  job_stability: "volatile", dependents: 2,
  income_type: "single", income_variability: "variable", current_cash: 22000
})
```

### "Where should my extra savings go?" → `analyze_funding_waterfall`
Ranks the next-best-dollar destinations (e.g. high-interest debt → employer-match → HSA → tax-
advantaged → taxable) for a given surplus.
REQUIRED: `annual_surplus`, `age`, `marginal_tax_rate`. Optional: account/match details that refine
the ordering (or resolve them via `plan_id`).

```
analyze_funding_waterfall({ annual_surplus: 24000, age: 38, marginal_tax_rate: 0.24 })
```

Cross-link: if high-rate revolving debt is competing for that surplus, run
`analyze_debt_consolidation` first to see whether a personal loan or 0% balance transfer lowers the
cost of clearing it before routing the rest.

### "What's my optimal student-loan path?" → `analyze_student_loans`
Compares the repayment strategies — standard 10-year, income-driven repayment (IDR), PSLF
forgiveness (if at a nonprofit/government employer), private refinance, and aggressive payoff —
returning total cost, payoff/forgiveness timeline, and any forgiveness tax bomb per strategy, plus
the **recommended path** and an opportunity-cost note (debt rate vs investing the surplus).
Nothing is strictly REQUIRED — every field has a server-side default — but for a meaningful answer
always pass the loans and income. Provide either `loans[]` each `{ balance, rate, loan_type }` (rate
as a fraction; `loan_type` is `federal`|`private`, default `federal`) OR the single-loan scalars
`balance` + `rate`; and `annual_income` (the AGI). Optional: `filing_status` (`single`|`married_joint`),
`family_size`, `state`, `employer_type` (`nonprofit_gov`|`forprofit`; `nonprofit_gov` unlocks PSLF),
`refinance_rate` (unlocks the refinance strategy), `refinance_term_months`, `extra_monthly_payment`,
`income_growth_rate`, `idr_multiplier`, `idr_payment_rate`, `forgiveness_months`, `pslf_payments_made`,
`tax_year`, `plan_id`. (The invest-instead return is fixed server-side at 7% — there is no input for it.)

```
analyze_student_loans({
  loans: [{ balance: 120000, rate: 0.065, loan_type: "federal" }],
  annual_income: 60000,
  filing_status: "single",
  family_size: 1,
  employer_type: "nonprofit_gov"
})
```

### "What's my take-home pay? / gross-to-net paycheck / how much free cash do I have each month / what's my real savings rate / am I overspending vs 50/30/20 / paycheck breakdown after taxes and 401k" → `analyze_cash_flow`

**Always CALL `analyze_cash_flow` for these — do not answer from general knowledge or quote 50/30/20 / savings-rate rules of thumb from memory. When the user gives the numbers (gross wages, deductions, monthly bills), run it and lead with its real output.**

Computes the household monthly cash-flow waterfall — distinct from `analyze_funding_waterfall` (which decides WHERE to route savings); this computes HOW MUCH free cash exists. It (1) walks GROSS-TO-NET — from gross household wages it subtracts pre-tax deductions (401k/403b deferrals, HSA, health premiums), then federal income tax, state income tax, and FICA (Social Security 6.2% to the 2026 wage base + Medicare 1.45% + 0.9% Additional Medicare over the threshold) to land on net take-home; (2) runs the MONTHLY BUDGET WATERFALL — net pay minus fixed essentials (housing, utilities, insurance, minimum debt payments, groceries) minus discretionary = free monthly cash flow (can be negative → shortfall); (3) computes the true SAVINGS RATE two ways — gross (all retirement + taxable contributions / gross) and net (free cash flow / net take-home) — and compares to the user's FIRE-required rate; and (4) classifies spending into the 50/30/20 NEEDS/WANTS/SAVINGS benchmark and flags overspend. Reuses the same federal/state/FICA engine as the rest of the toolset (2026 brackets/limits) — no rules of thumb.
REQUIRED for a meaningful answer: `gross_annual_wages`, plus the pre-tax deductions and monthly bills the user gave. Optional: `filing_status` (`single`|`married_joint`), `state_code`, pre-tax `employee_401k_deferral` / `hsa_contribution` / `health_premiums`, budget scalars (`housing`, `utilities`, `insurance`, `minimum_debt_payments`, `groceries`, `discretionary`), `taxable_contributions`, `fire_required_savings_rate_pct`, `plan_id`.

```
analyze_cash_flow({
  gross_annual_wages: 120000,
  filing_status: "single",
  state_code: "CA",
  employee_401k_deferral: 10000,
  hsa_contribution: 4400,
  health_premiums: 3600,
  housing: 2200, utilities: 300, insurance: 250, minimum_debt_payments: 400, groceries: 600,
  discretionary: 1500
})
```

Cross-links: once you know the **free monthly cash flow**, run `analyze_funding_waterfall` to route that surplus to the next-best dollar, and `analyze_emergency_fund` to confirm the cash cushion is right-sized before investing it.

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline** — the payoff order + months-to-debt-free + interest saved; the
  prepay-vs-invest verdict + interest saved / months shaved; the refinance monthly savings + the
  **break-even month** (and whether the user expects to hold the loan that long); or the ranked
  funding-waterfall destinations.
- **Read back the `assumed_defaults[]` array** verbatim. Every one of these tools returns a
  structured `assumed_defaults[]` in its payload — each entry is `{ field, assumed_value, note }`
  covering forecast-driving inputs the caller omitted (e.g. `investment_return` 0.07,
  `marginal_tax_rate`, `itemizes_deductions`, `old_monthly_pi`, `filing_status`, etc.). Surface these
  so the user can correct any silent assumption. `disclosures.key_assumptions` is a SEPARATE block of
  static prose describing the tool's methodology — read it for context, but the per-call assumptions
  to confirm with the user live in `assumed_defaults[]`.
- Honor `disclosures.not_advice` — it is a **boolean** flag (not a message). When true, present the
  output as planning estimates, not financial advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). The actual server-defined chains for these tools are: `analyze_debt_payoff` →
  `analyze_student_loans`; `analyze_funding_waterfall` → `analyze_student_loans`;
  `analyze_student_loans` → `analyze_refinance` and `analyze_student_loans` → `analyze_funding_waterfall`;
  `analyze_emergency_fund` → `analyze_funding_waterfall` and `analyze_emergency_fund` →
  `analyze_debt_payoff`; and `analyze_debt_payoff` → `analyze_emergency_fund`. For cars:
  `analyze_debt_payoff` → `analyze_auto_purchase`, and `analyze_auto_purchase` →
  `generate_financial_plan` (fold the car loan into the full FIRE timeline).
  (`analyze_mortgage_prepay` and `analyze_refinance` have no outgoing edges.) Use whatever the tool
  actually returns rather than guessing.
- **For a share link:** `analyze_funding_waterfall`, `analyze_student_loans`, and
  `analyze_emergency_fund` each return a `share_url` of their own, but only when called with a
  `plan_id` that resolves to a plan with earners. `analyze_debt_payoff`, `analyze_mortgage_prepay`,
  `analyze_refinance`, and `analyze_auto_purchase` never return one (`analyze_auto_purchase` is a
  scalar what-if in v1)
  — if the user wants a sharable plan for those, run `generate_financial_plan` (Step 1) and surface
  its `share_url`.
- **For `analyze_student_loans`:** lead with the headline (recommended path + total cost + payoff or
  forgiveness timeline), then list the per-strategy net costs so the tradeoff is visible. Stress that
  PSLF and IDR forgiveness rules are **policy-sensitive** and can change, and that the forgiveness
  tax bomb (for non-PSLF IDR forgiveness) is an estimate.

## Recommended call sequence (typical session)

1. (preferred) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Route by intent → one of the six tools (with `{ plan_id }` plus the REQUIRED raw fields).
   When routing surplus cash, check `analyze_emergency_fund` (is the cash cushion right-sized?)
   before `analyze_funding_waterfall` (where the next dollar goes).
3. Read back the headline + the `assumed_defaults[]` array.
4. Follow `next_actions[]` — debt/cashflow questions naturally chain (debt payoff → student-loan
   path; funding waterfall → student-loan path; student-loan path → refinance or funding waterfall;
   emergency-fund check → funding waterfall or debt payoff), or run `generate_financial_plan` for a
   share link.

## Fictional examples

**1.** *"I have a $9k credit card at 22% and a $14k car loan at 6%, with $400 extra a month — what's
my payoff order?"*
→ `analyze_debt_payoff({ debts: [{ name: "Visa", balance: 9000, rate: 0.22, min_payment: 250 },
{ name: "Car loan", balance: 14000, rate: 0.06, min_payment: 320 }], extra_monthly_payment: 400 })`.
Lead with the avalanche order (card first), months to debt-free, and interest saved vs snowball.
Read back the `assumed_defaults[]` (e.g. the assumed `investment_return` driving the invest-instead
comparison). The tool's `next_actions[]` chains `analyze_student_loans` — useful if any of the debts
are student loans with IDR/PSLF options.

**2.** *"$280k mortgage at 6.8%, refi to 5.9% costs $6k in closing — is it worth it and when do I
break even?"*
→ `analyze_refinance({ principal: 280000, old_rate: 0.068, old_remaining_months: 312,
new_rate: 0.059, new_term_months: 360, closing_costs: 6000 })`. Lead with the monthly savings and the
break-even month, and flag the term reset (360 vs 312 remaining) lifetime-interest tradeoff. Read
back the `assumed_defaults[]` (e.g. the `old_monthly_pi` derived from balance/rate/term that the
refinance is compared against). You can separately offer `analyze_mortgage_prepay` to compare keeping
the old payment as extra principal — note this isn't a server-suggested `next_actions[]` edge.

**3.** *"I owe $120k of federal student loans at 6.5%, make $60k, and work for a nonprofit — should I
chase PSLF or just refinance and pay it down?"*
→ `analyze_student_loans({ loans: [{ balance: 120000, rate: 0.065, loan_type: "federal" }],
annual_income: 60000, filing_status: "single", family_size: 1, employer_type: "nonprofit_gov" })`.
Lead with the recommended path (likely PSLF here — tax-free forgiveness after 120 qualifying payments
makes its net cost lowest), the forgiveness timeline, and the per-strategy net costs. Read back the
structured `assumed_defaults[]`, and flag that PSLF/IDR rules are policy-sensitive. (Add
`refinance_rate` to also price the refinance path — but note refinancing federal loans forfeits
PSLF/IDR protections.)

*(All examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All rates/decimals are fractions; all dollars are today's (real) dollars; tax brackets/limits are
  ~2026.
- Inputs that can't be inferred from a plan, by tool: `analyze_debt_payoff` requires the `debts[]`
  array + `extra_monthly_payment`; `analyze_mortgage_prepay` requires `principal` + `mortgage_rate` +
  `remaining_months`; `analyze_refinance` requires `principal` + `old_rate` + `old_remaining_months`
  + `new_rate` + `new_term_months` + `closing_costs`; `analyze_funding_waterfall` requires
  `annual_surplus` + `age` + `marginal_tax_rate`; `analyze_emergency_fund` requires
  `monthly_fixed_expenses`. `analyze_student_loans` has no strictly-required
  field (all default server-side) but is only meaningful when you pass the loans (`loans[]` or
  `balance`/`rate`) + `annual_income`. Ask for the meaningful inputs before calling.
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
- All six tools return a structured `assumed_defaults[]` array (`{ field, assumed_value, note }`) in
  their payload — read it back so the user can correct silent defaults. `disclosures.key_assumptions`
  is separate static methodology prose, and `disclosures.not_advice` is a boolean flag. On share
  links: `analyze_funding_waterfall`, `analyze_student_loans`, and `analyze_emergency_fund` return
  their own `share_url` when called with a `plan_id` resolving to a plan with earners;
  `analyze_debt_payoff`, `analyze_mortgage_prepay`, and `analyze_refinance` do not — chain
  `generate_financial_plan` for a sharable link.
- Server-defined `next_actions[]` edges for `analyze_emergency_fund` are → `analyze_funding_waterfall`
  and → `analyze_debt_payoff` (and `analyze_debt_payoff` → `analyze_emergency_fund`).
- A guaranteed return (paying off debt / mortgage prepay at rate r) is risk-free; the prepay-vs-
  invest verdict depends on the assumed investment return the server reports — surface it.
- Not financial advice. Planning estimates only.
