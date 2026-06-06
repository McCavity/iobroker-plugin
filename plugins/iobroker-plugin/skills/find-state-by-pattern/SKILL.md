---
name: find-state-by-pattern
description: Use when you need to locate an ioBroker state or object ID and aren't sure of the exact name — "wie heißt der State/Datenpunkt für X?", "finde den State der Couch-Lampe", "welche States hat das Zigbee-Gerät?", or before reading/writing a state whose ID you are guessing. Encodes the SimpleAPI glob-pattern pitfalls (whole-string, segment-anchored, script.js filtered).
---

# Find an ioBroker State by Pattern

Locate the right state/object ID without guessing — and without falling into
the SimpleAPI pattern traps that silently return zero matches.

## When to Use

Trigger when the exact state ID is unknown and you are about to search, read, or
write: "wie heißt der Datenpunkt für…", "finde den State…", "welche States hat
…", "gibt es einen State für …". Always run this **before** writing to a guessed
ID.

## How to Use

1. **Use segment-anchored glob patterns.** ioBroker (SimpleAPI) matches the
   pattern as a **whole-string glob against the full state ID**. Patterns that
   work: `zigbee.0.*`, `sonoff.*`, `*.POWER`, `zigbee.0.*battery`.

2. **In-string substring wildcards do NOT work.** `*battery*` returns **zero**
   matches on the SimpleAPI backend — it is a glob, not a substring search. Use
   a segment-anchored pattern (`zigbee.0.*battery`) or fetch a broader listing
   and filter client-side on `common.name`.

3. **Bare strings are auto-globbed.** On McCavity/iobroker-mcp,
   `search_objects("smartcontrol")` is rewritten to `smartcontrol.*` before the
   call — check the returned `effective_pattern` to see exactly what was sent.
   *(Backend note: the official ioBroker.mcp `search_objects` matches by
   keyword, role or room instead — different semantics; see below.)*

4. **`script.js.*` and `scenes.0.*` are filtered from listings.** Script and
   scene objects do not appear in `/objects` results (SimpleAPI, likely a
   payload-size guard). To read one, use the **exact path** with a read-state
   call: `read_state("script.js.scenes.lighting.smart-switches")`. On
   McCavity/iobroker-mcp, `list_objects` returns a `hint` field reminding you of
   this when a `script.js` pattern yields zero.

5. **Narrow by friendly name.** List a namespace, then match the device by its
   `common.name` field rather than guessing the numeric/MAC segment of the ID.

## Backend note (pattern semantics differ)

- **McCavity/iobroker-mcp (SimpleAPI):** whole-string segment-anchored glob, as
  above. `search_objects` = IDs only; `list_objects` = compact metadata.
- **official ioBroker.mcp:** `search_objects` matches by **keyword, role or
  room**, and there are structured discovery tools — `list_devices`,
  `list_rooms`, `list_functions`. On that backend prefer those for grouping
  instead of crafting globs.

Full mapping in `docs/tool-mapping.md`.

## Example Output

```
Gesucht: State der „Wohnzimmer Couch"-Lampe.

Pattern `zigbee.0.*` → 142 States. Gefiltert auf common.name ~ "Couch":
- zigbee.0.00158d0001abcd.state   (on/off)   common.name "Wohnzimmer Couch"
- zigbee.0.00158d0001abcd.brightness
- zigbee.0.00158d0001abcd.available

Zum Schalten: write_state("zigbee.0.00158d0001abcd.state", true).
```

## Anti-Patterns

- Do NOT use `*substring*` patterns — they return zero on the SimpleAPI backend.
- Do NOT expect `script.js.*` / `scenes.0.*` in listings — read them by exact path.
- Do NOT write to a guessed ID — confirm it exists via a find first.
- Do NOT assume the device name is in the ID — match on `common.name`.
