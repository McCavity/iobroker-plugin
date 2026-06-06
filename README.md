# iobroker-plugin

Claude-Code- und Codex-Plugin, das einen [ioBroker](https://www.iobroker.net/)-MCP-Server um drei Skills ergänzt, die *erklären wann* die ioBroker-Tools greifen — für Geräte-Diagnose, State-Suche und Batterie-Checks.

Das Plugin ist die dritte Iteration des Unit-3-Patterns aus dem [Hugging Face Context Course](https://huggingface.co/learn/context-course) (nach [paperless-bulk-plugin](https://github.com/McCavity/paperless-bulk-plugin)): *Skills beschreiben wie und wann, MCP-Server liefern was — das Plugin bringt beide so zusammen, dass Discovery automatisch greift, ohne dass der Nutzer den MCP explizit aufruft.*

## Backend-agnostisch

Anders als die Geschwister-Plugins ist dieses **nicht an einen bestimmten MCP-Server gebunden**. Die Skills beschreiben *Ziele* (State lesen, Historie abfragen, Adapter-Health prüfen) und nennen konkrete Tools nur als Beispiele — aus **beiden** unterstützten Backends:

- **McCavity/[iobroker-mcp](https://github.com/McCavity/iobroker-mcp)** — FastMCP-Wrapper um den **SimpleAPI**-Adapter, stdio. Stabil, im produktiven Einsatz.
- **offiziell [ioBroker/ioBroker.mcp](https://github.com/ioBroker/ioBroker.mcp)** (GermanBluefox) — ioBroker-**Adapter**, Streamable-HTTP/SSE. Zukunftslinie, Stand 2026-06 noch experimentell (v0.1.4, nicht auf npm).

Die vollständige Tool-Entsprechung steht in [`docs/tool-mapping.md`](docs/tool-mapping.md). Ein späterer Backend-Wechsel kostet damit nur einen `.mcp.json`-Swap, keine Skill-Umschreibung.

## Repo-Layout

Dieses Repo dient gleichzeitig als **Marketplace** und enthält **ein Plugin**:

```
iobroker-plugin/                       # Marketplace-Root
├── .claude-plugin/
│   └── marketplace.json               # Marketplace-Catalog
├── docs/
│   └── tool-mapping.md                # Backend-A ↔ Backend-B Tool-Mapping
└── plugins/
    └── iobroker-plugin/               # Plugin-Verzeichnis
        ├── .claude-plugin/plugin.json # Claude-Code-Manifest
        ├── .codex-plugin/plugin.json  # Codex-Manifest
        ├── .mcp.json                  # MCP-Backend-Referenz (Template, beide Backends dokumentiert)
        └── skills/...
```

## Bundled Skills

| Skill | Trigger | Workflow-Kern |
|---|---|---|
| `diagnose-device` | „warum reagiert X nicht?", „Gerät tot?" | online → battery → history → adapter-health, in dieser Reihenfolge (verhindert Fehlschlüsse) |
| `find-state-by-pattern` | „wie heißt der State für X?" | SimpleAPI-Glob-Fallen (whole-string, segment-anchored, `script.js`/`scenes.0` gefiltert) |
| `battery-check-pattern` | „welche Batterien sind schwach?" | `available=false`-Guard gegen `getState`-WARN-Spam + numerische Validierung vor Schwellvergleich |

Die Skills sind in den Lerneinträgen aus Hennings KI-OS verwurzelt (Glob-Pattern-Falle 2026-05-25, `available`-Guard + `undefined > N === false`-Falle 2026-05-27).

## MCP-Voraussetzung

Das Plugin enthält **keinen Server-Code** und registriert standardmäßig **keinen** MCP-Server — es ist **skills-only** und nutzt die ioBroker-MCP, die du **bereits registriert** hast (z.B. ein user-scope `iobroker`-Eintrag in `~/.claude.json`). So kann die Installation nie doppelt registrieren oder an unset Env-Vars scheitern.

Noch keine ioBroker-MCP registriert? Wähle ein Backend und kopiere eines der Templates aus [`.mcp.json`](plugins/iobroker-plugin/.mcp.json) in einen Top-Level-`mcpServers`-Block (entweder die Plugin-`.mcp.json` ODER dein user-scope `~/.claude.json`):

### Backend A — McCavity/iobroker-mcp (stdio)

Path-launched (venv-Python + `server.py`), Credentials aus einer `.env` neben `server.py`:

```bash
git clone git@github.com:McCavity/iobroker-mcp.git
cd iobroker-mcp && python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # IOBROKER_URL (+ optional IOBROKER_USER/PASS) setzen
```

```json
{ "mcpServers": { "iobroker": { "type": "stdio",
  "command": "/abs/pfad/iobroker-mcp/.venv/bin/python",
  "args": ["/abs/pfad/iobroker-mcp/server.py"], "env": {} } } }
```

### Backend B — offizieller ioBroker.mcp-Adapter (HTTP/SSE)

Adapter in ioBroker installieren + Instanz starten:

```json
{ "mcpServers": { "iobroker": { "type": "http", "url": "http://<host>:<port>/mcp" } } }
```

> Stand 2026-06 experimentell (v0.1.4, kein npm) — erst evaluieren, dann migrieren.

## Installation in Claude Code

Lokal (Repo gecloned):

```text
/plugin marketplace add ~/git/projects/own/iobroker-plugin
/plugin install iobroker-plugin@mccavity
```

Direkt vom GitHub:

```text
/plugin marketplace add McCavity/iobroker-plugin
/plugin install iobroker-plugin@mccavity
```

Marketplace-Name ist `mccavity` (gemäß `.claude-plugin/marketplace.json`).

## Installation in Codex

```bash
mkdir -p ~/.codex/plugins
cp -R /abs/pfad/iobroker-plugin/plugins/iobroker-plugin ~/.codex/plugins/iobroker-plugin
```

Dann `~/.agents/plugins/marketplace.json` um diesen Eintrag erweitern und Codex neu starten.

## Skill-Discovery — wie sie wirkt

Mit dem Plugin installiert wird folgendes möglich, **ohne dass der Nutzer den MCP explizit erwähnt**:

- „Warum reagiert der Flur-Bewegungsmelder nicht mehr?" → `diagnose-device` triggert, läuft die online→battery→history→adapter-Checkliste.
- „Wie heißt der Datenpunkt für die Couch-Lampe?" → `find-state-by-pattern` umschifft die Glob-Fallen.
- „Welche Batterien sind schwach?" → `battery-check-pattern` mit `available`-Guard.

Ohne das Plugin: der Nutzer muss daran denken, dass es eine ioBroker-MCP gibt, und die Tools selbst orchestrieren.

## Wartung

- Versionierung folgt [Semantic Versioning](https://semver.org/lang/de/).
- Skills sind atomar — neue Workflows kommen als neue Skill-Verzeichnisse rein, nicht als Erweiterung existierender Skills.
- Bei MCP-Tool-Updates (neue Tools, geänderte Signaturen): [`docs/tool-mapping.md`](docs/tool-mapping.md) + die betroffene SKILL.md aktualisieren, Version bumpen.
- Reift der offizielle ioBroker.mcp-Adapter (npm/beta), Migration via Mapping re-evaluieren.

## Lizenz

MIT — siehe [LICENSE](LICENSE).
