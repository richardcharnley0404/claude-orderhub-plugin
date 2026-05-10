# OrderHub Claude Code Plugin — v1 Design

**Status:** Approved 2026-05-10. Implementation pending.
**Owner:** Pixfizz Ltd · OrderHub
**Audience for v1:** OrderHub lab owners (org admins).
**Repo:** `richardcharnley0404/claude-orderhub-plugin` (this repo).

---

## 1. Overview

A Claude Code plugin distributed via a public marketplace repo (this one) that lets OrderHub lab owners interrogate their organisation's data in natural language. v1 is read-only and packages four polished workflows (daily briefing, pickup queue, order detail lookup, sales metrics) over the existing OrderHub MCP server. Authentication uses personal access tokens (PATs) that owners generate from a new OrderHub web settings page. Skills and slash commands self-update through the standard Claude Code marketplace mechanism; the daily briefing carries an inline footer when a new plugin version is available.

**Out of scope for v1** (deferred to v1.1+): write capabilities (notes, status updates, inventory adjustments, customer comms), an `orderhub-analyst` sub-agent for ad-hoc analysis, OAuth-based auth.

## 2. Architecture

Two pieces of infrastructure:

**(a) OrderHub backend additions** (Lovable / Supabase):

- New `personal_access_tokens` table with hash-only storage, soft revoke, optional expiry, and forward-looking `scopes` column.
- New "Personal access tokens" page in OH web settings, owner-only, GitHub-style "shown once on creation" UX.
- New HTTP MCP route as a Supabase Edge Function (`/mcp`) that speaks JSON-RPC, validates `Authorization: Bearer <token>` against the hash table, scopes the request to the resolved user/org, and proxies to existing tool implementations.
- New MCP tool `get_plugin_status()` returning the canonical latest plugin version and changelog URL.

**(b) Plugin marketplace repo** — this repo, `richardcharnley0404/claude-orderhub-plugin`:

- Marketplace manifest (`.claude-plugin/marketplace.json`) listing the single `orderhub` plugin.
- Plugin manifest (`plugins/orderhub/.claude-plugin/plugin.json`) declaring the HTTP MCP server with bearer-header substitution and a sensitive `userConfig.api_token` field.
- Four skills (daily-briefing, pickup-queue, order-lookup, sales-metrics) each with a `SKILL.md` and references for output rules.
- Four slash commands (`/orderhub:daily`, `/orderhub:pickup`, `/orderhub:order`, `/orderhub:sales`) sharing instructions with their paired skills.

**Lab-owner install flow:**

