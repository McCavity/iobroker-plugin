---
name: battery-check-pattern
description: Use when checking battery levels of ioBroker / Zigbee devices or hunting weak batteries — "welche Batterien sind schwach?", "Batteriestatus", "welche Sensoren muss ich bald tauschen?", "Batterie-Check". Applies the available=false guard (skip the read for offline devices to avoid ioBroker getState WARN-spam) and numeric validation before the threshold compare.
---

# Battery Check (defense-in-depth)

Report weak batteries across ioBroker devices **without** triggering the
server-side `getState` WARN-spam and **without** letting renamed/missing states
slip through as "fine".

## When to Use

Trigger when the user asks about battery state: "welche Batterien sind schwach",
"Batteriestatus", "welche brauchen bald neue Batterien", "Batterie-Check". Also
used as a sub-step of `diagnose-device`.

## How to Use

1. **Find the battery states.** Use a segment-anchored pattern with the dot
   before the leaf — `zigbee.0.*.battery` or `*.battery` (the glued
   `zigbee.0.*battery` returns zero, see `find-state-by-pattern`) — or list
   devices and filter. Unsure → `find-state-by-pattern`.

2. **Check ONLINE / `available` FIRST — skip the read if offline.** For each
   device, read its `available` / `.info.connection` state. If it is offline,
   **do not read the battery state at all.** Reason (learned 2026-05-27):
   reading a `battery` state on a long-offline device emits an ioBroker
   `getState` **WARN on the server**, even if you handle the `null` return
   defensively — the WARN fires at the adapter read, not at your comparison.
   Skipping the read avoids both the WARN-spam and a stale/misleading value.
   Put offline devices in a separate **"offline (übersprungen)"** bucket.

3. **Validate numeric before comparing.** Only compare if the value is actually
   a number (`typeof val === "number"`). The trap (Eintracht-Logo, 2026-05-27):
   `undefined > 20` evaluates to `false` in JS — no error, no `NaN`, it silently
   passes. A renamed or missing state (e.g. `battery` absent, or
   `illuminance` → `illuminance_raw` after an adapter re-pair) would otherwise
   slip through as "battery fine". A non-number → flag as **"unbekannt"**, not OK.

4. **Thresholds.** **critical ≤ 10 %**, **warning ≤ 20 %**, else OK.

5. **Report device NAMES, not state IDs.** Use `common.name`; the raw
   `zigbee.0.00158d…` ID is meaningless to the user.

## Backend note

The WARN-spam is **ioBroker server-side behaviour** — it applies regardless of
which MCP backend you read through. The `available`-first guard is universal.
Read tools: `read_states_bulk` / `read_state` (McCavity) · `get_states`
(official). Full mapping in `docs/tool-mapping.md`.

## Example Output

```
Batterie-Check (28 Geräte mit battery-State):

🔴 Kritisch (≤10%):
  - Tür-Sensor Keller        7%
🟠 Warnung (≤20%):
  - Bewegungsmelder Bad      18%
  - Fernbedienung Couch      19%
⚪ Unbekannt (kein numerischer Wert):
  - Rauchmelder Flur         (State fehlt — nach Re-Pair prüfen)
⏭️  Offline (übersprungen):  3 Geräte
✅ OK:                        21 Geräte

Nächster Schritt: CR2032/CR2450 für Keller-Sensor besorgen.
```

## Anti-Patterns

- Do NOT read the battery state of an offline device — it triggers a server-side `getState` WARN and returns a stale value.
- Do NOT compare a non-number against a threshold — `undefined > N` is silently `false`.
- Do NOT treat a missing battery state as 100 % / OK — flag it as unknown.
- Do NOT report raw state IDs — use the device's friendly name.
