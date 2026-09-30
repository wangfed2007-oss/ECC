# Universal Checklists

Fourteen gate-style checklists, each item a verifiable yes/no. Load this volume when the intent is "review this", "are we ready", or any pre-merge/pre-ship gate. Render the relevant checklist as a task list; when a repository is at hand, check items you can verify yourself and attach one line of evidence per checked item. A checklist gates a decision — it does not replace judgment (see the closing section).

## Checklist: Pull request review

- [ ] The PR does one thing; its title and description say what and why
- [ ] The diff matches the stated intent — no unrelated drive-by changes
- [ ] New behavior has tests; changed behavior has updated tests that would fail on the old code
- [ ] Error paths are handled: failures surface with actionable messages, nothing swallows exceptions silently
- [ ] Names read correctly at the call site; no misleading or stale identifiers left behind
- [ ] No dead code, commented-out blocks, or debug output remains
- [ ] Concurrency and resource lifetimes are sound (no leaked handles, unawaited promises, unjoined goroutines)
- [ ] Security scan of the diff: inputs validated at trust boundaries, no injection surface, no secrets, no widened permissions
- [ ] Performance-sensitive paths avoid new N+1s, unbounded loops, or loads of unbounded data
- [ ] Public API / schema changes are backward-compatible or the break is called out and versioned
- [ ] Docs, changelog, and configuration examples are updated where behavior changed
- [ ] CI is green on the current head; flaky reruns are not hiding a real failure

## Checklist: Security review of a change

- [ ] Every new external input (params, headers, files, messages, LLM/tool output) is validated or sanitized at the boundary
- [ ] Authorization is checked at the resource, not just the route — object-level access control for every id the caller supplies
- [ ] No string-built SQL/shell/template/HTML from user data — parameterized or escaped everywhere
- [ ] No secrets in code, config files, logs, error messages, or test fixtures; new secrets live in the secret store
- [ ] New dependencies were vetted (maintenance, downloads, license) and the lockfile audit is clean
- [ ] Sensitive data is encrypted in transit and at rest; nothing sensitive was added to caches, URLs, or client storage
- [ ] Error responses don't leak internals (stack traces, versions, queries, internal hostnames)
- [ ] Rate limiting / abuse controls cover any new unauthenticated or expensive endpoint
- [ ] Session/token handling unchanged, or changes reviewed against the auth design (expiry, rotation, revocation)
- [ ] Logging captures the security-relevant events of the change without logging the sensitive payloads themselves

## Checklist: Pre-release / production readiness

- [ ] Full test suite green on the exact artifact/commit being shipped
- [ ] The built artifact itself was smoke-tested (not the dev tree)
- [ ] Rollback is defined, scripted, and known-compatible with current data
- [ ] Migrations shipped ahead of, and compatible with, the previous app version (expand before contract)
- [ ] Config for the target environment is complete; no localhost/dev defaults can leak through
- [ ] Observability in place: key metrics, alerts on the new failure modes, logs with correlation ids
- [ ] Load expectations stated and, for hot paths, tested at expected peak
- [ ] Feature flags default safe; kill switch exists for the riskiest new path
- [ ] Changelog written; support/on-call informed of what is changing and what "bad" would look like
- [ ] Bake/monitoring window and its abort thresholds agreed before the deploy starts

## Checklist: Database migration safety

- [ ] The migration is a versioned file run by the migration tool — no ad-hoc production SQL
- [ ] Additive first: no destructive step (drop/rename/type-narrow) rides with the code that stops needing it
- [ ] Reversibility stated: the down path exists, or irreversibility is explicit and accepted in review
- [ ] Backfill is batched, idempotent, and resumable; expected runtime estimated against table size
- [ ] Lock behavior checked: no long exclusive locks on hot tables (online/concurrent index builds where supported)
- [ ] Previous app version runs correctly against the migrated schema (tested, not assumed)
- [ ] A recent backup/restore point exists and the restore path has actually been exercised
- [ ] Constraints and indexes accompany the new columns the queries will filter on
- [ ] Deployment order documented: migrate vs deploy sequence, and what happens if only one lands

## Checklist: API breaking-change review