1. `/plugin marketplace add richardcharnley0404/claude-orderhub-plugin`
2. `/plugin install orderhub`
3. Claude Code prompts for the access token (with deep link to OH's tokens page).
4. Owner pastes token. Plugin live.

**Update flow:**

- Pixfizz pushes commits, bumps `version` in `plugin.json`.
- Owner gets the update on next Claude Code session start (auto) or via `/plugin marketplace update`.
- Tokens, `userConfig` values, and per-user data in `${CLAUDE_PLUGIN_DATA}` survive the update; skills, commands, and the MCP URL are replaced.

## 3. Tool surface and skill/command mapping

Existing read tools kept as-is: `get_daily_briefing`, `get_inventory_status`, `get_order`, `get_order_statistics`, `get_orders_needing_attention`, `get_pending_pickups`, `get_performance_metrics`, `get_sales_summary`, `get_top_products`, `list_customers`, `list_film_scans`, `list_jobs`, `list_orders`, `search_orders`.

Existing write tool `update_job_status` remains callable but is not surfaced via a v1 skill. New tool `get_plugin_status()` returns `{ latest_version, changelog_url }`.

Four skills, four slash commands:

| Slash command | Skill | Tools orchestrated | Output |
|---|---|---|---|
| `/orderhub:daily` | `daily-briefing` | `get_daily_briefing`, `get_pending_pickups`, `get_orders_needing_attention`, `get_plugin_status` | One-screen card: today's revenue + delta vs yesterday, queue health, ready-for-pickup, count needing attention, top concern, plugin-update footer when applicable. |
| `/orderhub:pickup` | `pickup-queue` | `get_pending_pickups` | Three groups (ready-today / this-week / overdue), per-row order #, customer, total, days waiting; summary header. |
| `/orderhub:order <q>` | `order-lookup` | `search_orders` (fuzzy) → `get_order` | Order header (status, total, paid badge), line items, payment block, customer block, notes, suggested next action. |
| `/orderhub:sales [range]` | `sales-metrics` | `get_sales_summary`, `get_top_products`, `get_performance_metrics` | Revenue card + delta, top 5 products, performance metrics. Range parsing: `today`, `yesterday`, `7d`, `30d`, `mtd`, `qtd`, `ytd` (default `7d`). |

The remaining MCP tools stay accessible — Claude can call them when natural-language questions don't fit the four packaged workflows. v1 just doesn't ship curated narratives for them.

## 4. Personal access token (PAT) auth flow

**OH Settings → Personal access tokens UI:**

Owner-only page. Lists existing tokens (name, scopes, last used, created, revoke). "Generate new token" modal collects name, scopes (preset to `read:org` and locked in v1), optional expiry. Plaintext shown **once** on creation; never persisted.

**`personal_access_tokens` schema:**

```
id              uuid pk
user_id         uuid fk → auth.users
org_id          uuid                  -- snapshot of active org
name            text
token_hash      text unique           -- sha256(plaintext) hex
scopes          text[]                -- ['read:org'] in v1
last_used_at    timestamptz
created_at      timestamptz default now()
revoked_at      timestamptz           -- soft revoke
expires_at      timestamptz           -- nullable
```

Unique index on `token_hash`. RLS: owners can SELECT/UPDATE their own rows.

**Plugin manifest excerpt:**

```json
{
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
      "headers": { "Authorization": "Bearer ${user_config.api_token}" }
    }
  }
}
```

Token stored in OS-secure storage (macOS keychain, Windows credential manager, Linux libsecret), never in the plugin cache.

**MCP request validation per call:**

1. Read `Authorization: Bearer <token>`. Missing → 401 `unauthorized`.
2. `hash = sha256(token)`. Lookup. Not found / revoked / expired → 401 `invalid_token`.
3. Verify scope covers the requested operation. Mismatch → 403 `insufficient_scope`.
4. Update `last_used_at` (background, non-blocking).
5. Resolve `user_id`, `org_id`. All downstream queries org-scoped.
6. Audit row written: `mcp_audit_log` with `token_id`, `tool`, `params`, `at`.

Rotation = generate new token + re-paste via Claude Code's `userConfig` re-prompt. Revocation = `revoked_at = now()` from UI.

## 5. Plugin update mechanism

`plugin.json.version` is canonical. Pixfizz's release ritual: bump patch, mirror in `marketplace.json`, commit, tag.

**Daily-briefing footer logic** (in `SKILL.md`):

1. Call `get_daily_briefing` (main payload).
2. Call `get_plugin_status` → `{ latest_version, changelog_url }`.
3. Read installed version from `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json` via Read tool.
4. Compare semver. If `installed < latest`, append:
   ```
   📦 OrderHub plugin v{latest} available. Run /plugin update orderhub to refresh.
   ```
5. Rate-limited to once per terminal per day via `${CLAUDE_PLUGIN_DATA}/last_update_nudge.txt`.

**`get_plugin_status` server-side** reads a constant Pixfizz updates as part of release (e.g. an `app_settings` row). No GitHub API roundtrip.

**Backwards compatibility contract:**

- New tools added (never removed under same name without deprecation).
- Response shapes extended additively (new fields fine; rename/remove = breaking).
- `userConfig` extended additively.
- Hard breaks bump the major; the daily-briefing footer warns explicitly.

## 6. Repo + distribution layout

This repo:

```
claude-orderhub-plugin/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── orderhub/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── skills/
│       │   ├── daily-briefing/
│       │   │   ├── SKILL.md
│       │   │   └── references/output-format.md
│       │   ├── pickup-queue/SKILL.md
│       │   ├── order-lookup/SKILL.md
│       │   └── sales-metrics/
│       │       ├── SKILL.md
│       │       └── references/range-parsing.md
│       └── commands/
│           ├── daily.md
│           ├── pickup.md
│           ├── order.md
│           └── sales.md
├── docs/
│   └── design.md                     ← this file
├── CHANGELOG.md
├── README.md
└── LICENSE
```

OH backend additions (separate from this repo, lives in OrderHub / Lovable):

```
OrderHub (Lovable-managed Supabase)
├── supabase/migrations/              ← personal_access_tokens table + index + RLS
├── supabase/functions/mcp/index.ts   ← HTTP MCP route, bearer auth, audit log
└── (Lovable web app)
    └── Settings → Personal access tokens page (owner-only)
```

Why a separate repo from `ohpos`: different audiences, different release cadences, public visibility for marketplace fetch, cleaner permission model.

## 7. Testing & verification

**MCP edge function — vitest/deno test:**

| Test | Verifies |
|---|---|
| Missing Authorization header → 401 | Auth gate |
| Bad hash → 401 | Hash lookup |
| Revoked token → 401 | Soft-revoke |
| Expired token → 401 | Expiry |
| Out-of-scope tool → 403 | Scope guard (forward-compat) |
| Valid bearer + tool → success + audit row | Happy path + provenance |
| Cross-org user → cannot read other org | Multi-tenant safety |
| `get_plugin_status` returns expected shape | Update-nudge contract |

**Skills (LLM-driven):**

- Snapshot review with mock MCP responses; visual diff on changes.
- Tool-call assertions per skill (e.g. daily-briefing must call `get_daily_briefing` + `get_plugin_status`).
- Version-check footer fixtures (installed < latest → footer appears verbatim; equal → absent).
- Range parsing fixtures for `sales-metrics` (today / 7d / mtd / garbage input).

**Plugin-level smoke (manual, pre-release):**

1. Fresh Claude Code, no plugin.
2. `/plugin marketplace add` + `/plugin install`.
3. Token prompt → paste valid token.
4. Run all four slash commands; non-empty output each.
5. Bump version, push, update; verify new version installed and token survived.
6. Revoke token in OH; verify 401 surfaces clearly.

**Acceptance bar for v1 release:**

- Edge-function tests pass.
- Snapshot review of all four skills approved.
- Manual smoke checklist passes end-to-end.
- README + deep link to tokens page work.

## 8. Open questions / decisions captured

- **Scope label for v1**: `read:org`. Simple flat scope; v2 introduces `write:notes`, `write:inventory`, etc. (additive — old tokens never gain new scopes by default).
- **Token expiry default**: never. Soft 1-year nag in the UI; owners can opt in to shorter expiries per token.
- **Audit log location**: separate `mcp_audit_log` table (not `pos_audit_log`) — different actor model (PAT vs POS device) and different sensitivity profile.
- **Version pinning during alpha**: keep explicit `version` field from day one. No commit-SHA mode. Changelog discipline matters even before public launch.
- **Marketplace namespace**: `richardcharnley0404/claude-orderhub-plugin` for now; rename if/when Pixfizz creates a GitHub org.

---

*Pixfizz Ltd · OrderHub Claude Code Plugin · v1 spec lock 2026-05-10*
