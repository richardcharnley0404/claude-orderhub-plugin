---
name: pickup-queue
description: Use when the user asks "what's ready for pickup", "show me the pickup queue", "who's waiting", "any overdue collections", or runs `/orderhub:pickup`. Renders pending pickups grouped by readiness and overdue status.
---

# Pickup queue

Show the lab owner what's ready for collection right now, what'll be ready this week, and what's overdue.

## Tools to call

1. `get_pending_pickups` — returns the full pending list with per-order ready_date, customer, total, and days_waiting.

## Rendering

Group orders into three buckets. Bucketing is mutually exclusive — evaluate in this order, first match wins:

1. **Overdue** — `days_waiting >= 7` (regardless of `ready_date`). An order placed 8+ days ago belongs here even if `ready_date` is still in the future — that's a job the lab is taking too long on, which the owner needs to see.
2. **Ready today** — not overdue AND `ready_date <= today`.
3. **Ready this week** — not overdue AND `ready_date > today` AND `ready_date <= today + 7 days`.

Orders that are neither overdue nor ready within 7 days (ready further in the future) are not surfaced — they're not actionable for collection yet.

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
