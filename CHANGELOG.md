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

### Fixed
- Daily-briefing rate-limit step now writes the marker file unambiguously after appending the footer (prevents the nudge from re-firing within the same day).
- Pickup-queue bucket logic now catches orders aged 7+ days even when `ready_date` is still in the future.
- Order-lookup falls back to `search_orders` when a bare order number doesn't resolve via `get_order` directly.
- Range-parsing reference no longer claims `to` is "always today" — past-period ranges now correctly described.
- Daily-briefing "no write tools" rule clarified to exclude only MCP write tools, not the filesystem `Write` tool used by the rate-limit step.
