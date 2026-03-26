# Claude Code Research

Independent research on Claude Code internals, Claude Agent SDK, and related tooling. All findings are based on reverse engineering the publicly distributed npm packages.

Each report includes both English and Chinese versions.

## Reports

| # | Topic | Description |
|---|-------|-------------|
| 1 | [Agent SDK Cache Invalidation](./reports/agent-sdk-cache-invalidation/) | Why Agent SDK V1 `query()` costs 3-10x more than CLI per message — prompt cache invalidation caused by process-per-call architecture. Fix: V2 persistent session + SDK patch. |

## Related Issues

- [anthropics/claude-code#9769](https://github.com/anthropics/claude-code/issues/9769) — Request for system-reminder toggle (open since 2025-10)
- [anthropics/claude-code#16021](https://github.com/anthropics/claude-code/issues/16021) — File modification reminders injected every message
- [anthropics/claude-code#17601](https://github.com/anthropics/claude-code/issues/17601) — 10,000+ hidden injections consuming 15%+ context
- [anthropics/claude-agent-sdk-typescript#188](https://github.com/anthropics/claude-agent-sdk-typescript/issues/188) — SDK defaults to 1h cache TTL (2x write cost)

## SDK Version Baseline

Research is based on `@anthropic-ai/claude-agent-sdk` v0.2.76 / Claude Code v2.1.76 (March 2026). Findings may change with future SDK updates.

## License

CC-BY-4.0
