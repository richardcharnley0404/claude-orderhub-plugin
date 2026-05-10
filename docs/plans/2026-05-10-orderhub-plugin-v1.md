# OrderHub Claude Code Plugin v1 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a public Claude Code plugin (`richardcharnley0404/claude-orderhub-plugin`) that lets OrderHub lab owners interrogate their org via four polished read workflows (daily briefing, pickup queue, order lookup, sales metrics), authenticated by personal access tokens, with self-updating skills and an in-product update nudge.

**Architecture:** Plugin marketplace repo (this repo) declares an HTTP MCP server bound to a new bearer-authenticated `/mcp` Supabase Edge Function in OrderHub. Skills orchestrate existing read tools plus a new `get_plugin_status` tool for the version-check footer. Tokens stored in OS-secure storage by Claude Code; never in the plugin cache or repo.

**Tech Stack:** Claude Code plugin manifest (JSON), Markdown skills/commands, Supabase Edge Functions (Deno + TypeScript), Postgres migrations, Lovable web app for the token-management UI.

---

## How to read this plan

Two parallel work streams plus a final cross-cutting release section:

- **Stream A (Plugin repo)** — tasks executed in this repo by Claude / a subagent. Pure file authoring, JSON, Markdown, no live services needed for most steps.
- **Stream B (OrderHub backend)** — tasks the user (Richard) drives in Lovable / the OH Supabase project. Each task carries a copy-pasteable Lovable prompt and a manual verification step.
- **Stream C (Release & smoke)** — combines outputs from A + B for end-to-end verification and tagging.

Streams A and B can run in parallel. **Stream C blocks on both.** Sequence guidance per task is called out where it matters.

Repository spec referenced throughout: `docs/design.md`.

Existing OrderHub MCP tool surface (already implemented and live via the claude.ai integration): `get_daily_briefing`, `get_inventory_status`, `get_order`, `get_order_statistics`, `get_orders_needing_attention`, `get_pending_pickups`, `get_performance_metrics`, `get_sales_summary`, `get_top_products`, `list_customers`, `list_film_scans`, `list_jobs`, `list_orders`, `search_orders`, `update_job_status`. Stream B builds a parallel **bearer-authenticated HTTP entry point** that proxies to those existing implementations; it does not re-implement them.

---

# Stream A — Plugin marketplace repo

Working directory: `C:\Users\RichardCharnley\Documents\Claude\Projects\OrderHub\claude-orderhub-plugin\`. Branch: `main`. Initial commit `92bacc5` already landed (README, design.md, LICENSE, .gitignore).

## Task A1: Marketplace manifest

**Files:**
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Create the marketplace manifest**

Write `.claude-plugin/marketplace.json`:

```json
{
  "name": "claude-orderhub-plugin",
  "owner": {
    "name": "Pixfizz Ltd",
    "email": "support@pixfizz.com",
    "url": "https://pixfizz.com"
  },
  "plugins": [
    {
      "name": "orderhub",
      "source": "./plugins/orderhub",
      "version": "0.1.0",
      "description": "Interrogate your OrderHub organisation from Claude Code — daily briefing, pickup queue, order lookup, sales metrics.",
      "homepage": "https://github.com/richardcharnley0404/claude-orderhub-plugin"
    }
  ]
}
```

- [ ] **Step 2: Verify JSON is valid**

Run: `node -e "JSON.parse(require('fs').readFileSync('.claude-plugin/marketplace.json','utf8'))"`
Expected: no output (valid JSON exits silently).

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/marketplace.json
git commit -m "Add marketplace manifest"
```

## Task A2: Plugin manifest (with MCP block + userConfig)

**Files:**
- Create: `plugins/orderhub/.claude-plugin/plugin.json`

- [ ] **Step 1: Create the plugin manifest**

Write `plugins/orderhub/.claude-plugin/plugin.json`:

```json
{
  "name": "orderhub",
  "version": "0.1.0",
  "description": "OrderHub for Claude Code — read-only briefings, queues, order lookups, and sales metrics for lab owners.",
  "author": {
    "name": "Pixfizz Ltd",
    "email": "support@pixfizz.com"
  },
  "homepage": "https://github.com/richardcharnley0404/claude-orderhub-plugin",
  "userConfig": {
    "api_token": {
      "type": "string",
      "title": "OrderHub access token",
      "description": "Generate at https://app.orderhub.io/settings/tokens",
      "sensitive": true
    }
  },
  "mcpServers": {
    "orderhub": {
      "type": "http",
      "url": "https://mcp.orderhub.io/mcp",
      "headers": {
        "Authorization": "Bearer ${user_config.api_token}"
      }
    }
  }
}
```

