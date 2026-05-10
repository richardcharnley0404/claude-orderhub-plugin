# OrderHub for Claude

Ask Claude about your OrderHub organisation in plain English — daily briefings, pickup queue, order lookups, sales metrics. Works on Claude Desktop, Claude.ai web, and Claude Code.

This repo ships:

1. The **`claude-orderhub-plugin`** — for Claude Code users (developers / power users).
2. **Setup instructions** for Claude Desktop and Claude.ai web — most lab owners want this path.
3. A **Project template** for Claude.ai that gives Desktop / web users the same polished briefings the Claude Code plugin renders. See [`docs/claude-ai-project.md`](docs/claude-ai-project.md).

All three connect to the same OrderHub MCP server, so the tool surface and access control are identical no matter which one you use.

## What it does

| You ask Claude | Claude does |
|---|---|
| *"What's today's briefing?"* | Pulls today's revenue, queue health, ready-for-pickup count, and any orders needing attention. |
| *"Show me the pickup queue."* | Lists orders ready for collection, grouped into Ready today / This week / Overdue. |
| *"Where's order 4521?"* | Renders the full detail panel — line items, payment, customer, suggested next action. |
| *"How were sales last week?"* | Revenue with delta vs prior week, top 5 products, turnaround time, on-time rate. |

The same data is available to Claude on every platform. The Claude Code plugin adds polished formatting and slash-command shortcuts (`/orderhub:daily`, etc.) on top.

---

## Install — pick your platform

### Option 1: Claude Desktop (recommended for most lab owners)

This is the path most lab owners want. Works on macOS and Windows.

**1. Generate an OrderHub access token.** Go to https://orderhub.pixfizz.com/organizations → API tab → Personal access tokens → "Generate new token". Copy the value.

**2. Open Claude Desktop's config file.**

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%AppData%\Claude\claude_desktop_config.json`

If the file doesn't exist yet, create it.

**3. Add OrderHub to your `mcpServers`** (merge with anything already there):

```json
{
  "mcpServers": {
    "orderhub": {
      "type": "http",
      "url": "https://nazkcvruighrhpgcarxg.supabase.co/functions/v1/mcp",
      "headers": {
        "Authorization": "Bearer PASTE_YOUR_TOKEN_HERE"
      }
    }
  }
}
```

**4. Restart Claude Desktop.** The next time you open a chat you'll see the OrderHub tools available; Claude will use them automatically when you ask the right questions.

**5. (Optional) Add the OrderHub Project template** — see [`docs/claude-ai-project.md`](docs/claude-ai-project.md) for system instructions that teach Claude to render daily briefings, pickup queues, and sales reports in the same polished format the Claude Code plugin uses.

### Option 2: Claude.ai web

Claude.ai's "Connect Apps" / Integrations UI may support custom HTTP MCP servers depending on your plan. If yours does, use the same URL and bearer-token configuration as the Claude Desktop instructions above.

If your plan doesn't expose custom HTTP MCP setup in the web UI, the cleanest fallback is **Claude Desktop on the same machine** (Option 1) — Desktop and web share your account, but only Desktop reads the `claude_desktop_config.json` file.

To get the same polished output formatting in a web chat, create a Claude.ai Project and paste the system instructions from [`docs/claude-ai-project.md`](docs/claude-ai-project.md).

### Option 3: Claude Code (developers / power users)

If you live in Claude Code already, install the plugin from this marketplace:

```text
/plugin marketplace add richardcharnley0404/claude-orderhub-plugin
/plugin install orderhub
```

Paste your access token when prompted. You'll get the same tools plus four slash commands (`/orderhub:daily`, `/orderhub:pickup`, `/orderhub:order <q>`, `/orderhub:sales [range]`) and curated skill rendering wired in.

Plugin updates flow automatically on session start; your token survives updates.

---

## Token rotation & revocation

Personal access tokens are managed at https://orderhub.pixfizz.com/organizations → API tab. You can:

- See when each token was last used.
- Revoke any token immediately — calls authenticated with that token will start failing right away.
- Generate a new token to replace one — paste the new value into your Claude Desktop config / Claude Code prompt.

Tokens never expire by default but you can opt into 30 / 90 / 365 day expiry per token. We recommend rotating annually.

---

## Updates

Tool changes (new tools, improved descriptions, server-side fixes) ship the moment we deploy the OrderHub backend — no client action required. Every Claude install automatically picks them up on next call.

For the Claude Code plugin specifically (`docs/design.md`), version-pinned skill / slash-command updates flow via Claude Code's marketplace check on session start, or via `/plugin marketplace update` on demand. The daily-briefing surfaces a one-line nudge when a newer plugin version is available.

---

## Design & specs

- [`docs/design.md`](docs/design.md) — full v1 design spec.
- [`docs/plans/2026-05-10-orderhub-plugin-v1.md`](docs/plans/2026-05-10-orderhub-plugin-v1.md) — the implementation plan that built v0.1.0.
- [`docs/claude-ai-project.md`](docs/claude-ai-project.md) — Claude.ai Project system instructions that mimic the polished plugin rendering.

## License

MIT — see [LICENSE](LICENSE).

---

*Pixfizz Ltd · OrderHub for Claude*
