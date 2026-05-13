---
name: pickup-escalation
description: Use when the user asks for an "escalation email", "draft an escalation", "what orders need chasing", "internal email about overdue pickups", "email to production manager about stuck orders", or runs `/orderhub:escalate`. Produces a copy-pasteable plain-text email body listing pickup orders aged 5+ days, grouped by location, for the lab owner to forward to a production manager or counter staff.
---

# Pickup escalation email

Generate a plain-text email body the lab owner can paste into Gmail/Outlook and forward to a production manager (or counter staff) to chase stuck pickup orders. This is an **internal** email — the recipient is staff, not a customer.

## Tools to call

1. `get_pending_pickups` — pending list with `days_waiting`, customer, total, and location fields.

## Filtering

Include only orders where `days_waiting >= 5`.

If `get_pending_pickups` returns location data per order (e.g. `pickup_location_name`, `location`, or similar), retain it for grouping. If no location field is present, treat all orders as a single ungrouped bucket.

If zero orders pass the filter, output a single line and stop — do NOT generate an email body:

```
✅ Nothing to escalate — no pickup orders aged 5+ days.
```

## Output

Produce a plain-text email block (no HTML, no markdown inside the body) wrapped in a fenced code block so it's easy to copy. After the fenced block, append a single status line outside the block.

Example shape:

````
```
Subject: Pickup queue escalation — N orders 5+ days waiting

Hi,

The following pickup orders have been sitting in the queue for 5 or more days and need chasing:

— Manchester —
#4400 · Jane Smith · £45.00 · 12 days waiting
#4412 · John Doe · £120.00 · 7 days waiting

— Leeds —
#4399 · Old Customer · £30.00 · 9 days waiting

Suggested actions: chase production where work isn't finished, call the customer where the work is ready, or check whether anything has shipped without a status update.

Thanks
```

📧 Drafted escalation for N orders across L location(s). Copy the block above into your email client; recipient and signature are not pre-filled.
````

## Rules for the email body

- Subject line always includes total order count.
- Sort within each location group by `days_waiting` descending (longest wait first).
- If there's only one location (or no location data), omit the `— location —` headers entirely.
- Cap each location group at 25 rows; if more, list 25 then append `… and N more in <location>.` as the final row of that group.
- Leave salutation (`Hi`) and sign-off (`Thanks`) without names — the user fills these in.
- Plain text only inside the email body. No bold, no markdown, no HTML, no emoji.
- Currency: render `total` as-is from the MCP response with a `£` prefix. If the MCP returns a currency code, trust the MCP and switch the prefix accordingly.

## What NOT to do

- Don't call any MCP write tools.
- Don't include customer email addresses or phone numbers — internal staff don't need PII to action this.
- Don't speculate on causes ("probably stuck in printing because…"). Stick to facts from the data.
- Don't ask the user follow-up questions — this is a one-shot draft.
- Don't render an email body when the filter returns zero — just the ✅ line.