- [ ] **Step 2: Verify JSON is valid**

Run: `node -e "JSON.parse(require('fs').readFileSync('plugins/orderhub/.claude-plugin/plugin.json','utf8'))"`
Expected: no output.

- [ ] **Step 3: Commit**

```bash
git add plugins/orderhub/.claude-plugin/plugin.json
git commit -m "Add orderhub plugin manifest with HTTP MCP + bearer auth"
```

> ⚠ **Note:** The `mcp.orderhub.io/mcp` URL is the eventual production URL. During development the MCP runs on a Supabase project URL (`https://<project>.functions.supabase.co/mcp`). Stream B Task B7 covers cutting the production URL over.

## Task A3: Daily-briefing skill

**Files:**
- Create: `plugins/orderhub/skills/daily-briefing/SKILL.md`
- Create: `plugins/orderhub/skills/daily-briefing/references/output-format.md`

- [ ] **Step 1: Write the output-format reference**

Write `plugins/orderhub/skills/daily-briefing/references/output-format.md`:

```markdown
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
```

- [ ] **Step 2: Write the SKILL.md**

Write `plugins/orderhub/skills/daily-briefing/SKILL.md`:

```markdown
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
```

- [ ] **Step 3: Commit**

```bash
git add plugins/orderhub/skills/daily-briefing/
git commit -m "Add daily-briefing skill (SKILL.md + output-format reference)"
```

## Task A4: Pickup-queue skill

**Files:**
- Create: `plugins/orderhub/skills/pickup-queue/SKILL.md`

- [ ] **Step 1: Write the SKILL.md**

Write `plugins/orderhub/skills/pickup-queue/SKILL.md`:

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add plugins/orderhub/skills/pickup-queue/
git commit -m "Add pickup-queue skill"
```

## Task A5: Order-lookup skill

**Files:**
- Create: `plugins/orderhub/skills/order-lookup/SKILL.md`

- [ ] **Step 1: Write the SKILL.md**

Write `plugins/orderhub/skills/order-lookup/SKILL.md`:

```markdown
---
name: order-lookup
description: Use when the user asks "show me order X", "where's order #1234", "find Smith's recent order", "look up that 4521 order", or runs `/orderhub:order <query>`. Renders the full detail panel for a single order.
---

# Order lookup

Find one order and render its full detail.

## Resolving the query

The user query can be:

1. A bare order number (e.g. `4521`, `#4521`, `1234`) — call `get_order` with that number directly.
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
```

- [ ] **Step 2: Commit**

```bash
git add plugins/orderhub/skills/order-lookup/
git commit -m "Add order-lookup skill"
```

## Task A6: Sales-metrics skill

**Files:**
- Create: `plugins/orderhub/skills/sales-metrics/SKILL.md`
- Create: `plugins/orderhub/skills/sales-metrics/references/range-parsing.md`

- [ ] **Step 1: Write the range-parsing reference**

Write `plugins/orderhub/skills/sales-metrics/references/range-parsing.md`:

```markdown
# Sales Metrics — Range Parsing

