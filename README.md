# Reverse Engineering Claude Agent SDK: Hidden Token Cost & Fix

Every message sent through Claude Agent SDK's V1 `query()` API costs 2-3% of your 5-hour quota — even for a simple "hello". The same conversation in Claude Code CLI costs <1% after the first message.

This research reverse-engineers the 12MB minified `cli.js` to find the root cause: **prompt cache invalidation**, not token injection itself. Each `query()` spawns a fresh Node.js process, runtime-injected `system-reminder` content gets reassembled slightly differently, breaking the cache prefix — causing the entire ~45k token message history to be rewritten at 125% cost every single time.

## Key Findings

| Scenario | Cache Efficiency | Cost per Message |
|----------|-----------------|------------------|
| Agent SDK V1 `query()` + resume | **25%** (stuck) | ~2-3% quota |
| CLI interactive (after 1st msg) | **95%+** | <0.5% quota |
| **Agent SDK V2 persistent session (our fix)** | **84%** (and rising) | <0.5% quota |

## Solution

Patch the SDK's V2 `unstable_v2_createSession()` API to accept full options (5 patch points in `SDKSession` constructor), keeping the `cli.js` process alive across messages. Cache accumulates instead of rebuilding.

## Read the Full Report

- [**Full research report (Chinese)**](./report.md) — complete reverse engineering findings, A/B test data, and implementation details
- [**Blog post version**](https://cablate.com/articles/reverse-engineer-claude-agent-sdk-hidden-token-cost) — same content on CabLate

## Diagrams

All diagrams are in the [`images/`](./images/) directory, created with Excalidraw.

| Diagram | Description |
|---------|-------------|
| [token-cost-comparison](./images/token-cost-comparison.webp) | SDK vs CLI cost per message |
| [sdk-call-flow](./images/sdk-call-flow.webp) | How Agent SDK calls work |
| [system-reminder-content](./images/system-reminder-content.webp) | What a system-reminder looks like |
| [tracking-table-lifecycle](./images/tracking-table-lifecycle.webp) | CLI vs SDK file tracking table lifecycle |
| [cache-prefix-break](./images/cache-prefix-break.webp) | Where cache prefix breaks |
| [ab-test-results](./images/ab-test-results.webp) | Phase 1 A/B test (failed) |
| [v1-vs-v2-architecture](./images/v1-vs-v2-architecture.webp) | V1 vs V2 architecture comparison |
| [v2-cache-efficiency](./images/v2-cache-efficiency.webp) | V2 cache efficiency over messages |

## Related Issues

- [anthropics/claude-code#9769](https://github.com/anthropics/claude-code/issues/9769) — Request for system-reminder toggle (open since 2025-10)
- [anthropics/claude-code#16021](https://github.com/anthropics/claude-code/issues/16021) — File modification reminders injected every message
- [anthropics/claude-code#17601](https://github.com/anthropics/claude-code/issues/17601) — 10,000+ hidden injections consuming 15%+ context
- [anthropics/claude-agent-sdk-typescript#188](https://github.com/anthropics/claude-agent-sdk-typescript/issues/188) — SDK defaults to 1h cache TTL (2x write cost)

## SDK Version

Research based on `@anthropic-ai/claude-agent-sdk` v0.2.76 / Claude Code v2.1.76 (March 2026).

## License

CC-BY-4.0