- [ ] The change is classified: additive (safe) vs breaking (removal, rename, type change, semantics change, stricter validation)
- [ ] Breaking changes ship under a new version, not silently in place
- [ ] Deprecation is announced with a timeline; old surface keeps working through the announced window
- [ ] Known consumers identified and notified; unknown-consumer risk assessed (public APIs: assume unknown consumers exist)
- [ ] Error envelope, status codes, and pagination behavior unchanged — or the change is part of the versioned break
- [ ] Contract tests / OpenAPI spec updated and published with the change
- [ ] SDKs, docs, and examples updated in the same release
- [ ] Telemetry can distinguish old-surface vs new-surface traffic, so retirement is data-driven

## Checklist: Accessibility pass

- [ ] Semantic elements used for meaning (headings in order, lists, buttons vs links, landmarks) — no div-as-button
- [ ] Every interactive element is keyboard-reachable and operable; no keyboard traps
- [ ] Focus is visible, and managed on route changes, dialogs, and dismissals (returns to the trigger)
- [ ] Images and icons have appropriate alt text; decorative ones are hidden from assistive tech
- [ ] Form fields have programmatic labels; errors are announced and associated with their fields
- [ ] Text contrast meets WCAG AA (4.5:1 normal, 3:1 large/UI components)
- [ ] Nothing conveys meaning by color alone; state has a second signal (icon, text, pattern)
- [ ] Dynamic updates announce via live regions where users must know (async results, toasts)
- [ ] Page/zoom reflows to 200% without loss; touch targets meet minimum size
- [ ] Screen-reader pass on the changed flows (at least one: VoiceOver/NVDA) actually performed

## Checklist: Performance review

- [ ] The performance claim has a measurement: baseline and after, same harness, stated percentile
- [ ] Database queries per operation counted; no new N+1; new filters are index-backed (EXPLAIN checked)
- [ ] Payloads bounded: pagination or streaming on every list endpoint; no unbounded IN-memory collections
- [ ] Caching deliberate: what is cached, for how long, and how it invalidates — stated, not implied
- [ ] Frontend budgets respected: bundle size delta checked; heavy work off the main thread; assets sized/lazy where large
- [ ] Hot-path allocations and synchronous I/O reviewed; nothing blocking sits inside a per-request loop
- [ ] Timeouts and backpressure exist on every new external call (no unbounded queues or retries)
- [ ] Load behavior at expected peak stated; degradation mode is graceful, not cliff-edge

## Checklist: New-repo bootstrap

- [ ] LICENSE chosen and present; ownership/maintainers stated
- [ ] README: what it is, how to install, how to run, how to test — all verified by running them
- [ ] `.gitignore` correct for the stack; no build artifacts or secrets tracked
- [ ] Lockfile committed; dependency versions pinned by policy
- [ ] CI runs lint + typecheck + tests on every PR and is a required check
- [ ] Formatter and linter configured and enforced (locally and in CI), not just recommended
- [ ] Test harness wired with at least one real test proving the pipeline
- [ ] Security baseline: secret scanning on, dependency update automation on, SECURITY.md contact
- [ ] Contribution conventions stated (branch/commit/PR style); CODEOWNERS if review routing matters
- [ ] Environment template (`.env.example`) documents every required variable without real values

## Checklist: Dependency-adding decision

- [ ] The need is real: stdlib/platform or ~50 lines of own code cannot reasonably cover it
- [ ] Maintenance signal healthy: recent releases, responsive issues, more than one maintainer (or vendorable size)
- [ ] License is compatible with the project's distribution model
- [ ] Transitive cost inspected: dependency count and install size delta are acceptable
- [ ] Security posture checked: no open critical advisories; package name typo-checked against the intended one
- [ ] API surface used is small enough to wrap — the codebase depends on your wrapper, not the library, where feasible
- [ ] Alternatives compared (including "do nothing"); the winner's argument is written in the PR
- [ ] Removal path imaginable: what would migrating off it cost later?

## Checklist: Incident postmortem completeness

- [ ] Timeline reconstructed with timestamps: onset, detection, mitigation, resolution — detection gap and mitigation gap computed
- [ ] Impact quantified: users/requests/data affected, duration, SLO/error-budget consumption
- [ ] Root cause(s) reach process/system level — not "human error", but why the system let the error through
- [ ] Contributing factors and what went well both recorded (detection, tooling, luck)
- [ ] Every action item has an owner and a due date; items verified as filed in the tracker
- [ ] Actions address detection and blast-radius, not only the specific trigger
- [ ] The document is blameless: no names attached to fault, only to actions
- [ ] Postmortem shared beyond the responding team; recurring-pattern check against previous incidents done