The user-supplied range argument is parsed into an ISO date pair `{ from, to }`. `to` is always today (end of day in the user's locale).

## Recognised inputs

| Input | from | to |
|---|---|---|
| `today` | start of today | end of today |
| `yesterday` | start of yesterday | end of yesterday |
| `7d` (default) | 6 days before today | end of today |
| `30d` | 29 days before today | end of today |
| `mtd` | first day of current month | end of today |
| `qtd` | first day of current quarter | end of today |
| `ytd` | first day of current year | end of today |
| `last week` | Monday of previous ISO week | Sunday of previous ISO week |
| `last month` | first day of previous month | last day of previous month |

## Comparison ranges

For the delta metric, the prior period is the immediately preceding window of equal length. E.g. if `from = 2026-05-04`, `to = 2026-05-10` (7d), prior is `2026-04-27` to `2026-05-03`.

## Garbled input

If the input doesn't match any recognised form: assume `7d` and render a one-line note `(interpreted "<input>" as 7d)` above the revenue card.
```

- [ ] **Step 2: Write the SKILL.md**

Write `plugins/orderhub/skills/sales-metrics/SKILL.md`:

```markdown
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
```

- [ ] **Step 3: Commit**

```bash
git add plugins/orderhub/skills/sales-metrics/
git commit -m "Add sales-metrics skill (SKILL.md + range-parsing reference)"
```

## Task A7: Daily slash command

**Files:**
- Create: `plugins/orderhub/commands/daily.md`

- [ ] **Step 1: Write the slash command**

Write `plugins/orderhub/commands/daily.md`:

```markdown
---
description: OrderHub daily briefing — revenue, queue health, ready for pickup, top concern.
---

Run the `daily-briefing` skill exactly per its `SKILL.md` instructions. Render the full briefing for today.
```

- [ ] **Step 2: Commit**

```bash
git add plugins/orderhub/commands/daily.md
git commit -m "Add /orderhub:daily slash command"
```

## Task A8: Pickup slash command

**Files:**
- Create: `plugins/orderhub/commands/pickup.md`

- [ ] **Step 1: Write the slash command**

Write `plugins/orderhub/commands/pickup.md`:

```markdown
---
description: OrderHub pickup queue — orders ready for collection, grouped by readiness and overdue status.
---

Run the `pickup-queue` skill exactly per its `SKILL.md` instructions.
```

- [ ] **Step 2: Commit**

```bash
git add plugins/orderhub/commands/pickup.md
git commit -m "Add /orderhub:pickup slash command"
```

## Task A9: Order slash command

**Files:**
- Create: `plugins/orderhub/commands/order.md`

- [ ] **Step 1: Write the slash command**

Write `plugins/orderhub/commands/order.md`:

```markdown
---
description: OrderHub order lookup — fetches one order's full detail. Pass an order number, customer name, email, or phone fragment.
argument-hint: <order-number-or-customer>
---

Run the `order-lookup` skill exactly per its `SKILL.md` instructions, with the user-supplied query: $ARGUMENTS

If $ARGUMENTS is empty, ask the user for a query before proceeding.
```

- [ ] **Step 2: Commit**

```bash
git add plugins/orderhub/commands/order.md
git commit -m "Add /orderhub:order slash command"
```

## Task A10: Sales slash command

**Files:**
- Create: `plugins/orderhub/commands/sales.md`

- [ ] **Step 1: Write the slash command**

Write `plugins/orderhub/commands/sales.md`:

```markdown
---
description: OrderHub sales metrics — revenue with delta, top products, and performance for a chosen range.
argument-hint: [today|yesterday|7d|30d|mtd|qtd|ytd|last week|last month]
---

Run the `sales-metrics` skill exactly per its `SKILL.md` instructions, with range argument: $ARGUMENTS

If $ARGUMENTS is empty, default to `7d`.
```

- [ ] **Step 2: Commit**

```bash
git add plugins/orderhub/commands/sales.md
git commit -m "Add /orderhub:sales slash command"
```

## Task A11: CHANGELOG

**Files:**
- Create: `CHANGELOG.md`

- [ ] **Step 1: Write the changelog**

Write `CHANGELOG.md`:

```markdown
# Changelog

All notable changes to the OrderHub Claude Code plugin.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] — 2026-05-10

