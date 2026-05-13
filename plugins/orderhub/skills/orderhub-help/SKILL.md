---
name: orderhub-help
description: Use when the user asks "what can the orderhub plugin do", "what does orderhub do", "list orderhub features", "show me orderhub commands", "help with orderhub", "what can I ask orderhub", or runs `/orderhub:help`. Renders a plain-language menu of every OrderHub capability with example prompts the lab owner can copy and paste.
---

# OrderHub plugin help

Render a one-screen menu that tells a lab owner what the OrderHub plugin can do and exactly what to type to use each feature. The audience is non-technical — a lab manager who runs a photo lab, not a developer. Avoid jargon (no "MCP", no "skill", no "endpoint", no UUIDs).

## Output

Render this menu verbatim. Do not add commentary before or after, do not personalise, do not invent features. The menu below is the source of truth.

```
🏷  OrderHub plugin — what you can ask

Each feature works two ways: type the slash command, or just ask in plain English.

────────────────────────────────────────
📋 Morning briefing
   Today's revenue, queue health, pickups ready, and the top concern.
   • /orderhub:daily
   • "Give me today's OrderHub briefing"
   • "What's happening today?"

📦 Pickup queue
   Orders ready for collection, grouped by ready-today, this-week, and overdue.
   • /orderhub:pickup
   • "What's ready for pickup?"
   • "Anything overdue for collection?"

📧 Escalation email
   Drafts an internal email listing orders stuck 5+ days, ready to forward
   to a production manager or counter staff.
   • /orderhub:escalate
   • "Draft an escalation email for stuck pickups"
   • "What orders need chasing?"

🔍 Order lookup
   Full detail for a single order — by order number, customer name, email, or phone.
   • /orderhub:order 4521
   • "Show me order 4521"
   • "Find Smith's recent order"

📈 Sales metrics
   Revenue (with comparison to the prior period), top products, and turnaround time.
   Pick a range: 7d, 30d, mtd (month-to-date), last week, last month.
   • /orderhub:sales 7d
   • "How were sales this week?"
   • "Top products this month"

────────────────────────────────────────
Tip: anything the plugin reports is read-only — it will never change an order
or contact a customer on your behalf. Email drafts and lists are produced for
you to copy and send yourself.
```

## What NOT to do

- Don't add features that aren't in the menu above. If you don't see it listed, the plugin doesn't do it yet.
- Don't drop the slash-command lines because the user isn't in Claude Code — the slash commands work in Claude Desktop, Claude.ai web, and Claude Code identically.
- Don't ask follow-up questions. This is a one-shot menu.
- Don't translate or paraphrase the menu — render it exactly as written so the lab owner sees the same screen every time.
