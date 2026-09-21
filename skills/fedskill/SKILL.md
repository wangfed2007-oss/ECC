---
name: fedskill
description: "The all-in-one universal reference — an ultra-complete dictionary and playbook the AI can use on ANY project: architecture, APIs, data, testing, git, CI/CD, security, performance, debugging, refactoring, docs, incidents, every major stack, every common workflow, plus modern AI/agent engineering (context engineering, MCP budgets, orchestration, evals, agent security). Routes any question to the right volume and loads only what the question needs. TRIGGER when the user asks for a definition, a best practice, a decision table, a workflow with run-order and stop condition, a checklist, a stack idiom, or general engineering/AI guidance not covered by a more specific project skill. DO NOT TRIGGER when a dedicated project skill matches the task (most specific wins), when the user wants ECC repository navigation (use ecc-guide) or ECC command pipelines (use ecc-recipes), or when the user wants the work executed rather than explained — then execute with the relevant workflow volume as silent background."
argument-hint: "<term | topic | workflow | checklist | stack | empty=index>"
metadata:
  origin: community
  author: wangfed2007
  version: "1.0.0"
  role: universal-reference
---

# FedSkill — The Universal Dictionary

One skill, every project. FedSkill is a complete, project-agnostic reference
library — a dictionary for the AI — split into six deep volumes under
`references/`, with this file as the router. The router always loads cheap;
volumes load only when a question needs them.

**Contract:** FedSkill explains, defines, routes, and hands over playbooks. It
adapts to the project at hand; it never overrides a repository's own rules,
skills, or conventions — the most specific surface always wins.

## The Library

| Volume | File | Covers | Load when the user asks about |
|---|---|---|---|
| Glossary | `references/glossary.md` | 250+ A–Z definitions across software, web, data, infra, security, testing, AI/ML, agentic AI, process | "what is / what does X mean", term disambiguation, vocabulary |
| AI & Agent Engineering | `references/ai-agent-engineering.md` | Context engineering, surface classes (skill/command/agent/hook/rule/MCP), prompt design, MCP budget, orchestration patterns, memory, evals, agent security, cost, failure modes | building or debugging AI agents, prompts, context, MCP, evals, agent safety |
| Software Engineering | `references/software-engineering.md` | Architecture decisions, API design, data modeling, testing strategy, git, CI/CD, security baseline, performance, debugging, refactoring, docs, incidents | "how should I design/test/secure/debug/document X" |
| Workflows | `references/workflows.md` | 15 run-order pipelines (feature, bugfix, TDD, refactor, migration, release, incident, perf hunt, security audit, onboarding…) each with steps, artifacts, STOP condition | "what's the process for X", multi-step tasks needing an ordered plan |
| Stack Playbooks | `references/stack-playbooks.md` | Per-stack detection files, build/test/lint commands, idioms, traps: JS/TS, React, Python, Go, Rust, JVM, .NET, mobile, SQL, shell, Docker/K8s, CI | language- or stack-specific idioms, commands, pitfalls, stack detection |
| Checklists | `references/checklists.md` | 14 verifiable checklists: PR review, security, release, migration, a11y, perf, agent-config review… | "review this", "am I ready to ship", gate-style verification |

## Prime Directives

1. **Route, then read.** Pick the volume from the table above before opening
   anything. One question rarely needs more than one volume.
2. **Grep before you load.** Inside a volume, jump to the matching section or
   term (`rg -n "<term>" skills/fedskill/references/<volume>.md`) instead of
   reading the whole file. Volumes are 400–700 lines each; the full library is
   a budget, not a single read.
3. **Project rules outrank the dictionary.** If the repository defines its own
   convention (CLAUDE.md, rules/, a project skill), apply it and use FedSkill
   only to fill gaps. Never cite this library to overrule a local rule.
4. **Answer from the volume, verify in the project.** A definition comes from
   the glossary; a claim about *this* codebase comes from its files.
5. **Executed work stays silent.** When the user wants the task done, do it —
   the relevant workflow or checklist runs as your internal discipline, not as
   a lecture.

## Intent Router

| User intent sounds like | Go to | Then |
|---|---|---|
| "what is X / define X / X vs Y" | Glossary → letter section | quote the entry, add the "vs"/Trap line if present |
| "how do agents/MCP/context/evals work" | AI & Agent Engineering → matching numbered section | summarize, link deeper section |
| "how should I architect/design/test/secure X" | Software Engineering → matching section | give the decision table or method, then one next action |
| "what's the process / step-by-step for X" | Workflows → `## Workflow: <name>` | give run-order + STOP condition verbatim |
| "how do I do X in <language/stack>" | Stack Playbooks → `## <Stack>` | give the command/idiom + relevant Trap |
| "review / verify / am I ready" | Checklists → `## Checklist: <name>` | render the checklist, tick what you can verify from the repo |
| "which part of ECC do I use" (inside ECC repo) | `ecc-guide` skill | hand over |
| "which ECC commands in what order" | `ecc-recipes` skill | hand over |
| unclear | ask one clarifying question OR show the six-volume index | never dump multiple volumes |

**Tie-break:** definition → Glossary; decision → Software Engineering or AI &
Agent Engineering; sequence → Workflows; gate → Checklists; syntax → Stack
Playbooks.

## Read Budget

| Tier | Question | Reads | Cost |
|---|---|---|---|
| T0 | Concept you already hold confidently and the volume confirms | none | ~0 |
| T1 | One term or one command | one `rg` into one volume | <1k tokens |
| T2 | One method, workflow, or checklist | one section of one volume | <3k tokens |
| T3 | Cross-cutting design question | sections from 2 volumes max | <8k tokens |

Never load all six volumes for one question. If two volumes disagree, prefer
the more specific one (Stack Playbooks > Software Engineering for stack
questions; AI & Agent Engineering > Glossary for agent depth) and say so.

## Answer Shapes

Definition:

```text
<term> — <definition from glossary>.
vs <neighbor>: <one-line disambiguation>   (if present)
Trap: <the known trap>                     (if present)
In this project: <how it applies here, from repo files>
```

Method / decision:

```text
Recommendation: <the choice>.
Why: <top 1-2 reasons from the volume>
Decision table: <the relevant rows only>
Next: <one concrete action in this repo>
```

Workflow:

```text
Workflow: <name>
Run-order: <numbered steps>
STOP when: <condition>
First step here: <command or file for this repo>
```

Checklist: render the task list, mark items verifiable from the repo as
checked/unchecked with a one-line evidence note each.

## Keeping the Dictionary Alive

- A recurring question with no matching entry → add the entry to the right
  volume (alphabetical for glossary, thematic elsewhere) in the same style.
- A wrong or stale entry → fix it in place; the library claims 2026–2027
  practice and must stay current.
- A new stack or workflow the team adopts → new `## <Stack>` or
  `## Workflow: <name>` section following the volume's existing shape.
- Keep the router (this file) stable; volumes absorb the growth.

## Anti-Patterns

- Loading several volumes for a single-term question
- Reciting a volume instead of answering, or answering instead of executing
- Overruling a project's own rules with the dictionary
- Quoting counts, versions, or APIs from memory when the volume or repo can be read
- Adding entries that duplicate an existing term under a synonym — extend the
  existing entry instead

## Related Surfaces

- `ecc-guide` — navigation of the ECC repository's own catalog
- `ecc-recipes` — ECC command-group pipelines with run-order and stop condition
- `context-budget` — token accounting when sessions feel heavy
- Project-local skills — always more specific, always win