### Added
- Initial design spec (`docs/design.md`).
- Marketplace manifest (`.claude-plugin/marketplace.json`).
- `orderhub` plugin manifest with HTTP MCP server declaration and personal-access-token `userConfig`.
- Four skills: `daily-briefing`, `pickup-queue`, `order-lookup`, `sales-metrics`.
- Four slash commands: `/orderhub:daily`, `/orderhub:pickup`, `/orderhub:order`, `/orderhub:sales`.
- Plugin-update nudge in `daily-briefing` (rate-limited daily, sourced from `get_plugin_status`).
- README with install steps and lab-owner-facing overview.
- MIT license.
```

- [ ] **Step 2: Commit**

```bash
git add CHANGELOG.md
git commit -m "Add CHANGELOG.md with v0.1.0 entry"
```

## Task A12: Self-test — JSON validity sweep

**Files:**
- (none — verification only)

- [ ] **Step 1: Validate every JSON file in the repo**

Run:
```bash
node -e "
const fs = require('fs');
const path = require('path');
function walk(d) {
  for (const f of fs.readdirSync(d, {withFileTypes:true})) {
    const p = path.join(d, f.name);
    if (f.isDirectory() && !p.includes('node_modules') && !p.includes('.git')) walk(p);
    else if (f.name.endsWith('.json')) {
      try { JSON.parse(fs.readFileSync(p,'utf8')); console.log('OK', p); }
      catch(e) { console.error('FAIL', p, e.message); process.exit(1); }
    }
  }
}
walk('.');
"
```

Expected output:
```
OK .\.claude-plugin\marketplace.json
OK .\plugins\orderhub\.claude-plugin\plugin.json
```

If any file fails, fix it before continuing.

- [ ] **Step 2: Confirm directory tree matches the spec**

Run: `find plugins -type f -name '*.md' -o -name '*.json' | sort`

Expected output (exact):
```
plugins/orderhub/.claude-plugin/plugin.json
plugins/orderhub/commands/daily.md
plugins/orderhub/commands/order.md
plugins/orderhub/commands/pickup.md
plugins/orderhub/commands/sales.md
plugins/orderhub/skills/daily-briefing/SKILL.md
plugins/orderhub/skills/daily-briefing/references/output-format.md
plugins/orderhub/skills/order-lookup/SKILL.md
plugins/orderhub/skills/pickup-queue/SKILL.md
plugins/orderhub/skills/sales-metrics/SKILL.md
plugins/orderhub/skills/sales-metrics/references/range-parsing.md
```

If anything is missing, return to the relevant Task and complete it.

## Task A13: Local install + dry smoke (no live MCP needed)

**Files:**
- (none — manual verification)

This task verifies the plugin loads in Claude Code without errors. Live MCP responses are not required — Stream B Task B7 covers end-to-end smoke.

- [ ] **Step 1: Set local Claude Code env to the plugin path**

In a fresh Claude Code session:

```text
/plugin marketplace add C:\Users\RichardCharnley\Documents\Claude\Projects\OrderHub\claude-orderhub-plugin
/plugin install orderhub@claude-orderhub-plugin
```

When prompted for `OrderHub access token`, paste any non-empty placeholder string (e.g. `dev-placeholder`). The MCP will return 401 since there's no valid token / route yet, but the plugin should LOAD without any manifest errors.

Expected:
- No JSON parse errors.
- Skills appear when `/skills` is run (or whatever Claude Code's discovery surface is).
- `/orderhub:daily` is recognised as a command (it'll fail at the MCP call — that's fine).

- [ ] **Step 2: Tear down local install before publishing**

```text
/plugin uninstall orderhub
/plugin marketplace remove claude-orderhub-plugin
```

- [ ] **Step 3: No commit needed** (verification only).

---

# Stream B — OrderHub backend (Lovable / Supabase)

Working location: the OrderHub Supabase project (managed via Lovable). Each task includes a copy-pasteable Lovable prompt and a manual verification step you (Richard) run in Supabase Studio or Lovable's preview.

## Task B1: Migration — `personal_access_tokens` table

**Files (in OH repo):**
- Create: `supabase/migrations/<timestamp>_personal_access_tokens.sql`

- [ ] **Step 1: Lovable prompt**

Paste into Lovable:

```
Create a new Supabase migration that adds a `personal_access_tokens` table for authenticating Claude Code plugin requests. The table:

- id uuid primary key default gen_random_uuid()
- user_id uuid not null references auth.users(id) on delete cascade
- org_id uuid not null  -- snapshot of the active organisation when token was created
- name text not null
- token_hash text not null unique  -- sha256(plaintext) hex, NEVER store plaintext
- scopes text[] not null default array['read:org']  -- forward-looking; v1 only uses 'read:org'
- last_used_at timestamptz
- created_at timestamptz not null default now()
- revoked_at timestamptz  -- soft revoke
- expires_at timestamptz  -- nullable

Create a unique index on token_hash. Enable Row Level Security with policies:
- SELECT: user_id = auth.uid()
- INSERT: user_id = auth.uid()
- UPDATE: user_id = auth.uid()  (owners can soft-revoke their own tokens by setting revoked_at)
- No DELETE policy (we soft-delete via revoked_at).

Do not create a service role bypass policy yet — we'll add that on the edge function side.
```

- [ ] **Step 2: Verify in Supabase Studio**

In SQL Editor, run:

```sql
\d personal_access_tokens
SELECT pol.polname, pol.polcmd
FROM pg_policy pol
JOIN pg_class c ON c.oid = pol.polrelid
WHERE c.relname = 'personal_access_tokens';
```

Expected: 4 policies (SELECT, INSERT, UPDATE, plus the implicit). Table has all columns above.

- [ ] **Step 3: Commit migration in OH repo**

Lovable should auto-commit. Verify the migration file is committed with a sensible message.

## Task B2: Migration — `mcp_audit_log` table

**Files (in OH repo):**
- Create: `supabase/migrations/<timestamp>_mcp_audit_log.sql`

- [ ] **Step 1: Lovable prompt**

```
Create a new Supabase migration that adds an `mcp_audit_log` table for recording MCP tool calls made via personal access tokens.

