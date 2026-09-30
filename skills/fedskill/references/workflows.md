# Universal Workflows

Fifteen run-order pipelines that apply to any project, each with an explicit trigger, numbered steps, the artifacts it must produce, a STOP condition, and failure exits. Load this volume when a task needs an ordered plan — "what's the process for X" — or when executing a multi-step change and you want the discipline running silently in the background. Steps name intents, not tools: substitute the project's own commands (see `stack-playbooks.md`).

## Workflow: Feature development

**Trigger:** a new capability is requested with at least a one-sentence spec.
**Preconditions:** the repo builds and its test suite passes on the base branch; requirements have one named owner who can accept or reject the result.

1. Restate the requirement in one paragraph: user-visible behavior, out-of-scope items, acceptance criteria. Get the restatement confirmed if an owner is available.
2. Locate the seams: entry points, modules, and data structures the feature touches. Read them before designing.
3. Write the plan: ordered slices, each independently shippable and testable. Prefer the thinnest vertical slice first.
4. For each slice: write or extend tests for the new behavior, implement until green, refactor while green.
5. Run the project's full fast-check set (lint, typecheck, unit) locally.
6. Self-review the diff adversarially: naming, dead code, error paths, missing tests, accidental scope.
7. Open a small PR per slice with the restated requirement in the description; link the acceptance criteria.

**Artifacts:** confirmed restatement, plan with slices, tests per slice, PR(s).
**STOP when:** all acceptance criteria demonstrably pass and CI is green on the final slice.
**Failure exits:** requirement cannot be restated crisply → return to the owner with the specific ambiguity; a slice balloons past its estimate → stop, re-plan the remaining slices before writing more code.

## Workflow: Bug fix

**Trigger:** a defect report — behavior differs from expectation.
**Preconditions:** the expected behavior is stated or recoverable from tests/docs.

1. Reproduce the failure deterministically. No reproduction, no fix — instrument or narrow until it reproduces.
2. Encode the reproduction as a failing regression test at the lowest level that exhibits the bug.
3. Diagnose to the root cause, not the first plausible cause: explain *why* the code produces the wrong result before touching it.
4. Fix with the smallest change that makes the regression test pass without breaking existing tests.
5. Sweep for siblings: search for the same pattern elsewhere; fix or file each occurrence.
6. Run the full fast-check set; ship with the regression test in the same commit as the fix.

**Artifacts:** reproduction steps, failing-then-passing regression test, root-cause note in the commit message.
**STOP when:** regression test passes, prior suite passes, and the sibling sweep is done.
**Failure exits:** cannot reproduce → downgrade to instrumented monitoring with an alert, do not "fix" blind; root cause is in a dependency → pin or patch upstream, record the tracking link.

## Workflow: TDD loop

**Trigger:** implementing well-specified behavior where tests can lead.
**Preconditions:** a test runner wired up; behavior decomposable into cases.

1. RED — write one test for the next smallest behavior. Run it; it must fail for the expected reason (assertion, not import error).
2. GREEN — write the minimum code that passes. Resist generalizing beyond the test.
3. REFACTOR — with all tests green, remove duplication and improve names. No new behavior in this step.
4. Commit at green. Repeat from RED for the next case.
5. Every third or fourth loop, step back: is the test list still the right decomposition? Prune or add cases.

**Artifacts:** a commit history alternating test+impl pairs; a test file that reads as the spec.
**STOP when:** the behavior list is exhausted and edge cases (empty, null, boundary, error) each have a test.
**Failure exits:** a test is hard to write → the design is telling you the seam is wrong; refactor the seam first, then resume.

## Workflow: Refactor without behavior change

**Trigger:** structure is impeding work; behavior must stay identical.
**Preconditions:** the code to refactor is exercised by tests — if not, step 1 creates them.

1. Pin behavior with characterization tests: capture current outputs for representative and edge inputs, even if current behavior looks wrong. Wrong-but-current is the contract for this workflow.
2. Declare the target shape in two or three sentences (what moves where, what dies).
3. Move in mechanical steps — rename, extract, inline, move — running tests after each step. Never mix a mechanical step with a judgment step in one commit.
4. If a genuine bug surfaces, stop: record it, finish the refactor preserving the buggy behavior, fix the bug afterward in its own commit via the Bug fix workflow.
5. Diff review: confirm no observable-behavior lines changed (I/O, persisted formats, public APIs, error messages consumers parse).

