# OrderHub for Claude

Ask Claude about your OrderHub organisation in plain English — daily briefings, pickup queue, order lookups, sales metrics — without leaving the chat. Works in **Cowork**, **Claude Desktop**, and **Claude Code**.

## Two-plugin model

OrderHub for Claude ships as two plugins that work together:

| Plugin | Role | Where it comes from |
|---|---|---|
| **OrderHub Authenticator** | Registers the OrderHub MCP connector in your Claude client. Install once per device, leave installed. | Downloaded directly from your **OrderHub web app** → Settings → API Integrations → Download Plugin (on a "Claude Desktop" API key card). |
| **OrderHub** (this repo) | Contains the auto-updating skills and slash commands (`/orderhub:daily`, etc.). | This GitHub marketplace, installable via Claude Code (developers) or — in a future release — directly into Cowork. |

The Authenticator's job is one-and-done: register an MCP connector with your API key embedded. The connector persists in Claude's settings even if the Authenticator plugin is later uninstalled. The marketplace plugin's skills route through that persistent connector.

## Install — pick your client

### Cowork / Claude Desktop (recommended for most lab owners)

1. In your **OrderHub web app** → Settings → **API Integrations** → create or open an API key with **Integration Type = "Claude Desktop"**.
2. On the key's card, click **Download Plugin**. A file named `OrderHub-Authenticator.plugin` downloads.
3. In Cowork (or Claude Desktop): **Settings → Plugins** → drag the downloaded file in to install.
4. Open a new chat. Ask: *"What OrderHub tools do you have?"* — you should see the OrderHub tool list. Try *"Show me today's OrderHub briefing."*

To get the same polished briefing format the Claude Code plugin renders, also paste the system instructions from [`docs/claude-ai-project.md`](docs/claude-ai-project.md) into a Cowork / Claude.ai **Project** named "OrderHub Operations". Then open chats inside that Project for curated output.

### Claude Code (developers / power users)

```text
/plugin marketplace add richardcharnley0404/claude-orderhub-plugin
/plugin install orderhub@claude-orderhub-plugin
```

When prompted for the OrderHub access token, paste a Personal access token from your **OrderHub web app** → Settings → API tab → Personal access tokens. The token is stored in your OS keychain by Claude Code.

This gives you the four curated slash commands (`/orderhub:daily`, `/orderhub:pickup`, `/orderhub:order <q>`, `/orderhub:sales [range]`) plus auto-updating skill rendering — without needing the Authenticator plugin path.

## What you get

| Ask Claude | Claude does |
|---|---|
| *"What's today's briefing?"* | Pulls today's revenue, queue health, ready-for-pickup count, and any orders needing attention. |
| *"Show me the pickup queue."* | Lists orders ready for collection, grouped into Ready today / This week / Overdue. |
| *"Where's order 4521?"* | Renders the full detail panel — line items, payment, customer, suggested next action. |
| *"How were sales last week?"* | Revenue with delta vs prior week, top 5 products, turnaround time, on-time rate. |

Same data, every platform. Claude Code adds slash-command shortcuts.

## Token rotation & revocation

**Cowork / Claude Desktop users:**
- Manage your Claude Desktop API keys at OrderHub web → Settings → API Integrations.
- To rotate: deactivate the old key, create a new "Claude Desktop" key, click Download Plugin on the new card, install the fresh `.plugin` file in Cowork. The previous connector keeps working until you deactivate the old key.
- To revoke an installed plugin's access immediately: deactivate the corresponding API key in OrderHub web.

**Claude Code users:**
- Manage Personal access tokens at OrderHub web → Settings → API tab.
- Rotate via `/plugin config orderhub` (paste a new token) and revoke the old one in OrderHub.

## Updates

The **marketplace plugin** auto-updates: new skills, refined output formatting, and tool surface changes flow on Claude Code session start (or via `/plugin marketplace update`). Your token and connector persist across updates.

The **Authenticator plugin** doesn't change often (it's just an MCP URL plus your API key). When server-side changes do require a new Authenticator (rare), the OrderHub web app's API Integrations page will say so — re-download and re-install.

The MCP server itself is server-side, so tool changes, new tools, and bug fixes reach every install instantly — no client action needed.

## Architecture

```
                  ┌──────────────────────────────────┐
                  │ OrderHub web app                 │
                  │ Settings → API Integrations      │
                  │                                  │
                  │  ┌────────────────────────────┐  │
                  │  │ Key: "Laptop"  (Claude    │  │
                  │  │  Desktop type)            │  │
                  │  │ [ Download Plugin ]   ────┼──┼─→  user downloads file
                  │  └────────────────────────────┘  │
                  └──────────────────────────────────┘
                                                      │
                                          OrderHub-Authenticator.plugin
                                                      │
                                                      ▼
       ┌───────────────────────────────────────────────────────┐
       │ Cowork / Claude Desktop                                │
       │                                                        │
       │  Install plugin → MCP connector "orderhub" registers   │
       │  pointing at /mcp-server/mcp?token=<key>              │
       │  (persistent across plugin uninstall)                  │
       │                                                        │
       │  ──────────────────────────────────────────────       │
       │                                                        │
       │  Install OrderHub marketplace plugin (from GitHub):    │
       │  Skills + slash commands call mcp__orderhub__<tool>   │
       │  → routed through the persistent connector             │
       │  → results synthesised into curated cards              │
       └───────────────────────────────────────────────────────┘
```

## Design & specs

- [`docs/design.md`](docs/design.md) — full v1 design spec.
- [`docs/plans/2026-05-10-orderhub-plugin-v1.md`](docs/plans/2026-05-10-orderhub-plugin-v1.md) — implementation plan for v0.1.0.
- [`docs/claude-ai-project.md`](docs/claude-ai-project.md) — Claude.ai Project system instructions that replicate the curated rendering for non-Claude-Code users.

## License

MIT — see [LICENSE](LICENSE).

---

*Pixfizz Ltd · OrderHub for Claude*
