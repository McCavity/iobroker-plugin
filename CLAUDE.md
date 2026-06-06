# iobroker-plugin — Conventions for AI agents

> Last update: 2026-06-06

This file orients AI coding agents (Claude Code, Codex, etc.) when working on
this repository. Humans should read [README.md](README.md) first.

## What this project is

A Claude-Code + Codex **plugin** (skills bundle) wrapping an ioBroker MCP server.
Third iteration of the Unit-3 plugin pattern after
[paperless-bulk-plugin](https://github.com/McCavity/paperless-bulk-plugin) — same
layout (marketplace-root + `plugins/<name>/` with CC *and* Codex manifest), with
one twist: **backend-agnostic**.

There is **no server code here.** The plugin contributes three skills and
references an ioBroker MCP server via `.mcp.json`.

## Key design choices

- **Backend-agnostic skills.** Skills describe *goals*, not fixed tool calls, and
  name tools from BOTH backends as examples. Two supported backends:
  - **McCavity/iobroker-mcp** — FastMCP over the SimpleAPI adapter, stdio,
    path-launched, credentials in a local `.env`. The production backend.
  - **official ioBroker/ioBroker.mcp** (GermanBluefox) — ioBroker adapter,
    Streamable HTTP/SSE. Experimental as of 2026-06 (v0.1.4, no npm).
  - The single source of truth for the mapping is [`docs/tool-mapping.md`](docs/tool-mapping.md).
    When you touch a skill's tool references, update that table too.
- **Skills are atomic.** New workflows = new skill directories, never an
  extension of an existing skill.
- **Descriptions carry the discovery clause.** Every SKILL.md `description` must
  contain a concrete "Use when …" with real German trigger phrasings — that is
  what makes progressive-disclosure discovery fire. Keep < 800 lines per skill.

## Where the skill content comes from

The three skills encode hard-won ioBroker lessons from Henning's KI-OS learning log:

- `find-state-by-pattern` — SimpleAPI whole-string glob (bare → `name.*`,
  `*substring*` returns zero, `script.js.*`/`scenes.0.*` filtered from listings).
  Learned 2026-05-25.
- `battery-check-pattern` — the `available=false` guard (skip the read to avoid
  the server-side `getState` WARN-spam) + `typeof === number` before the
  threshold compare (the `undefined > N === false` trap). Learned 2026-05-27.
- `diagnose-device` — online → battery → history → adapter-health ordering, so a
  stale value isn't read as live and an adapter-down isn't mistaken for a device
  fault.

If you change ioBroker behaviour assumptions, re-ground them against the KI-OS
learning log rather than guessing.

## .mcp.json caveat

The shipped `.mcp.json` is a **template**. Backend A uses path placeholders
(`${IOBROKER_MCP_PYTHON}` / `${IOBROKER_MCP_SERVER}`) because iobroker-mcp is not
a pip module (unlike paperless-bulk-mcp). If `iobroker` is already registered
user-scope (Henning's machine: `~/.claude.json`), the `mcpServers` block must be
removed to avoid a double registration — the plugin then only contributes skills.

## Conventions

- Solo-maintainer repo: branch protection with `required_approving_review_count: 0`
  and `enforce_admins: false`. PR-based workflow for changes; the initial scaffold
  is a single commit on `main`.
- Semantic Versioning for the plugin + marketplace `version` fields (keep them in sync).
- Markdown/JSON only — no build step, no dependencies.
