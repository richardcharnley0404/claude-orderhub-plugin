# Daily Briefing — Output Format

Render the briefing as a single, scannable card. Always in this exact order:

## Required sections

1. **Header line:** `📊 OrderHub — <today's date in user locale>`
2. **Revenue today:**
   - One line: `Revenue today: £X · Y orders`
   - If yesterday's data is in the response, append a delta: `(+12% vs yesterday)` or `(−4% vs yesterday)`. Skip if no comparison data.
3. **Queue health:** one line summarising `get_orders_needing_attention`. Examples:
   - `✅ Queue healthy — no orders need attention.`
   - `⚠ 3 orders need attention — 1 overdue, 2 stalled.`
4. **Ready for pickup:** one line from `get_pending_pickups` total count. Example:
   - `📦 Ready for pickup: 12 orders (longest wait: 3 days)`
5. **Top concern (optional):** if `get_orders_needing_attention` returned anything, surface the single most urgent one inline. Example:
   - `Top concern: Order #4521 — overdue 3 days, customer Smith.`
6. **Plugin update footer (only when applicable):** see SKILL.md step 5.

## Tone

Terse, factual, no marketing language. No emojis beyond the section markers above. Never invent data — if a tool returns nothing for a metric, omit that line rather than fabricate.

## Length

Keep under 12 lines total. The lab owner will read this between phone calls.
