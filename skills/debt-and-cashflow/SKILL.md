---
name: debt-and-cashflow
version: 1.0.0
description: Pay down debt and route surplus cash optimally by orchestrating the public planfi MCP. Use whenever someone wants to know the order to pay off their debts (avalanche vs snowball), whether to prepay a mortgage or invest the difference, whether refinancing is worth it and when it breaks even, or where the next best dollar of savings should go — e.g. "what's my payoff order if I have a credit card and a car loan?", "should I pay extra on my mortgage or invest it?", "is it worth refinancing and when do I break even?", "where should my extra $X/month go?".
---

# Debt and Cashflow

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All the math — amortization, avalanche/snowball ordering, prepay-vs-invest opportunity cost,
refinance break-even, and the next-best-dollar funding waterfall — lives server-side. This skill
only gathers inputs and calls the tools — it does **not** compute anything locally and bakes in no
defaults of its own. Read-only.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_debt_payoff`):
`analyze_debt_payoff`, `analyze_mortgage_prepay`, `analyze_refinance`, `analyze_funding_waterfall`,
plus optional `generate_financial_plan` (for `plan_id` chaining + a `share_url`). Use whichever name
your environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — (Optional but preferred) build a plan first to chain context + get a share link

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. **All four** debt-and-cashflow tools accept `{ plan_id }` (plus
inline overrides), so they can resolve age, marginal tax rate, and surplus from the saved plan
instead of you re-sending every figure. `generate_financial_plan` also returns a **`share_url`**
(planfi.app) — none of the four tools below emit a share link themselves, so this is the way to give
the user one.

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

### "Should I prepay my mortgage or invest the difference?" → `analyze_mortgage_prepay`
Compares paying extra principal (guaranteed return = the mortgage rate) against investing that money,
returning interest saved, months shaved off, and the prepay-vs-invest crossover.
REQUIRED: `principal`, `mortgage_rate`, `remaining_months`. Optional: `extra_monthly_payment`,
`expected_investment_return`, `marginal_tax_rate`.

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

### "Where should my extra savings go?" → `analyze_funding_waterfall`
Ranks the next-best-dollar destinations (e.g. high-interest debt → employer-match → HSA → tax-
advantaged → taxable) for a given surplus.
REQUIRED: `annual_surplus`, `age`, `marginal_tax_rate`. Optional: account/match details that refine
the ordering (or resolve them via `plan_id`).

```
analyze_funding_waterfall({ annual_surplus: 24000, age: 38, marginal_tax_rate: 0.24 })
```

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline** — the payoff order + months-to-debt-free + interest saved; the
  prepay-vs-invest verdict + interest saved / months shaved; the refinance monthly savings + the
  **break-even month** (and whether the user expects to hold the loan that long); or the ranked
  funding-waterfall destinations.
- **Read back `disclosures.key_assumptions`** verbatim. These tools do **not** emit a structured
  `assumed_defaults[]` array — instead they apply silent Zod defaults (e.g. expected investment
  return, payment-application timing, marginal tax rate) and expose what they assumed only as
  **prose in `disclosures.key_assumptions`**. Surface those so the user can correct any silent
  assumption.
- Honor `disclosures.not_advice` — present as planning estimates, not financial advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). These commonly chain — e.g. debt payoff → funding waterfall (where the freed-up
  payment goes next), or mortgage prepay → refinance. Use the server-suggested chains rather than
  guessing.
- **For a share link:** these four tools don't return one. If the user wants a sharable plan, run
  `generate_financial_plan` (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (preferred) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Route by intent → one of the four tools (with `{ plan_id }` plus the REQUIRED raw fields).
3. Read back the headline + `disclosures.key_assumptions`.
4. Follow `next_actions[]` — debt/cashflow questions naturally chain (payoff → where the freed-up
   payment goes via `analyze_funding_waterfall`; prepay ↔ refinance), or back to
   `generate_financial_plan` for a share link.

## Fictional examples

**1.** *"I have a $9k credit card at 22% and a $14k car loan at 6%, with $400 extra a month — what's
my payoff order?"*
→ `analyze_debt_payoff({ debts: [{ name: "Visa", balance: 9000, rate: 0.22, min_payment: 250 },
{ name: "Car loan", balance: 14000, rate: 0.06, min_payment: 320 }], extra_monthly_payment: 400 })`.
Lead with the avalanche order (card first), months to debt-free, and interest saved vs snowball;
chain `analyze_funding_waterfall` for where the freed-up payment should go once the card is paid.
Read back key_assumptions (payment timing, whether order is avalanche vs snowball).

**2.** *"$280k mortgage at 6.8%, refi to 5.9% costs $6k in closing — is it worth it and when do I
break even?"*
→ `analyze_refinance({ principal: 280000, old_rate: 0.068, old_remaining_months: 312,
new_rate: 0.059, new_term_months: 360, closing_costs: 6000 })`. Lead with the monthly savings and the
break-even month, and flag the term reset (360 vs 312 remaining) lifetime-interest tradeoff; offer
`analyze_mortgage_prepay` to compare keeping the old payment as extra principal. Read back
key_assumptions.

*(Both examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All rates/decimals are fractions; all dollars are today's (real) dollars; tax brackets/limits are
  ~2026.
- All four tools have REQUIRED inputs that can't be inferred from a plan: `analyze_debt_payoff`
  needs the `debts[]` array + `extra_monthly_payment`; `analyze_mortgage_prepay` needs `principal` +
  `mortgage_rate` + `remaining_months`; `analyze_refinance` needs both loans' terms + `closing_costs`;
  `analyze_funding_waterfall` needs `annual_surplus` + `age` + `marginal_tax_rate`. Ask for them
  before calling.
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
  All four tools accept `plan_id`.
- These four tools surface assumptions as **prose in `disclosures.key_assumptions`**, not a
  structured `assumed_defaults[]`, and they do **not** return a `share_url` — chain
  `generate_financial_plan` for a sharable link. (Server follow-up tracked in `SKILL_AUTHORING.md`.)
- A guaranteed return (paying off debt / mortgage prepay at rate r) is risk-free; the prepay-vs-
  invest verdict depends on the assumed investment return the server reports — surface it.
- Not financial advice. Planning estimates only.