- id bigint primary key generated always as identity
- token_id uuid not null references personal_access_tokens(id) on delete cascade
- user_id uuid not null  -- denormalised for easier owner queries
- org_id uuid not null   -- denormalised
- tool text not null     -- e.g. 'get_daily_briefing'
- params jsonb           -- the tool arguments (be mindful of PII; never log full PIs)
- status text not null check (status in ('success','error','denied'))
- error_code text        -- nullable; populated on status='error' or 'denied'
- duration_ms int
- at timestamptz not null default now()

Index on (org_id, at desc) for owner audit queries.
Index on (token_id, at desc) for per-token traceability.

RLS: SELECT only by users where user_id = auth.uid(). No INSERT/UPDATE/DELETE from clients — only the service role (the edge function) writes here.
```

- [ ] **Step 2: Verify in Supabase Studio**

```sql
\d mcp_audit_log
\di mcp_audit_log_*
```

Expected: table with columns above, 2 indexes, RLS enabled.

- [ ] **Step 3: Commit migration in OH repo**

## Task B3: Edge function — `/mcp` route with bearer auth

**Files (in OH repo):**
- Create: `supabase/functions/mcp/index.ts`
- Create: `supabase/functions/mcp/auth.ts`
- Create: `supabase/functions/mcp/audit.ts`

- [ ] **Step 1: Lovable prompt — auth module**

```
Create a Supabase Edge Function file at supabase/functions/mcp/auth.ts that exports:

  export async function authenticateRequest(req: Request, supabaseAdmin: SupabaseClient):
    Promise<{ token_id: string; user_id: string; org_id: string; scopes: string[] } | { error: 'unauthorized' | 'invalid_token' }>

