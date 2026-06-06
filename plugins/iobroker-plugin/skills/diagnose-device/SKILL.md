---
name: diagnose-device
description: Use when a smart-home device seems unresponsive, stuck, or "dead" in ioBroker — "warum reagiert die Lampe/der Sensor/die Steckdose X nicht?", "Gerät X ist offline?", "warum schaltet Y nicht mehr?", Zigbee/Tasmota/Shelly troubleshooting. Walks an online → battery → history → adapter-health checklist over an ioBroker MCP server (backend-agnostic).
---

# Diagnose an ioBroker Device

Systematically work out *why* a device is not responding, in an order that
avoids the classic false conclusions (stale value mistaken for a live reading,
one device blamed when the whole adapter is down).

## When to Use

Trigger when the user reports a device misbehaving: "warum reagiert X nicht",
"Gerät tot", "Sensor liefert nichts", "Steckdose schaltet nicht", "Lampe geht
nicht an". Also use proactively when another skill (e.g. `battery-check-pattern`)
surfaces an offline device the user then asks about.

## How to Use

Run the checklist **in order** and stop drilling as soon as a level explains the
fault. The order matters — each step rules out a class of cause.

1. **Locate the device's states.** If you don't know the exact namespace, hand
   to `find-state-by-pattern`. Use a segment-anchored pattern
   (`zigbee.0.*`, `sonoff.*`) and match the device by its `common.name`.

2. **Online / reachability FIRST.** Read the device's `available` /
   `.info.connection` / `link_quality` state. If it reports offline
   (`available=false`, `connection=false`) → **that is the headline.** Do NOT
   keep reading **value** states (they are stale and will mislead), and **skip
   the battery read** (see step 3 / `battery-check-pattern` — reading a battery
   state on an offline device triggers a server-side `getState` WARN). You MAY
   still read the **last-change timestamp (`lc`) or history of the reachability
   state itself** to report *when* the device was last seen — that is exactly
   what step 4 does (on the reachability state, not on the stale value states).
   Then jump to step 5 to check whether it is just this device or the whole adapter.

3. **Battery (only if battery-powered AND online).** Delegate to
   `battery-check-pattern` — it carries the `available`-guard and the numeric
   validation. A flat/critical battery is a common silent-death cause for Zigbee
   sensors.

4. **History — when did it last report?** Read the last-change timestamp (`lc`)
   of a representative state, or query history over the last 24–48 h. A flat
   line or no recent points = the device died silently (vs. a value that is
   simply unchanged because nothing happened).

5. **Adapter health.** Is the adapter instance alive? Check
   `system.adapter.<adapter>.<n>.alive` and `.connected`/`.info.connection`, and
   ping the server. **If the adapter is down, many devices look dead at once** —
   restart the adapter, not the device.

6. **Synthesize + recommend.** Name the level the fault sits at — *device*
   (re-pair, move closer, swap battery), *adapter* (restart instance), *battery*
   (swap), or *stale-state-only* (nothing actually wrong). Give one concrete
   next action.

## Backend-agnostic tool notes

Name the goal, not a fixed tool. Equivalents on the two supported backends
(full table in `docs/tool-mapping.md`):

| Goal | McCavity/iobroker-mcp (SimpleAPI) | official ioBroker.mcp |
|---|---|---|
| Read a state | `read_state`, `read_states_bulk` | `get_states` |
| Find states | `search_objects`, `list_objects` | `search_objects`, `list_devices` |
| History | `query_history` | `history_query` |
| Adapter / server health | `health_check` | `list_instances`, `system_info`, `get_logs` |

## Example Output

```
Diagnose „Bewegungsmelder Flur" (zigbee.0.00158d000XXXXX):

1. Online: ❌ available=false seit lc 2026-06-03 19:12 → Gerät ist offline.
2. Battery: übersprungen (offline — Read würde WARN auslösen).
3. Historie: letzter Datenpunkt 03.06. 19:12, danach Stille.
4. Adapter zigbee.0: alive ✓, connection ✓ → nur DIESES Gerät ist weg, nicht der Adapter.

→ Wahrscheinlich Batterie leer oder außer Funkreichweite. Nächster Schritt:
  Batterie prüfen/tauschen, dann Re-Pair (zigbee.0 Pairing-Modus).
```

## Handoff

- Unknown state ID → `find-state-by-pattern`
- Battery suspected → `battery-check-pattern`

## Anti-Patterns

- Do NOT read value states before the online check — a stale value reads as a live one.
- Do NOT read the battery state of an offline device — it triggers a server-side `getState` WARN and returns a stale/empty value.
- Do NOT blame the device before ruling out an adapter-down (one restart can fix "ten dead devices").
- Do NOT paste raw JSON at the user — report device names and the level the fault sits at.
