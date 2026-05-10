---
name: pickup-queue
description: Use when the user asks "what's ready for pickup", "show me the pickup queue", "who's waiting", "any overdue collections", or runs `/orderhub:pickup`. Renders pending pickups grouped by readiness and overdue status.
---

# Pickup queue

Show the lab owner what's ready for collection right now, what'll be ready this week, and what's overdue.

## Tools to call

1. `get_pending_pickups` — returns the full pending list with per-order ready_date, customer, total, and days_waiting.

## Rendering

Group orders into three buckets based on `ready_date` relative to today:

1. **Ready today** — `ready_date <= today` AND `days_waiting < 7`
2. **Overdue** — `ready_date <= today` AND `days_waiting >= 7`
3. **Ready this week** — `ready_date > today` AND `ready_date <= today + 7 days`

Output structure:

```
📦 Pickup queue — N total ready, M overdue

## Ready today (X)
- #4521 · Jane Smith · £45.00 · waiting 2 days
- #4519 · John Doe · £120.00 · waiting 1 day
…

## Overdue (Y)
- #4400 · Old Customer · £30.00 · waiting 12 days ⚠

## Ready this week (Z)
- #4530 · …
```

If a bucket is empty, omit it entirely (don't print "(0)").

If the entire list is empty: `📦 Pickup queue is clear — no orders waiting collection.`

## Sort within buckets

By `days_waiting` descending (longest wait first) within each bucket. The most-aged order is always at the top of its group.

## What NOT to do

- Don't include orders that have already been picked up (`picked_up_at` should be null in the response — `get_pending_pickups` filters these out, but double-check).
- Don't render more than 25 rows per bucket — if there are more, list 25 and append `… and N more.`
