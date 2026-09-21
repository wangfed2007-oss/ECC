---
description: Query the FedSkill universal dictionary — definitions, methods, workflows, checklists, and stack playbooks for any project, routed to the right reference volume.
---

# /fedskill

Entry point for the `fedskill` skill — the all-in-one universal reference
library. Routing, read-budget, and answer-shape logic live in
`skills/fedskill/SKILL.md`; read it and follow it. This file only defines the
entry point.

## Usage

```text
/fedskill                        # show the six-volume index
/fedskill <term>                 # dictionary lookup (glossary)
/fedskill workflow <name|task>   # run-order pipeline with STOP condition
/fedskill checklist <name>       # verifiable gate checklist
/fedskill stack <language|tool>  # stack playbook: commands, idioms, traps
/fedskill agents <topic>         # AI/agent engineering deep reference
/fedskill design <question>      # software engineering method or decision table
```

## Operating Rules

1. Route to ONE volume from the skill's intent router before reading anything.
2. Grep to the matching section; never load the whole library for one question.
3. Project rules and project skills outrank the dictionary — most specific wins.
4. Lead with the answer in the skill's answer shape, then one next action for this repo.
5. Advisory when asked to explain; silent background discipline when asked to execute.

## Volumes

| Volume | File |
|---|---|
| Glossary (A–Z) | `skills/fedskill/references/glossary.md` |
| AI & Agent Engineering | `skills/fedskill/references/ai-agent-engineering.md` |
| Software Engineering | `skills/fedskill/references/software-engineering.md` |
| Workflows | `skills/fedskill/references/workflows.md` |
| Stack Playbooks | `skills/fedskill/references/stack-playbooks.md` |
| Checklists | `skills/fedskill/references/checklists.md` |

## Related

`ecc-guide` (ECC catalog navigation) · `ecc-recipes` (ECC command pipelines) ·
`context-budget` (token accounting)