**Artifacts:** characterization tests, a series of small mechanical commits, separate bug tickets for anything found.
**STOP when:** target shape reached, all tests green, diff contains no behavior edits.
**Failure exits:** characterization is infeasible (nondeterminism, hidden I/O) → first extract the nondeterminism behind a seam, then restart.

## Workflow: Codebase onboarding

**Trigger:** you (human or agent) are new to a repository and must act in it.
**Preconditions:** read access to the repo.

1. Read the orientation files in order: README, CONTRIBUTING, top-level config/manifests, CI pipeline definition, and any agent-guidance file (`CLAUDE.md`, `AGENTS.md`).
2. Map the skeleton: top-level directories, entry points, how the app starts, where tests live. One screen of notes, not an essay.
3. Run the project: install, build, test. A failing baseline is finding #1 — record it before changing anything.
4. Trace one representative request/operation end to end through the layers.
5. Extract conventions: naming, error handling, test style, commit style — from recent merged PRs, not just docs.
6. Make one small, safe, reviewed change (doc fix, test gap) to validate the full contribute-review-merge loop.

**Artifacts:** skeleton map, baseline status (build/test results), conventions note, one merged trivial change.
**STOP when:** you can predict where a named feature lives before searching for it.
**Failure exits:** the project doesn't build → fix or document the setup gap first; that is the highest-value onboarding contribution.

## Workflow: Dependency upgrade

**Trigger:** security advisory, needed feature, or scheduled hygiene.
**Preconditions:** lockfile present; CI green on base.

1. Inventory: current version, target version, changelog/breaking-changes list between them, transitive impact.
2. Classify: patch (low risk), minor (review changelog), major (read the migration guide fully before deciding).
3. Upgrade in rings: dev-dependencies and tooling first, then libraries, then frameworks/runtimes — separate PRs per ring.
4. Regenerate the lockfile with the project's own tool; never hand-edit generated files.
5. Run the full suite plus a manual smoke of the features the dependency serves.
6. For majors: grep for every deprecated API named in the migration guide; fix all call sites in the same PR.

**Artifacts:** per-ring PRs with changelog links, updated lockfile, green CI.
**STOP when:** all target versions in, suite green, no deprecation warnings introduced.
**Failure exits:** a major upgrade fans out beyond ~a day of work → split: land a compatibility shim or pin, schedule the migration as its own project.

## Workflow: Database migration

**Trigger:** a schema change on a live datastore.
**Preconditions:** migrations are versioned files run by a tool, never ad-hoc SQL in production; a backup/restore path exists and has been tested.

1. Design expand-first: the new schema element is added alongside the old (new column, new table), never replacing it in the same release.
2. EXPAND — ship the additive migration. Old code keeps working; verify it is backward-compatible by running the previous app version against the migrated schema in staging.
3. BACKFILL — copy/derive data into the new element in batches sized to avoid lock pressure; make the backfill idempotent and resumable.
4. SWITCH — deploy code that writes to both and reads from the new element; watch error rates and data-drift checks.
5. CONTRACT — only after a full release cycle of clean reads: stop dual-writes, then drop the old element in its own migration.
6. At every step, record the rollback: which migration reverses it and whether data loss is possible.

**Artifacts:** ordered migration files, backfill job with progress metric, rollback notes per step.
**STOP when:** contract migration applied and a full cycle passes with no reference to the old element.
**Failure exits:** backfill drifts from live writes → pause at step 3, add dual-write before continuing; long-running lock detected → abort, redesign with batched or online-DDL approach.

## Workflow: Release

**Trigger:** a planned version ships.
**Preconditions:** versioning scheme and changelog convention exist; deploy and rollback are scripted.

1. Freeze scope: enumerate what is in; everything else waits. Cut a release branch only if the repo's convention uses one.
2. Verify: full test suite, packaging/build of the actual artifact, smoke test of the built artifact (not the dev tree).
3. Write the changelog from merged changes: user-facing language, breaking changes flagged first.
4. Tag with the version; build once, promote that same artifact through environments — never rebuild per environment.
5. Deploy with the project's strategy (rolling/canary/blue-green); watch error rate, latency, and one business metric for the bake period.
6. Announce: changelog to users/team; close the release ticket with links to tag, artifact, and dashboard.

**Artifacts:** tag, immutable artifact, changelog, deploy record with metrics snapshot.
**STOP when:** bake period passes within thresholds and the announcement is out.
**Failure exits:** verification fails → unfreeze is not allowed; fix-forward on the release scope only, or abandon the release. Bake fails → roll back first, diagnose second.

## Workflow: Incident response

