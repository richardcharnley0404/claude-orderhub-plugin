---
name: daily-briefing
description: Use when the user asks for "today's briefing", "daily snapshot", "what's happening today", "morning report", or runs `/orderhub:daily`. Synthesises today's revenue, queue health, ready-for-pickup, and top concern from the OrderHub MCP into a one-screen card.
---

# Daily briefing

A polished one-screen morning summary for OrderHub lab owners. Read `references/output-format.md` for the rendering rules — they are mandatory and not subject to creative interpretation.

## Tools to call

In this order:

1. `get_daily_briefing` — main payload (revenue, order count, ready count).
2. `get_pending_pickups` — for the ready-for-pickup line and longest-wait detail.
3. `get_orders_needing_attention` — for the queue-health line and top concern.
4. `get_plugin_status` — for the update footer (see step 5).

If any tool errors, render the briefing with whatever did succeed and a one-liner like `(queue health unavailable — MCP error)`. Never block the whole briefing on one failure.

## Rendering

Follow `references/output-format.md` exactly. Do not add sections, reorder, or substitute synonyms.

## Plugin update footer

After the main briefing:

1. Read `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json` with the Read tool. Parse the `version` field — call this `installed`.
2. Compare `installed` vs `latest_version` from `get_plugin_status` using semver order.
3. If `installed < latest`, append:
   ```
   📦 OrderHub plugin v<latest> available. Run /plugin update orderhub to refresh.
   ```
4. Rate-limit: before showing the footer, attempt to read `${CLAUDE_PLUGIN_DATA}/last_update_nudge.txt`. If the file's contents (an ISO date) match today's date, skip the footer. Otherwise show it and write today's date to that file.
5. If the version comparison fails or any of the above errors, just skip the footer — never let the update nudge break the briefing.

## What NOT to do

- Don't call any write tools.
- Don't speculate ("revenue is probably down because…"). Stick to the numbers.
- Don't ask the user follow-up questions — this is a one-shot summary.
