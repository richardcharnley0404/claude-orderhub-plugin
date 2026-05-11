# Changelog

All notable changes to the OrderHub Claude Code plugin.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.2.0] — 2026-05-11

### Changed
- **Two-plugin install model** is now the recommended path for Cowork / Claude Desktop users. The OrderHub web app's API Integrations page (Integration Type = "Claude Desktop") generates a per-user "OrderHub Authenticator" `.plugin` file that registers the MCP connector. This marketplace plugin then runs alongside it, providing the auto-updating skills and slash commands. Architecture proven end-to-end empirically: connector persists across plugin uninstalls; marketplace plugin's skills route through the connector via `mcp__orderhub__<tool>`.
- README rewritten around the two-plugin model. Cowork / Claude Desktop path is now the Authenticator download flow; Claude Code path is unchanged (marketplace install + paste PAT).
- Plugin manifest: `userConfig.api_token` description clarified to be Claude Code-only; MCP URL no longer carries the `?token=${user_config.api_token}` interpolation experiment from 0.1.2 (Cowork never honoured `${user_config.X}` substitution; the experiment was inconclusive and removed).
- Plugin manifest: `mcpServers.orderhub.url` reverts to the bare endpoint without query params. Auth is via `Authorization: Bearer ${user_config.api_token}` header only (Claude Code resolves the interpolation; Cowork never reaches this code path because the Authenticator plugin's connector takes precedence).

### Fixed
- Empirical Cowork investigation confirmed: literal Bearer headers in plugin manifests are not forwarded to the MCP server by Cowork, and `${user_config.X}` substitution is Claude Code-only. The 0.2.0 architecture works around both by routing Cowork users through the OrderHub-generated Authenticator plugin instead.

## [0.1.2] — 2026-05-10

### Changed
- Test commit: added `?token=${user_config.api_token}` to MCP URL alongside the Bearer header. Probe to test whether Claude Code expanded `${user_config.X}` interpolation inside the URL field. Bearer header retained so auth kept working. Empirical result via Cowork connector panel: Cowork stored the URL with the literal placeholder (no substitution). Experiment retired in v0.2.0.

## [0.1.1] — 2026-05-10

### Added
- README rewritten to position the OrderHub MCP as the primary integration with three install paths: Claude Desktop, Claude.ai web, and Claude Code (the existing plugin). Claude Code is now framed as the developer / power-user channel rather than the only option.
- New `docs/claude-ai-project.md` — Claude.ai Project template with system instructions that recreate the Claude Code plugin's polished workflow rendering (daily briefing, pickup queue, order lookup, sales metrics) for users on Claude Desktop or Claude.ai web.

### Notes
- No code or skill changes — this is a docs-only release expanding the audience surface. The Claude Code plugin behaviour and version contract are unchanged.

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

### Fixed
- Daily-briefing rate-limit step now writes the marker file unambiguously after appending the footer (prevents the nudge from re-firing within the same day).
- Pickup-queue bucket logic now catches orders aged 7+ days even when `ready_date` is still in the future.
- Order-lookup falls back to `search_orders` when a bare order number doesn't resolve via `get_order` directly.
- Range-parsing reference no longer claims `to` is "always today" — past-period ranges now correctly described.
- Daily-briefing "no write tools" rule clarified to exclude only MCP write tools, not the filesystem `Write` tool used by the rate-limit step.