**Trigger:** production impact — outage, data issue, security event.
**Preconditions:** severity levels and an escalation contact are defined somewhere findable.

1. Declare: state severity, appoint one incident lead, open one comms channel. The lead coordinates and communicates; others execute.
2. MITIGATE before diagnosing: roll back the last change, fail over, disable the feature flag, scale up — the fastest action that stops user impact wins.
3. Communicate on a fixed cadence (e.g., every 30 min for high severity) even when the update is "no change".
4. Diagnose with the timeline: what changed nearest to onset — deploys, config, data, traffic, dependencies.
5. Fix and verify in production with the same metrics that detected the incident.
6. Within a few days: blameless postmortem — timeline, impact quantified, root cause(s), and actions each with an owner and date. Actions without owners are decorations.

**Artifacts:** incident channel log, timeline, postmortem with owned actions.
**STOP when:** impact ended, postmortem published, actions filed.
**Failure exits:** mitigation unknown after ~15 minutes of high-severity impact → escalate up the contact chain immediately; that is what it is for.

## Workflow: Performance hunt

**Trigger:** a measured performance problem or a target to hit.
**Preconditions:** the metric is defined (which percentile, which operation, which load).

1. Baseline: measure the current number under reproducible conditions; record the harness so the measurement can be repeated.
2. Profile before hypothesizing: CPU/alloc/IO/query profile of the real workload. The bottleneck is where the profile says, not where intuition says.
3. Form one hypothesis; predict the improvement size before changing anything.
4. Change one thing; re-measure with the same harness; keep only wins that reproduce and don't regress correctness or another metric.
5. Repeat from step 2 — after each win the bottleneck moves.
6. Stop at the target, not at perfection; record the final harness + numbers next to the target for the next hunt.

**Artifacts:** measurement harness, before/after numbers per change, profile snapshots.
**STOP when:** target met with margin, or remaining wins cost more than they return — say which.
**Failure exits:** cannot reproduce the slowness → instrument production with percentile timers first; optimizing unmeasured code is refactoring cosplay.

## Workflow: Security audit

**Trigger:** scheduled review, pre-release gate, or post-incident hardening.
**Preconditions:** scope agreed — which repos/services/configs are in.

1. Scope and threat-model lite: assets worth stealing, entry points, trust boundaries — one page.
2. Scan mechanically: dependency audit, secret scan across history, static analysis, config linting (permissions, CORS, headers, agent/tool configs).
3. Review by hand along trust boundaries: input validation, authn/authz checks, injection surfaces (SQL, shell, template, prompt), unsafe deserialization.
4. Triage findings by exploitability × impact; false-positive out with a recorded reason, never silently.
5. Fix highs immediately; file mediums/lows with owners; re-scan to prove fixes and catch regressions from the fixes.
6. Report: scope, method, findings, fixed vs accepted-risk (accepted-risk items need a name and a date).

**Artifacts:** threat sketch, scan outputs, triaged finding list, re-scan proof, report.
**STOP when:** highs fixed and verified, everything else owned and dated.
**Failure exits:** a finding suggests active compromise → switch to Incident response immediately; the audit resumes after.

## Workflow: Legacy strangler migration

**Trigger:** replacing a legacy system incrementally while it keeps serving.
**Preconditions:** the legacy system's inbound edges are enumerable (routes, queues, jobs).

1. Find the seam: the narrowest interface where traffic can be intercepted (router, facade, queue consumer).
2. Install the proxy/facade in front of the legacy system with 100% pass-through; ship this alone and let it bake.
3. Pick the first slice by risk-adjusted value: meaningful but not the scariest. Port it to the new system.
4. Route the slice: shadow first (new system runs, results compared, legacy still answers), then cut over with a per-slice flag.
5. Repeat per slice; keep a living inventory of slices with states (legacy / shadow / cut over / retired).
6. Retire legacy pieces as their last slice cuts over — deleting is the point; a strangler that never deletes is just two systems.

**Artifacts:** seam/proxy, slice inventory, comparison reports from shadow phases, deletion commits.
**STOP when:** legacy receives zero traffic for a full business cycle and is decommissioned.
**Failure exits:** shadow comparison shows divergence you can't explain → stop cutting over; the divergence is either a legacy bug (document it) or a port bug (fix it) — decide which before proceeding.

## Workflow: Prototype-to-production hardening

**Trigger:** a prototype/spike is promoted to a real feature.
**Preconditions:** an explicit decision that this code lives — otherwise delete it; hardening dead code is waste.