Behaviour:
1. Read 'Authorization' header. If missing or doesn't start with 'Bearer ', return { error: 'unauthorized' }.
2. Extract the token. SHA-256 hash it (hex lowercase).
3. SELECT id, user_id, org_id, scopes, revoked_at, expires_at FROM personal_access_tokens WHERE token_hash = $1.
4. If no row, return { error: 'invalid_token' }.
5. If revoked_at IS NOT NULL, return { error: 'invalid_token' }.
6. If expires_at IS NOT NULL AND expires_at < now(), return { error: 'invalid_token' }.
7. Fire-and-forget UPDATE personal_access_tokens SET last_used_at = now() WHERE id = $1 (don't await).
8. Return { token_id, user_id, org_id, scopes }.

Use Deno's crypto.subtle for SHA-256. Use the supabase-js admin client (service role) for the DB read.
```

- [ ] **Step 2: Lovable prompt — audit module**

```
Create supabase/functions/mcp/audit.ts that exports:

  export async function writeAuditLog(supabaseAdmin: SupabaseClient, params: {
    token_id: string
    user_id: string
    org_id: string
    tool: string
    params: unknown
    status: 'success' | 'error' | 'denied'
    error_code?: string
    duration_ms: number
  }): Promise<void>

Behaviour: insert one row into mcp_audit_log. Don't throw on insert failure — log to console.error and return. We don't want audit-log failures to bring down a tool call.
```

- [ ] **Step 3: Lovable prompt — main route**

```
Create supabase/functions/mcp/index.ts that:

1. Accepts POST requests at /mcp with a JSON body matching the MCP JSON-RPC protocol.
2. Calls authenticateRequest. On error, returns 401 with body { jsonrpc: '2.0', error: { code: -32000, message: <error code> }, id: <request id> }.
3. For 'tools/list' requests, returns the static list of tool descriptors (see step 4).
4. For 'tools/call' requests, dispatches to the existing tool implementations in our codebase. Re-use the same handlers we already use for the claude.ai-integration MCP. Pass the resolved org_id to scope every query.
5. After dispatch, writes an mcp_audit_log row with status='success' or 'error' and duration_ms.
6. Returns the standard MCP JSON-RPC response.

The supported tools for v1 are exactly:
- get_daily_briefing
- get_inventory_status
- get_order
- get_order_statistics
- get_orders_needing_attention
- get_pending_pickups
- get_performance_metrics
- get_sales_summary
- get_top_products
- list_customers
- list_film_scans
- list_jobs
- list_orders
- search_orders
- update_job_status
- get_plugin_status   ← NEW, see Task B4

For scope check: every tool except update_job_status requires 'read:org' in scopes; update_job_status requires 'write:jobs' (which v1 tokens don't have, so calls return 403 'insufficient_scope'). This is forward-compat — v2 will issue write-scoped tokens.

CORS: enable for any origin since Claude Code calls from the user's machine.
```

- [ ] **Step 4: Verify locally with curl**

After Lovable deploys the function, run from terminal:

```bash
# Without auth — expect 401
curl -X POST https://<project>.functions.supabase.co/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'

# With a bogus token — expect 401
curl -X POST https://<project>.functions.supabase.co/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer not-a-real-token" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

Expected: both return HTTP 401 with a JSON-RPC error body.

- [ ] **Step 5: Commit (Lovable auto-commits)**

## Task B4: Add `get_plugin_status` MCP tool

**Files (in OH repo):**
- Modify: wherever the existing tool implementations live (Lovable will know — most likely `supabase/functions/_shared/mcp-tools/` or similar).
- Modify: `supabase/functions/mcp/index.ts` to register the new tool.

- [ ] **Step 1: Lovable prompt**

```
Add a new MCP tool `get_plugin_status` to our tool registry. It takes no arguments.

It returns:
{
  latest_version: string  // e.g. "0.1.0"
  changelog_url: string   // e.g. "https://github.com/richardcharnley0404/claude-orderhub-plugin/blob/main/CHANGELOG.md"
}

Source: read from a new `app_settings` row, key='plugin_latest_version'. Create the row with value '0.1.0' if it doesn't exist. We'll bump this manually as part of each plugin release.

If you need to create the app_settings table, do so as a separate migration:
  - key text primary key
  - value text not null
  - updated_at timestamptz not null default now()

Wire the tool into the MCP route's tool list and dispatch.

Scope required: 'read:org' (same as other read tools).
```

- [ ] **Step 2: Verify with a real token (after Task B5)**

Defer verification until B5 issues a real token. The tool will be exercised end-to-end in Stream C.

- [ ] **Step 3: Commit**

## Task B5: Lovable web — Settings → Personal access tokens page

**Files (in OH repo):**
- Create: a new React route in the Lovable web app, e.g. `src/pages/settings/PersonalAccessTokens.tsx` (path will depend on Lovable's structure).
- Modify: the Settings layout to include a "Personal access tokens" tab/sidebar entry.

- [ ] **Step 1: Lovable prompt**

```
Add a new "Personal access tokens" page to the Settings area of the OrderHub web app, accessible only to users with the 'owner' role for the current organisation.

The page lists existing tokens for the current user (name, scopes, last used, created, revoke button) and has a "Generate new token" button that opens a modal with:
- Name (free text, required)
- Scopes: a checkbox list, but v1 has only one option ("read:org") and it's preset checked + disabled. UI is in place for v2 to add more.
- Expiry: a select with options "Never (default)", "30 days", "90 days", "1 year". A soft warning under "Never" that says "Consider setting an expiry — you can always rotate tokens later."

On submit:
1. Generate a high-entropy random token: `oh_pat_` + 40 hex chars (160 bits of randomness).
2. Compute sha256(token) hex lowercase.
3. INSERT INTO personal_access_tokens (user_id, org_id, name, token_hash, scopes, expires_at) VALUES (...).
4. Show the plaintext token in a modal with a copy button and a clear "you won't see this again" warning. Replace the modal with a "Done" confirmation when the user closes it.

Revoke action: UPDATE personal_access_tokens SET revoked_at = now() WHERE id = ?. Show a confirmation dialog ("Revoke X? Any Claude Code install using this token will stop working immediately.").

The whole UI must be RLS-safe — the queries use the user's authenticated session, never the service role.

Add a link from the page header: "How to use this in Claude Code: <link to README.md install section>".
```

- [ ] **Step 2: Manual verification**

In OH web:
- Log in as a lab owner.
- Navigate to Settings → Personal access tokens.
- Generate a token named "Smoke test".
- Copy it.
- Verify the row appears in the list with `last_used_at = null`.
- Revoke it. Verify the row gets a strikethrough or "Revoked" pill.

- [ ] **Step 3: Commit**

## Task B6: Smoke — generated token works against /mcp

**Files:**
- (none — verification only)

- [ ] **Step 1: Generate a real token**

In OH web Settings → Personal access tokens, generate a token named "Local smoke test". Copy the plaintext.

- [ ] **Step 2: Curl the MCP route with the real token**

```bash
TOKEN="oh_pat_..."   # paste the real one

# tools/list — expect 200 with the full tool list
curl -X POST https://<project>.functions.supabase.co/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' | jq .

# tools/call get_plugin_status — expect 200 with { latest_version, changelog_url }
curl -X POST https://<project>.functions.supabase.co/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"get_plugin_status","arguments":{}},"id":2}' | jq .
```

Expected: both succeed. The audit log has 2 new rows.

- [ ] **Step 3: Verify audit log**

In SQL Editor:

```sql
SELECT tool, status, duration_ms, at FROM mcp_audit_log ORDER BY at DESC LIMIT 5;
```

Expected: 2 rows with status='success'.

- [ ] **Step 4: Verify revoked token returns 401**

Revoke the token in the UI, then re-run the curl above. Expect 401 `invalid_token` and an audit row is **not** written (the request never reached dispatch — it was rejected at auth).

## Task B7: Cut MCP URL over to production domain

**Files:**
- Modify: `plugins/orderhub/.claude-plugin/plugin.json` (in this repo, Stream A).

- [ ] **Step 1: Confirm the production MCP URL**

Decide and document. Two options:
- A custom domain `https://mcp.orderhub.io/mcp` (requires DNS + Supabase custom domain setup).
- The raw Supabase URL `https://<project>.functions.supabase.co/mcp`.

For v1, **the raw Supabase URL is acceptable**. Custom domain is a later concern.

- [ ] **Step 2: Update plugin.json**

If the plugin.json from Task A2 has a placeholder URL, replace it now with the real one. Edit `plugins/orderhub/.claude-plugin/plugin.json`:

```json
"url": "https://<actual-project>.functions.supabase.co/mcp"
```

- [ ] **Step 3: Commit**

```bash
git add plugins/orderhub/.claude-plugin/plugin.json
git commit -m "Wire plugin MCP URL to production Supabase endpoint"
```

---

# Stream C — Release & smoke

## Task C1: End-to-end smoke from a fresh Claude Code

**Files:**
- (none — manual verification)

- [ ] **Step 1: Start fresh**

In Claude Code:
```text
/plugin uninstall orderhub
/plugin marketplace remove claude-orderhub-plugin
```

(If neither is installed, both will error harmlessly.)

- [ ] **Step 2: Install from the local repo path**

```text
/plugin marketplace add C:\Users\RichardCharnley\Documents\Claude\Projects\OrderHub\claude-orderhub-plugin
/plugin install orderhub@claude-orderhub-plugin
```

When prompted for the access token, paste a real generated PAT from Task B5.

- [ ] **Step 3: Run all four slash commands**

In sequence:
```text
/orderhub:daily
/orderhub:pickup
/orderhub:order 4521          (use any real order number from your dataset)
/orderhub:sales 7d
```

Expected for each:
- Non-empty output that follows the rendering rules in the relevant SKILL.md.
- No "MCP error" or "Tool not found" messages.
- Daily briefing does NOT show a plugin-update footer (installed === latest).

- [ ] **Step 4: Test the natural-language path**

In a fresh chat, type: `What's our pickup queue look like?`

Expected: Claude triggers the `pickup-queue` skill automatically and renders the same output as `/orderhub:pickup`.

- [ ] **Step 5: Test the plugin-update nudge**

In SQL Editor, bump the canonical version:

```sql
UPDATE app_settings SET value = '0.2.0', updated_at = now() WHERE key = 'plugin_latest_version';
```

Run `/orderhub:daily` again. Expected: the briefing now includes the footer:
```
📦 OrderHub plugin v0.2.0 available. Run /plugin update orderhub to refresh.
```

Run `/orderhub:daily` a second time within the same minute. Expected: footer is **NOT** shown (rate-limited via `last_update_nudge.txt`).

Reset the version after testing:
```sql
UPDATE app_settings SET value = '0.1.0', updated_at = now() WHERE key = 'plugin_latest_version';
```

- [ ] **Step 6: Test revoked-token UX**

Revoke the smoke-test token in OH web Settings. Run `/orderhub:daily` again. Expected: a clear failure message ("Authentication failed — your OrderHub access token may have been revoked. Generate a new one at https://orderhub.pixfizz.com/organizations (API tab) and update via /plugin config orderhub.") rather than a silent crash.

(The exact phrasing of this failure message is TBD — file an enhancement issue if the UX is too rough. v1 acceptable bar: "user can tell something's wrong with auth.")

- [ ] **Step 7: No commit needed** (verification only).

## Task C2: Push to GitHub & create the public repo

**Files:**
- (none — git remote operations only)

- [ ] **Step 1: Create the GitHub repo**

Either via the web UI at github.com → New repository → name `claude-orderhub-plugin`, public, no README/license/gitignore (we already have them).

Or via gh CLI:
```bash
gh repo create richardcharnley0404/claude-orderhub-plugin --public --description "Interrogate your OrderHub organisation from Claude Code"
```

- [ ] **Step 2: Wire remote and push**

```bash
git remote add origin https://github.com/richardcharnley0404/claude-orderhub-plugin.git
git push -u origin main
```

- [ ] **Step 3: Tag v0.1.0**

```bash
git tag -a v0.1.0 -m "v0.1.0 — initial release: read-only workflows for lab owners"
git push origin v0.1.0
```

- [ ] **Step 4: Verify the marketplace install path works from GitHub**

In a fresh Claude Code:
```text
/plugin marketplace remove claude-orderhub-plugin   (clear the local-path version)
/plugin marketplace add richardcharnley0404/claude-orderhub-plugin
/plugin install orderhub@claude-orderhub-plugin
```

Re-run Tasks C1 Steps 2-3 to confirm the GitHub-sourced install behaves identically to the local-path install.

## Task C3: Update OH `app_settings.plugin_latest_version` to match

**Files:**
- (none — DB update via SQL Editor)

- [ ] **Step 1: Sync the canonical latest version**

In Supabase SQL Editor:

```sql
UPDATE app_settings SET value = '0.1.0', updated_at = now() WHERE key = 'plugin_latest_version';
SELECT * FROM app_settings WHERE key = 'plugin_latest_version';
```

Expected: value = `0.1.0`. This means freshly-installed lab owners with `0.1.0` will not see an update footer (correct — they're current).

- [ ] **Step 2: Document the release ritual**

Append to `CHANGELOG.md`:

```markdown
## Release ritual

For each release:
1. Bump `version` in `plugins/orderhub/.claude-plugin/plugin.json` AND in `.claude-plugin/marketplace.json`.
2. Add a changelog section at the top of this file.
3. Commit, tag (`vX.Y.Z`), push.
4. In OH Supabase SQL Editor, update `app_settings.plugin_latest_version` to match.
5. Lab owners with `0.X.Y < new version` will see the update footer in their next daily briefing.
```

Commit:
```bash
git add CHANGELOG.md
git commit -m "Document release ritual in CHANGELOG"
git push origin main
```

## Task C4: Announce v0.1.0 to pilot lab owners

**Files:**
- (none — communications)

- [ ] **Step 1: Draft the announcement**

A single paragraph + a code block. Pick the comms channel (email, Slack, in-app banner) per Pixfizz's normal customer outreach.

```
OrderHub for Claude Code is live.

If you use Claude Code (claude.com/code), you can now ask Claude about your OrderHub data in plain English — daily briefings, pickup queue, order lookups, sales metrics. Install:

  /plugin marketplace add richardcharnley0404/claude-orderhub-plugin
  /plugin install orderhub

You'll be prompted for a personal access token — generate one at https://orderhub.pixfizz.com/organizations (API tab). The plugin self-updates as we ship more workflows.

Read-only for now; write actions (notes, inventory adjustments) coming in v0.2.

— Pixfizz
```

- [ ] **Step 2: Send.**

- [ ] **Step 3: Watch the audit log for early adoption signal**

```sql
SELECT date_trunc('hour', at) AS hour, count(*) AS calls, count(DISTINCT user_id) AS unique_users
FROM mcp_audit_log
WHERE at > now() - interval '7 days'
GROUP BY 1 ORDER BY 1 DESC;
```

This is your first real-usage telemetry. Anomalies, errors, and tool-call distributions will inform the v0.2 spec.

---

# Self-review checklist

Run before declaring v0.1.0 complete:

- [ ] All Stream A tasks committed.
- [ ] All Stream B tasks merged in OH repo with Lovable.
- [ ] Stream C C1 smoke passes for all four commands + natural-language path + update nudge + revoked-token path.
- [ ] Stream C C2 GitHub repo public and tag v0.1.0 visible.
- [ ] Stream C C3 `app_settings.plugin_latest_version = '0.1.0'`.
- [ ] CHANGELOG mentions v0.1.0 and release ritual.

# Out of scope (deferred to v0.2+)

- Write tools surfaced via skills (notes, status updates, inventory adjustments, customer comms).
- `orderhub-analyst` sub-agent for ad-hoc multi-step investigations.
- Custom domain `mcp.orderhub.io` (using raw Supabase URL in v0.1.0).
- OAuth-based auth (PAT is v0.1.0 model).
- Snapshot/regression test framework for skill outputs (manual review for now).
- Formal Deno test suite for the `/mcp` edge function (the spec §7 cases — missing auth, bad hash, revoked, expired, out-of-scope, cross-org — are covered in v0.1.0 by manual curl smoke in Task B6 + C1; promote to automated tests once Lovable's Deno test integration is comfortable).
- Telemetry beyond the audit log (no separate telemetry table).
- More than 5 top products in `sales-metrics`.

---

*Pixfizz Ltd · OrderHub Claude Code Plugin · v0.1.0 implementation plan · 2026-05-10*
