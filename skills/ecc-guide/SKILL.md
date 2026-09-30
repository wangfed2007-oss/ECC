---
name: ecc-guide
description: "Route any \"which part of ECC do I use for X?\" question to the exact surface — skill, command, agent, hook, rule, MCP connector, install profile — by reading the live repository, never memory. Answers in one screen with a canonical path and a verify command. TRIGGER when the user asks what ECC includes, where a component lives, which surface fits a task, how to install/reset/migrate/uninstall ECC, why a hook or connector behaves as it does, or how commands, skills, agents, hooks and rules relate. DO NOT TRIGGER when the user wants the work executed (invoke the component instead), wants a multi-command pipeline with run-order and stop condition (use ecc-recipes), or wants the interactive install wizard (use configure-ecc)."
argument-hint: "<topic | find: query | component name | empty=menu>"
metadata:
  origin: community
  version: "2.0.0"
  surface-baseline: "2027"
---

# ECC Guide

The navigation layer for Everything Claude Code. Turns a vague intent into one
named surface, its canonical file, and a command that proves the answer.

**Contract:** advisory and read-only. This skill resolves and explains surfaces.
It never installs, never mutates config, and never runs the surface it names.

## When To Use

- "What does ECC include?" / "How do I do X with ECC?"
- Finding a skill, command, agent, hook, rule, MCP connector, or install profile
- Choosing between two surfaces that look like they overlap
- Understanding install paths, scopes, duplicate installs, reset, uninstall
- Explaining how commands, skills, agents, hooks, rules and MCP relate
- Diagnosing "ECC is installed but X is not showing up"

### Do Not Use When

| Situation | Route to |
|---|---|
| User wants the task executed now | The actual skill/command |
| User wants a multi-command pipeline with run-order + stop condition | `ecc-recipes` |
| User wants to install, reconfigure, or migrate scope interactively | `configure-ecc` |
| User wants a draft prompt rewritten | `prompt-optimizer` |
| User wants token/cost accounting of ECC surfaces | `context-budget`, `ecc-tools-cost-audit` |

## Prime Directive

**Answer from current files, never from memory.** ECC's catalog moves weekly.
Any hardcoded count, feature list, or install flag is a future wrong answer.

Three rules that follow from it:

1. Never state a count you did not just read.
2. Never claim a component exists without hitting the filesystem.
3. If the checkout is unavailable, say so and answer structurally ("skills live
   in `skills/<name>/SKILL.md`") instead of guessing names.

## Read Budget Ladder

Escalate only as far as the question requires. Most questions stop at T1.

| Tier | Question shape | Reads | Target cost |
|---|---|---|---|
| **T0** | Conceptual — "skill vs command?", "what is a hook profile?" | none | ~0 tokens |
| **T1** | Existence / location — "is there a Rust reviewer?" | one `find` or `rg` | < 1k tokens |
| **T2** | Choice — "which of these fits my repo?" | frontmatter of 2-4 candidates | < 4k tokens |
| **T3** | Survey — "show me every X" (only when explicitly asked) | `catalog.js --json` | 10k+ tokens |

Cheap probes, in escalation order:

```bash
# T1 — existence and location (fastest, no deps)
ls skills/<name>/SKILL.md commands/<name>.md agents/<name>.md 2>/dev/null
rg -l "<query>" skills commands agents rules docs --max-count 1

# T2 — decide between candidates without reading whole files
head -6 skills/<name>/SKILL.md            # frontmatter only
rg -n "^description:" skills/*/SKILL.md | rg -i "<query>"

# T3 — full live catalog (only on explicit "list everything")
node scripts/ci/catalog.js --json
node scripts/install-plan.js --list-profiles
node scripts/install-plan.js --list-components --json
```

Efficiency rules: read frontmatter before bodies, cache what you read for the
rest of the session, batch independent probes into one call, and never re-run
`catalog.js` twice in a session.

## Intent Router

Map the user's words to a surface before reading anything.

| User says | Surface class | Resolve at | Verify with |
|---|---|---|---|
| "workflow / playbook / how do I do X" | skill | `skills/<name>/SKILL.md` | `ls skills/<name>/` |
| "slash command / `/x`" | command | `commands/<name>.md` | `ls commands/` |
| "delegate / subagent / parallel" | agent | `agents/<name>.md` | `ls agents/` |
| "automatic / on every edit / block me from" | hook | `hooks/hooks.json`, `scripts/hooks/` | `cat hooks/README.md` |
| "always follow / standard / policy" | rule | `rules/` | `ls rules/` |
| "connect to <external system>" | MCP or skill | `mcp-configs/mcp-servers.json`, `docs/MCP-CONNECTOR-POLICY.md` | see MCP Budget |
| "install / profile / scope / uninstall" | installer | `manifests/install-*.json`, `README.md` | `node scripts/install-plan.js --list-profiles` |
| "set ECC up for this repo" | onboarding | `/project-init` | dry-run plan |
| "is my setup healthy / safe" | audit | `/harness-audit`, `/security-scan` | `npm run harness:audit -- --format text` |

**Precedence when two surfaces match:** skill > agent > command > hook.
Skills are the primary workflow surface; commands are maintained compatibility
shims; agents matter when the work should run in a separate context window;
hooks matter only when the behavior must fire without being asked.

## Surface Model

One sentence each — use these when the user is confused about the layers:

- **Skill** — a workflow the model loads *when relevant*. Progressive disclosure:
  costs context only on activation.
- **Command** — an explicit user-typed entry point. Deterministic invocation,
  same content class as a skill.
- **Agent** — delegated work in a *separate* context window. Use for wide search,
  independent review, and anything that would flood the main thread.
- **Hook** — deterministic automation on a lifecycle event. Runs whether or not
  the model cooperates; this is the enforcement layer.
- **Rule** — always-loaded guidance. Highest context tax per line, so the
  shortest surface wins here.
- **MCP connector** — a live external system with session state. Tool schemas
  load into *every* session, used or not.

## MCP Connector Budget (2027 posture)

Canonical policy: `docs/MCP-CONNECTOR-POLICY.md`. The short version:

A connector earns a default slot only if it is **universal** *and* the job
genuinely needs what MCP provides — held-open session state, streaming, an auth
handshake, or structured browsing. Stateless request/response work is a skill
wrapping a CLI or REST API, not a server.

The 2027 field default across serious harnesses remains **zero to two default
connectors plus native built-ins**. ECC ships one (`chrome-devtools`); six were
retired to skills in the June 2026 audit and remain opt-in in
`mcp-configs/mcp-servers.json`. That audit's verdict still holds in 2027 —
harness-native search, memory, and thinking have only absorbed more of what
those servers used to justify.

When a user asks for a new connector, ask in this order:

1. Does a CLI or REST API already do this? → skill, not server.
2. Is the value the *held session*, or a one-shot call? → one-shot means skill.
3. Does it need a key? → fails universality; opt-in only, never a default.
4. What is the schema tax on every session that never calls it?

Opt out of shipped connectors with `ECC_DISABLED_MCPS="chrome-devtools"`.

## Install Guidance

Always plan, then dry-run, then apply. Never hand-copy component files when the
managed installer supports the target.

```bash
node scripts/install-plan.js --list-profiles
node scripts/install-plan.js --profile minimal --target claude --json
node scripts/install-apply.js --profile minimal --target claude --dry-run

# single skill instead of a profile
node scripts/install-plan.js --skills <skill-id> --target claude --json
```

Flags worth knowing: `--modules`, `--with`, `--without`, `--family`, `--config`,
`--target`. Targets cover Claude Code, Codex, Cursor, OpenCode, Kimi, Gemini,
CodeBuddy, JoyCode, Qwen — check `--list-components --json` for live support
rather than asserting a matrix from memory.

**Duplicate-surface warning:** stacking a plugin install *and* a full manual or
profile install produces two copies of every component. Confirm intent first.

## Diagnostics

Triage in this order; stop at the first layer that explains the symptom.

| Symptom | First check |
|---|---|
| Component missing after install | install scope and target dir — `.claude/`, `.codex/`, `.cursor/`, `.opencode/`, `.gemini/`, `.kimi-code/`, `.codebuddy/`, `.joycode/`, `.qwen/` |
| Component appears twice | plugin install stacked on manual/profile install |
| Hook not firing | `hooks/hooks.json` matcher, then `ECC_HOOK_PROFILE` / `ECC_DISABLED_HOOKS` |
| Hook firing too much | hook profile is `strict`; drop to `standard` or `minimal` |
| `Cannot find module` from a script | dependencies not installed — `npm ci` |
| Connector tools absent | `ECC_DISABLED_MCPS`, then the harness's own MCP config |
| Sessions feel slow / context thin | `context-budget`, then trim rules and connectors first |

Repo health, in ascending cost:

```bash
npm run harness:audit -- --format text
npm run observability:ready
npm test
```

## Answer Shapes

Lead with the answer. One screen, then offer depth.

```text
Use <surface>. It fits because <one reason>.
Canonical: <path>
Verify:    <command>
Next:      <one concrete action>
```

For a search:

```text
Best matches:
- <path> — <why it matters>
- <path> — <why it matters>
Start with: <one> because <reason>.
```

For an install:

```text
Detected: <stack evidence>
Target:   <harness>  Scope: <user|project|local>
Plan:     <profile/modules/skills>
Dry run:  <command>
Would change: <paths>
Approval needed before apply: <yes/no>
```

## Anti-Patterns

- Dumping the full catalog when one path was asked for
- Quoting counts, versions, or profile names from memory
- Recommending a retired command shim when a skill-first path exists
- Manual `cp` instructions when the installer supports the target
- Escalating to T3 for a T1 question
- Explaining all six surface classes when the user asked about one
- Recommending a new MCP connector without applying the four budget questions

## Related Surfaces

| Need | Surface |
|---|---|
| Pipeline of commands with run-order + stop condition | `ecc-recipes` |
| Interactive install / reconfigure / scope migration | `configure-ecc` |
| Stack-aware onboarding for a target repo | `/project-init` |
| Deterministic readiness scorecard | `/harness-audit` |
| Skill quality review | `/skill-health` |
| Generate a skill from local git history | `/skill-create` |
| Config security review | `/security-scan` |
| Token/cost accounting | `context-budget`, `ecc-tools-cost-audit` |
| Universal engineering dictionary, workflows, checklists for any project | `fedskill` |
