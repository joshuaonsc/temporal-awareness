---
name: temporal-awareness
description: >
  Give Claude access to the current wall-clock time — a capability it does not natively
  have (Claude only knows the date the conversation started — not the time of day, and
  in long conversations not even the current date). Use this skill whenever the user
  asks what time it is, how long it has been since their last message, or anything
  where knowing the actual time would improve the response. This includes implicit time
  needs the user may not phrase as a time question: "is the market open right now",
  "is it too late to call the West Coast", "should I grab lunch", "good morning" (is it
  actually morning?), the user returning after an apparent break ("I'm back", "sorry,
  got pulled away"), or any scheduling, duration, or elapsed-time context. Also use
  when the user has instructed Claude to timestamp every turn. If in doubt about
  whether time context would help, consult this skill — an unnecessary check costs one
  fast tool call, while a missed one means answering as if no time has passed.
---

# Temporal Awareness

Claude's system context provides the **date the conversation started**, but not the
time of day — and in a long-running conversation, not even necessarily today's date.
This skill closes both gaps using two methods:

- **Primary: bash_tool** (requires Code Execution) — lower latency, no rate limits,
  returns epoch seconds for easy delta arithmetic.
- **Fallback: MCP time tool** (requires a connected time server) — works without Code
  Execution, but adds a network round-trip and returns ISO 8601 without epoch values.

## Method selection

1. If bash_tool is available (Code Execution is on), use it. Stop here.
2. If bash_tool is unavailable, use tool_search to check for a time-returning MCP tool
   (query: "current time"). MCP tools are deferred, so this check is required before
   first use in each conversation — never assume the server is still connected.
3. If neither is available, tell the user that time-checking requires either Code
   Execution or a connected time server — and offer the fix: the free, authless
   CurrentTimeUTC server can be added under Settings → Connectors → Add Custom
   Connector, URL `https://a.currenttimeutc.com/mcp`.

Once a method is chosen for a conversation, prefer that same method for all subsequent
time checks in that conversation. Mixing sources introduces clock drift between the
container and the MCP server, which corrupts elapsed-time deltas.

## Primary: bash_tool

Run a single command that returns both a human-readable timestamp and epoch seconds
(for elapsed-time arithmetic):

```bash
date '+%Y-%m-%d %H:%M:%S %Z (epoch:%s)'
```

If the user's timezone is known (from location, memory, or explicit mention), convert:

```bash
TZ='America/New_York' date '+%Y-%m-%d %H:%M:%S %Z (epoch:%s)'
```

If the timezone is unknown, resolve it from the user's location — from system context,
memory, or a location-returning tool if one is available. Fall back to UTC if no
timezone can be determined.

To find an IANA identifier you are unsure of (avoid `timedatectl` — it is missing on
macOS and broken in containers without systemd, which includes most code-execution
sandboxes):

```bash
find -L /usr/share/zoneinfo -type f | sed 's|/usr/share/zoneinfo/||' | grep -i '<search_term>'
```

### Elapsed time via bash_tool

Prior timestamps persist in conversation history, so no external state is needed:
subtract a previous turn's epoch value from the current one and convert to
human-readable units (611 seconds apart → "about 10 minutes").

## Fallback: MCP time tool

The expected tools from a connected time server are:

- `get_utc_time` — returns current UTC time in ISO 8601 format.
- `convert_time` — converts between IANA timezones (params: `time`, `from_tz`, `to_tz`).
- `list_timezones` — returns supported IANA timezone identifiers.

The server is a free third-party service with no SLA — it may rate-limit or disappear
entirely. For always-on per-turn time checks, bash_tool is strongly preferred.

### Elapsed time via MCP

The MCP path returns ISO 8601 timestamps without epoch seconds. To compute deltas,
parse the ISO timestamps from conversation history and subtract. This is less clean
than epoch arithmetic but functional.

## Presenting time in responses

How to surface time depends on why it was checked:

- **Explicit time question** ("what time is it?"): State the time directly.
- **Implicit time need** ("is the market open?"): Check the time, use it to answer the
  actual question, and mention the current time as supporting context.
- **Silent context** ("good morning!"): If the time contradicts the greeting or adds
  useful color, note it naturally ("it's actually 2 AM where you are — burning the
  midnight oil?"). Otherwise, just respond normally.
- **Always-on mode**: If a user preference or memory instructs per-turn time checks,
  check the time at the start of each response. Whether to display it depends on the
  preference wording — if the user asked for visible timestamps, include a brief line
  at the top; if they asked only for time awareness, use the result silently and
  surface it only when relevant. If the user wants this behavior but has no preference
  set yet, suggest one, e.g.: "Always check the current time at the start of each
  response using the temporal-awareness skill."

## General limitations

The timestamp reflects when Claude retrieves the time during response generation, not
the moment the user pressed send — expect seconds of skew, and neither the container
clock (bash_tool) nor the MCP server guarantees sub-second NTP accuracy. The practical
consequence: round elapsed times rather than implying precision the clocks don't have
("about 10 minutes", not "10 minutes 11 seconds").

<!-- Fallback server: CurrentTimeUTC by jairampatel
  https://github.com/jairampatel/currenttimeutc-mcp -->
