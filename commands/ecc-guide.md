---
description: Navigate ECC's live surface — skills, commands, agents, hooks, rules, MCP connectors, install profiles — and get one canonical path plus a verify command back.
---

# /ecc-guide

Conversational map of Everything Claude Code. Backed by the `ecc-guide` skill,
which owns the full routing, read-budget, and MCP-budget logic. Read
`skills/ecc-guide/SKILL.md` and follow it; this file is the entry point only.

## Usage

```text
/ecc-guide                     # compact menu
/ecc-guide setup | install     # install paths, profiles, scopes
/ecc-guide skills | commands | agents | hooks | rules | mcp
/ecc-guide find: <query>       # search every surface
/ecc-guide <feature-or-file>   # resolve one component
```

## Operating Rules

1. Answer from current files, never memory — no hardcoded counts or feature lists.
2. Stay on the lowest read tier the question needs (T0 conceptual → T3 full catalog).
3. Lead with the answer: surface, canonical path, verify command, one next action.
4. Never invent a component; check the filesystem before claiming it exists.
5. Advisory only — resolve and explain, never install or run the named surface.

## Where To Look

| Surface | Canonical location |
|---|---|
| Skills | `skills/*/SKILL.md` |
| Commands | `commands/*.md` |
| Agents | `agents/*.md` |
| Hooks | `hooks/hooks.json`, `hooks/README.md`, `scripts/hooks/` |
| Rules | `rules/` |
| MCP connectors | `mcp-configs/mcp-servers.json`, `docs/MCP-CONNECTOR-POLICY.md` |
| Install profiles | `manifests/install-*.json`, `README.md` |
| Live catalog | `node scripts/ci/catalog.js --json` |

## Modes

- **No argument** — compact menu: install, pick skills, commands vs skills,
  agents and delegation, hooks and safety, MCP budget, troubleshooting. Then ask
  what they want next.
- **Topic** — 3-6 bullets on the current surface, the canonical directory, one
  verify command. No exhaustive lists unless asked.
- **`find: <query>`** — `rg` across skills, commands, agents, rules, docs; group
  by surface, strongest match first, one next action each.
- **Feature name** — exact-path lookup first (`skills/<n>/SKILL.md`,
  `commands/<n>.md`, `agents/<n>.md`), then `rg`. Explain what it does, when to
  use it, and which file is canonical.

## Related

`/project-init` · `/harness-audit` · `/skill-health` · `/skill-create` ·
`/security-scan` · `ecc-recipes` (pipelines) · `configure-ecc` (install wizard)
