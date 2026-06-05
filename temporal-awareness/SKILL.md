---
name: temporal-awareness
description: >
  Give Claude access to the current wall-clock time — a capability it does not natively
  have (Claude only knows the date, not the time of day). This skill provides time
  access via Code Execution and/or MCP server tools. Use this skill whenever the
  user asks what time it is, how long it has been since their last message, or anything
  where knowing the actual time would improve the response. This includes implicit time
  needs the user may not phrase as a time question: "is the market open right now",
  "is it too late to call the West Coast", "should I grab lunch", "good morning" (is it
  actually morning?), or any scheduling, duration, or elapsed-time context. Also use when
  the user has instructed Claude to timestamp every turn. If in doubt about whether time
  context would help, consult this skill — the check is cheap.
---

# Temporal Awareness

Claude's system context provides the current **date** but not the time of day. This
skill closes that gap using two methods:

- **Primary: bash_tool** (requires Code Execution) — lower latency, no rate limits,
  returns epoch seconds for easy delta arithmetic.
- **Fallback: MCP time tool** (requires a connected time server) — works without Code
  Execution, but adds a network round-trip and returns ISO 8601 without epoch values.

## Method selection

1. If bash_tool is available (Code Execution is on), use it. Stop here.
2. If bash_tool is unavailable, use tool_search to check for a time-returning MCP tool
   (query: "current time"). If found, use it.
3. If neither is available, tell the user that time-checking requires either Code
   Execution or a connected time server, and cannot proceed.

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

If the timezone is unknown, check whether a find_location tool is available to resolve
it from the user's location. Fall back to UTC if no timezone can be determined.

Common timezone identifiers: `America/New_York`, `America/Chicago`, `America/Denver`,
`America/Los_Angeles`, `Europe/London`, `Asia/Tokyo`. To search for others:

```bash
timedatectl list-timezones | grep -i '<search_term>'
```

### Elapsed time via bash_tool

The epoch value from the combined command makes delta computation trivial:

1. At the start of the current response, run the combined date command.
2. Look back through conversation history for a prior epoch value from a previous
   turn's bash_tool output.
3. Compute the delta in seconds, then convert to human-readable units.

**Example:**
```
Previous turn: 1749097200
Current turn:  1749097811
Delta: 611 seconds → ~10 minutes 11 seconds
```

No external state is needed — prior timestamps persist in conversation history.

## Fallback: MCP time tool

If bash_tool is unavailable, check for a connected MCP time server via tool_search
(query: "current time" or "UTC time"). The expected tools are:

- `get_utc_time` — returns current UTC time in ISO 8601 format.
- `convert_time` — converts between IANA timezones (params: `time`, `from_tz`, `to_tz`).
- `list_timezones` — returns supported IANA timezone identifiers.

MCP tools are deferred and must be loaded via tool_search before first use in each
conversation. Do not assume they are available without checking — the user may have
disconnected the server.

### Elapsed time via MCP

The MCP path returns ISO 8601 timestamps without epoch seconds. To compute deltas,
parse the ISO timestamps from conversation history and subtract. This is less clean
than epoch arithmetic but functional.

### MCP-specific limitations

- **Third-party dependency.** The MCP server is a free service with no SLA. It may
  become unavailable at any time. If a tool_search returns no time-related tools,
  the fallback is simply not available.
- **Network latency.** Each call is a round-trip to an external server, adding
  measurable latency compared to a local `date` command.
- **Rate limits.** Free MCP servers may rate-limit under heavy usage. For always-on
  per-turn time checks, bash_tool is strongly preferred.

## Presenting time in responses

How to surface time depends on why it was checked:

- **Explicit time question** ("what time is it?"): State the time directly.
- **Implicit time need** ("is the market open?"): Check the time, use it to answer the
  actual question, and mention the current time as supporting context.
- **Silent context** ("good morning!"): If the time contradicts the greeting or adds
  useful color, note it naturally ("it's actually 2 AM where you are — burning the
  midnight oil?"). Otherwise, just respond normally.
- **Always-on mode** (via user preference): Check the time at the start of each
  response. Whether to display it depends on the preference wording — if the user
  asked for visible timestamps, include a brief line at the top; if they asked only
  for time awareness, use the result silently and surface it only when relevant.

## General limitations

- **Offset from send time.** The timestamp reflects when Claude retrieves the time
  during response generation, not the moment the user pressed send. Expect seconds of
  skew; under high load, potentially more.
- **Clock accuracy.** Both the container clock (bash_tool) and the MCP server depend
  on their respective NTP sync. Neither is guaranteed to sub-second precision.

## Automatic per-turn usage

This skill triggers on time-related queries by default. For automatic timestamping on
every turn regardless of topic, the user should add a User Preference or memory edit
such as: "Always check the current time at the start of each response using the
temporal-awareness skill."

<!-- Reference: MCP fallback server
  CurrentTimeUTC by jairampatel — https://github.com/jairampatel/currenttimeutc-mcp
  Remote endpoint: https://a.currenttimeutc.com/mcp (Streamable HTTP, authless, free)
  PulseMCP listing: https://www.pulsemcp.com/servers/currenttimeutc -->