## Checklist: AI-agent config review

- [ ] Tool grants are least-privilege: the agent has only the tools/scopes its tasks need; write/destructive tools justified individually
- [ ] Irreversible or outward-facing actions (deploy, delete, publish, spend, external messages) require approval or are gated by policy
- [ ] Hooks reviewed: what they execute, with what input; they fail safe (non-blocking on error) and are pinned to reviewed scripts
- [ ] MCP connectors pass the budget test (universal + session-stateful); each one's schema tax is accepted knowingly; unused connectors removed
- [ ] No secrets in prompts, skills, hooks, or MCP configs; credentials come from the environment/secret store
- [ ] Untrusted content (fetched pages, tool output, user files, PR comments) is treated as data — the config nowhere instructs the agent to obey embedded instructions
- [ ] Prompt-injection surfaces enumerated: which tools return third-party text, and what the blast radius of a hijacked turn would be
- [ ] Skills/rules don't contradict each other; precedence between project rules and global rules is defined
- [ ] Logging/audit exists for agent actions with enough detail to reconstruct a session
- [ ] A kill path exists: how a human stops the agent, revokes its credentials, and rolls back its changes

## Checklist: Prompt/skill quality review

- [ ] One concern per skill/prompt; mixed concerns are split
- [ ] The description states TRIGGER and DO-NOT-TRIGGER conditions concretely enough to route against
- [ ] Instructions are imperative and testable — a reviewer could verify compliance from a transcript
- [ ] "Answer from files/data, not memory" applies wherever the content can go stale; no hardcoded counts or versions
- [ ] Output contract specified: shape, length, and what to do when the answer is unknown
- [ ] Anti-patterns / failure modes are listed, not just the happy path
- [ ] Examples (if present) match the current instructions — stale examples are worse than none
- [ ] Token footprint proportionate: always-loaded text minimal, bulk pushed to on-demand references
- [ ] No secrets, personal paths, or environment-specific absolutes embedded
- [ ] Tried against at least one realistic input; the produced behavior matched the intent

## Checklist: Estimation sanity

- [ ] The work is decomposed into slices small enough that each is estimable from experience
- [ ] Unknowns are listed explicitly; each big unknown has a time-boxed spike rather than a padded guess
- [ ] Dependencies on other people/teams/systems identified, with their availability checked, not assumed
- [ ] The estimate includes tests, review cycles, docs, and deploy — not just "code complete"
- [ ] Historical calibration consulted: what did the last similar task actually take?
- [ ] Buffer is explicit and proportional to the unknowns, not a silent multiplier
- [ ] The first slice ships something observable early enough to correct course
- [ ] The "wrong estimate" cost is known: what happens, and who must hear it, if this takes 2×?

## Checklist: Code deletion safety

- [ ] All references found: code, configs, CI, scripts, docs, scheduled jobs — searched by symbol and by string
- [ ] Runtime evidence consulted where available (logs/metrics/flag analytics show the path is genuinely unused)
- [ ] Feature flags referencing the code are retired in the same change, not orphaned
- [ ] Public surface check: nothing external (API consumers, plugins, other repos) imports what is being removed
- [ ] Data retention decided: persisted data the code owned is migrated, archived, or explicitly scheduled for deletion
- [ ] Tests covering only the deleted behavior are removed with it; tests that also guard kept behavior are preserved
- [ ] Docs and examples referencing the feature are updated in the same PR
- [ ] The deletion is its own commit/PR, revertable in one step

## Using checklists well

- A checklist gates a decision; it does not replace judgment — an all-green list with a bad design is still a bad design, and one red item may be consciously waived with a recorded reason.
- Tailor per repository: strike items that structurally cannot apply, add the repo's recurring failure as a new item — then keep each list under 20 items or it stops being run.
- Check items with evidence ("ran `EXPLAIN`, index used"), not optimism; an unverifiable item is answered "unknown", never silently checked.
- The right time is before the irreversible step: review before merge, readiness before deploy, deletion safety before the delete lands.
