# OrderHub for Claude Code

A Claude Code plugin that lets OrderHub lab owners interrogate their organisation's data in natural language — daily briefings, pickup queues, order lookups, and sales metrics — without leaving the chat.

> **Status:** v1 design locked, implementation in progress. Not yet published.

## What it does (v1)

Four polished workflows surfaced as both natural-language skills and slash commands:

| Slash command | Use it for |
|---|---|
| `/orderhub:daily` | Today's revenue, queue health, ready-for-pickup, and concerns at a glance |
| `/orderhub:pickup` | Pickup queue grouped by ready-today / this-week / overdue |
| `/orderhub:order <q>` | Full detail for one order — header, line items, payment, customer, notes |
| `/orderhub:sales [range]` | Sales summary, top products, performance metrics for a chosen range |

You can also ask in plain English — "what's our queue look like?", "show me yesterday's sales", "find order 1234" — and the right skill picks up automatically.

## Install (will be available once published)

```text
/plugin marketplace add richardcharnley0404/claude-orderhub-plugin
/plugin install orderhub
```

You'll be prompted for a personal access token. Generate one at:

> https://app.orderhub.io/settings/tokens *(coming soon)*

Paste it into Claude Code and you're ready.

## Updates

Plugin updates flow automatically — Claude Code re-checks the marketplace on session start. Your token survives updates; only skills, commands, and the MCP server URL are replaced.

## Design

See [`docs/design.md`](docs/design.md) for the full v1 spec.

## License

MIT — see [LICENSE](LICENSE).

---

*Pixfizz Ltd · OrderHub Claude Code Plugin*
