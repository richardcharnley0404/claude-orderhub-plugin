---
name: order-lookup
description: Use when the user asks "show me order X", "where's order #1234", "find Smith's recent order", "look up that 4521 order", or runs `/orderhub:order <query>`. Renders the full detail panel for a single order.
---

# Order lookup

Find one order and render its full detail.

## Resolving the query

The user query can be:

1. A bare order number (e.g. `4521`, `#4521`, `1234`) — call `get_order` with that number directly. If `get_order` returns "not found" (the backend may expect a UUID rather than a customer-facing order number), fall back to `search_orders` with `q=<number>`, then call `get_order` on the resolved id.
2. A name or email or phone fragment (e.g. `Smith`, `jane@example.com`, `07700`) — call `search_orders` first with `q=<fragment>`. If exactly one result, proceed with `get_order` on its id. If multiple, list them as a numbered shortlist (max 10) and ask the user to pick. If zero, render `No orders found matching "<query>".`
3. Ambiguous (looks like neither a number nor a name) — try `search_orders` first.

## Tools to call

- `search_orders` (only when needed for resolution)
- `get_order` — once an order id is determined

## Rendering (single order)

```
📄 Order #<order_number> · <status_badge>

Customer: <first_name> <last_name>
<phone_or_email>

Created: <created_at, locale-formatted>
Ready: <ready_date or "—">

Line items
- <quantity>× <product_name> <variant> · £<line_total>
- …

Subtotal:    £<subtotal>
Discount:    −£<discount_amount>   (only if > 0; show name)
Tax:         £<tax_total>
Total:       £<order_total>

Payment: <paid badge> · <payment_method or "—">

Notes: <notes if non-empty>

Suggested next action: <derived from status>
```

### Status badges (text, not emojis)

- `confirmed` → `Confirmed`
- `completed` → `Ready for pickup`
- `picked_up` → `Picked up <date>`
- `cancelled` → `Cancelled`
- Anything else → use the raw status string

### Suggested next action heuristic

- If `paid === false` and `payment_method !== 'in_house_account'`: `"Customer needs to pay £<balance_due>."`
- If `paid === false` and `payment_method === 'in_house_account'`: `"On account · invoice due <payment_due_date>."`
- If `status === 'completed'` and `picked_up_at === null`: `"Ready for collection."`
- If `picked_up_at !== null`: `"Already picked up — no action needed."`
- Otherwise: omit the suggested-next-action line.

## What NOT to do

- Don't fabricate fields the response didn't include.
- Don't fetch more than one order per invocation. If the user wants multiple, render the first and offer to look up others.
