# temporal-awareness

A Claude skill that gives Claude access to the current wall-clock time — a capability it does not natively have.

Claude knows the date the conversation started, but not the time of day — and in a long conversation, not even necessarily today's date. This skill closes both gaps using two methods:

- **Primary: Code Execution** (`bash_tool`) — runs `date` locally in the container. Lower latency, no rate limits, returns epoch seconds for easy elapsed-time arithmetic.
- **Fallback: MCP time server** — calls a connected time server via MCP. Works when Code Execution is off, at the cost of a network round-trip.

## Install

Download [`temporal-awareness.skill`](./temporal-awareness.skill) and add it in Claude:

**Settings → Profile → Skills → Add skill**

The skill works immediately with Code Execution enabled — no other setup needed.

## MCP fallback setup (optional)

To enable the fallback for conversations without Code Execution:

1. Go to **Settings → Connectors → Add Custom Connector**
2. Name: `CurrentTimeUTC`
3. URL: `https://a.currenttimeutc.com/mcp`
4. Click **Add**

This connects to [CurrentTimeUTC](https://github.com/jairampatel/currenttimeutc-mcp), a free, authless MCP time server. The skill will automatically detect and use it when Code Execution is unavailable.

## Always-on mode

By default, the skill triggers on time-related queries. To have Claude check the time on every turn, add a User Preference:

**Settings → Profile → User preferences**

> Always check the current time at the start of each response using the temporal-awareness skill.

## What it does

- Checks current time in any IANA timezone
- Tracks elapsed time between conversation turns
- Adapts presentation based on context — states the time directly for explicit questions, uses it silently for implicit needs (e.g., "is the market open?"), and suppresses it when irrelevant
- Falls back gracefully across methods: Code Execution → MCP → inform user

## Skill structure

```
temporal-awareness/
└── SKILL.md          # Skill instructions (the only file Claude reads)
```

## License

[MIT](./LICENSE)
