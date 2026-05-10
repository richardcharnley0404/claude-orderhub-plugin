---
name: sales-metrics
description: Use when the user asks "how were sales this week", "show me sales metrics", "top products this month", "what's our turnaround time", or runs `/orderhub:sales [range]`. Renders revenue, top products, and performance metrics for a chosen range.
---

# Sales metrics

A range-windowed business snapshot — revenue with delta, top products, and operational performance.

## Range argument

Read `references/range-parsing.md` to map the user's range argument (e.g. `7d`, `mtd`, `last week`) into `{ from, to }` and a comparison `{ prior_from, prior_to }`. Default to `7d` when no argument is supplied.

## Tools to call

For the chosen range:

1. `get_sales_summary({ from, to })` — current-period revenue, order count, AOV.
2. `get_sales_summary({ from: prior_from, to: prior_to })` — prior-period totals for the delta. Skip on error.
3. `get_top_products({ from, to, limit: 5 })` — top 5 by units sold.
4. `get_performance_metrics({ from, to })` — turnaround time avg, on-time rate.

## Rendering

```
📈 Sales — <range label, e.g. "last 7 days">

Revenue: £<total> · <orders> orders · AOV £<avg>
        (+12% vs prior 7 days)   ← only when prior data succeeded

Top products
1. 8×8 Photobook · 23 units · £1,150
2. A4 Print · 18 units · £540
…

Performance
Avg turnaround: 2.4 days
On-time rate:   94%
```

If the prior-period call failed, just omit the delta line. No `(comparison unavailable)` apology — staff already see what's there.

## Range label formatting

- `today` → `today`
- `yesterday` → `yesterday`
- `7d` → `last 7 days`
- `30d` → `last 30 days`
- `mtd` → `month to date`
- `qtd` → `quarter to date`
- `ytd` → `year to date`
- `last week` → `last week`
- `last month` → `last month`

## What NOT to do

- Don't render more than 5 top products even if the response has more.
- Don't include products with zero units sold.
- Don't editorialise ("a great week!", "concerning trend"). Numbers only.
