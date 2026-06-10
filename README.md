# temporal-awareness

Wall-clock and elapsed-time awareness for Claude.

Claude knows the date the conversation started, but not the time of day — and in a long conversation, not even necessarily today's date. This skill closes both gaps using two methods:

- **Primary: Code Execution** (`bash_tool`) — runs `date` locally. Fast, no rate limits, returns epoch seconds for elapsed-time arithmetic.
- **Fallback: MCP time server** — calls a connected time server when Code Execution is off, at the cost of a network round-trip. If neither method is available, Claude explains how to enable one.

## What it does

- Checks the current time in any IANA timezone
- Tracks elapsed time between conversation turns
- Adapts presentation to context — states the time for explicit questions, uses it silently for implicit needs (e.g., "is the market open?"), and suppresses it when irrelevant

## Install

Download [`temporal-awareness.skill`](./temporal-awareness.skill) and add it in Claude:

**Settings → Profile → Skills → Add skill**

The package is a single plain-text file (`SKILL.md`); with Code Execution enabled it works immediately — no other setup needed.

## MCP fallback setup (optional)

To enable the fallback for conversations without Code Execution:

1. Go to **Settings → Connectors → Add Custom Connector**
2. Name: `CurrentTimeUTC`
3. URL: `https://a.currenttimeutc.com/mcp`
4. Click **Add**

This connects to [CurrentTimeUTC](https://github.com/jairampatel/currenttimeutc-mcp), a free, authless MCP time server; the skill detects and uses it automatically.

## Always-on mode

By default, the skill triggers on time-related queries. To have Claude check the time on every turn, add a User Preference:

**Settings → Profile → User preferences**

> Always check the current time at the start of each response using the temporal-awareness skill.

## License

[MIT](./LICENSE)
[MIT](LICENSE) © 2026 [joshuaonsc](https://github.com/joshuaonsc)