1. Cut scope to what production actually needs; delete demo paths, dead flags, and speculative options first.
2. Add the test layer the prototype skipped: characterization tests over current behavior, then unit tests on the risky logic.
3. Error paths: every external call gets timeout, retry-or-fail decision, and a user-visible failure mode.
4. Observability: logs with correlation ids at boundaries, metrics on the operations that matter, alerts on the failure modes from step 3.
5. Security pass: secrets to the secret store, input validation at trust boundaries, dependency audit.
6. Docs: README section or runbook — how to run, configure, and diagnose it.

**Artifacts:** reduced-scope diff, test suite, runbook.
**STOP when:** the Production readiness checklist (see `checklists.md`) passes.
**Failure exits:** hardening cost approaches rewrite cost → rewrite using the prototype as the spec, not the base.

## Workflow: AI-agent task loop

**Trigger:** an AI agent executes any nontrivial task autonomously.
**Preconditions:** task stated; tool access scoped to what the task needs.

1. Understand: restate the goal and the done-condition; read the relevant files before forming a plan; surface blocking ambiguity now, not mid-run.
2. Plan: smallest sequence of verifiable steps; identify the irreversible ones — those get checkpoints or human approval.
3. Act in small steps: after each change, verify it (run the test, re-read the diff, execute the command) before building on it.
4. Self-review at the end as an adversary: what would a reviewer or CI reject? Fix findings before reporting.
5. Report honestly: what was done, what was verified vs assumed, what remains. Never report unverified success — "tests pass" only after running them.

**Artifacts:** plan, verified diffs/commits, honest final report.
**STOP when:** the done-condition demonstrably holds, or a blocker genuinely requires the human — named precisely.
**Failure exits:** two consecutive fix attempts fail the same verification → stop looping; re-diagnose from scratch or escalate with the evidence gathered.

## Workflow: Documentation sprint

**Trigger:** docs materially lag the system.
**Preconditions:** a defined audience (new dev? operator? API consumer?) — docs without an audience converge on noise.

1. Inventory what exists; mark each page current / stale / wrong. Wrong beats absent as a priority — wrong docs cost more than none.
2. List the top tasks the audience actually performs; gaps against that list are the backlog, ranked by frequency × pain.
3. Write task-first: each page answers one "how do I…" with prerequisites, steps, and verification. Reference material (API tables, schemas) is generated from source where possible, not hand-copied.
4. Verify every command and code block by executing it against the current system.
5. Publish and wire freshness: docs live next to code, reviewed in the same PRs that change behavior; add a docs line-item to the PR checklist.

**Artifacts:** inventory with verdicts, ranked backlog, verified pages, PR-checklist hook.
**STOP when:** the top audience tasks each have a current, verified page.
**Failure exits:** a page cannot be verified because the feature misbehaves → that's a bug report, not a doc; file it and mark the page blocked.

## Choosing a workflow

| The intent sounds like | Workflow | First step |
|---|---|---|
| "add / build / implement <capability>" | Feature development | restate the requirement + acceptance criteria |
| "X is broken / behaves wrong" | Bug fix | reproduce deterministically |
| "implement this spec test-first" | TDD loop | write the first failing test |
| "clean this up without changing behavior" | Refactor without behavior change | characterization tests |
| "get to know this codebase" | Codebase onboarding | read README/CONTRIBUTING/CI/agent files |
| "bump / upgrade <dependency>" | Dependency upgrade | changelog + breaking-changes inventory |
| "change the schema" | Database migration | design the expand step |
| "ship version X" | Release | freeze scope |
| "production is down / data is wrong" | Incident response | declare severity + lead, then mitigate |
| "it's slow / hit this latency target" | Performance hunt | baseline measurement |
| "audit / harden security" | Security audit | scope + threat sketch |
| "replace the legacy system gradually" | Legacy strangler migration | find the seam |
| "make the prototype real" | Prototype-to-production hardening | cut scope |
| "agent, do this task" | AI-agent task loop | restate goal + done-condition |
| "the docs are out of date" | Documentation sprint | inventory with current/stale/wrong verdicts |

## How to use this volume

- Match intent via the "Choosing a workflow" table, then execute that one workflow's run-order top to bottom; don't blend workflows mid-run — switch explicitly (e.g., Refactor → Bug fix) when a failure exit says so.
- Quote STOP conditions verbatim when planning: they are the done-definition the final report must demonstrate.
- Failure exits are part of the workflow, not admissions of defeat — taking one early is cheaper than pushing a broken run-order.
- Steps name intents; bind them to real commands from the project or from `stack-playbooks.md` before executing.
