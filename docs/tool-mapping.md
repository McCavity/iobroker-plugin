# Tool Mapping — backend-agnostic ioBroker MCP

The `iobroker-plugin` skills describe **goals**, not fixed tool calls, so they
work over either of the two ioBroker MCP backends. This table is the reference
the skills point at.

| Goal | McCavity/iobroker-mcp (SimpleAPI, stdio) | official ioBroker.mcp (Bluefox, HTTP/SSE) |
|---|---|---|
| Read a state's value + metadata | `read_state` | `get_states`, `get_object` |
| Read many states at once | `read_states_bulk` | `get_states` (multiple IDs) |
| Find states/objects by pattern | `search_objects` (IDs), `list_objects` (metadata) | `search_objects` (keyword/role/room) |
| Discover devices / rooms / functions | `list_objects` + client-side filter | `list_devices`, `list_rooms`, `list_functions` |
| Query historical values | `query_history` | `history_query` |
| Write a state | `write_state`, `toggle_state` | `set_state` |
| Create / update an object | — *(not exposed)* | `set_object` |
| Adapter / instance / host status | `health_check` (route ping) | `list_instances`, `list_hosts`, `system_info` |
| System logs | — *(not exposed)* | `get_logs`, `write_log` |
| File storage read/write | — *(not exposed)* | `read_file`, `write_file` |

## MCP resources (official adapter only)

The official adapter additionally exposes subscribable MCP **resources** — there
is no equivalent on the SimpleAPI-backed MCP (which is purely tool-based):

- `iobstate://<id>` — state value, ack, timestamp, quality (subscribable via SSE)
- `iobobject://<id>` — object definition
- `ioblog://all` / `ioblog://<source>` — log entries
- `iobfile://<adapter>/<path>` — files

## Semantic differences worth knowing

- **Pattern matching.** SimpleAPI = whole-string segment-anchored **glob**
  (`zigbee.0.*battery`; `*battery*` returns zero). The official adapter searches
  by **keyword/role/room** and offers structured `list_*` discovery instead.
  See `skills/find-state-by-pattern`.
- **`script.js.*` / `scenes.0.*` filtering** is a SimpleAPI behaviour — read by
  exact path. Not applicable the same way on the official adapter.
- **Write surface.** McCavity/iobroker-mcp is read-heavy with two write tools
  (`write_state`, `toggle_state`). The official adapter adds object/file/log
  writes — all permission-gated.
- **WARN-spam on offline `getState`** is **server-side ioBroker behaviour** and
  applies through *either* backend — hence the `available`-first guard in
  `skills/battery-check-pattern` is backend-independent.

## Status of the official adapter (as of 2026-06)

`ioBroker/ioBroker.mcp` (author @GermanBluefox) is **experimental**: v0.1.4
(2026-05-28), not yet published to npm, Streamable-HTTP/SSE transport, runs as an
ioBroker adapter (standalone web server or inside an existing web adapter), with
ACL enforcement. Track it before migrating — when it reaches npm/beta, re-point
`.mcp.json` at an HTTP transport and the skills carry over via this table.
