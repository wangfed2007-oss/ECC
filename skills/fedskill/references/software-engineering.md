# Universal Software Engineering

Project-agnostic engineering playbook: architecture decisions, API design, data modeling, testing, git workflow, CI/CD, security, performance, debugging, refactoring, documentation, and incident response. Load this volume when making engineering decisions that are not tied to a specific language or framework — choosing an architecture, designing an API, planning a migration, structuring tests, or responding to an incident. Query the section that matches the decision at hand; each entry is self-contained.

## Architecture Decision Method

### Decision procedure

1. **Constraints first.** List hard constraints before options: team size/skills, latency SLO, data volume, compliance, budget, deadline. Options that violate a hard constraint are dead — do not debate them.
2. **Reversibility check.** Classify the decision: one-way door (schema of a public API, primary datastore, language) vs two-way door (library choice, internal module boundary). Spend decision effort proportional to reversal cost. Two-way doors: decide fast, document, move on.
3. **Boring-tech bias.** Default to technology the team already operates. Each project earns a small number of "innovation tokens" — spend at most one or two on novel tech, and only where it addresses a core constraint.
4. **Cost of the null option.** Always evaluate "do nothing / defer" as a candidate. Premature architecture is the most expensive kind.
5. **Write it down.** Any decision that survives step 2 as a one-way door gets an ADR (template below).

Trap: choosing architecture by résumé or conference talk. The question is never "is X good?" but "is X good for these constraints, operated by this team?"

### ADR template (inline)

```markdown
# ADR-NNNN: <short decision title>

- Status: Proposed | Accepted | Deprecated | Superseded by ADR-MMMM
- Date: YYYY-MM-DD
- Deciders: <names/roles>

## Context
What forces are at play: constraints, requirements, current pain. Facts, not opinions.

## Decision
"We will <decision>." One sentence, active voice.

## Options considered
1. <option> — pros / cons (one line each)
2. <option> — pros / cons

## Consequences
What becomes easier, what becomes harder, what we now must do (ops, training, migration).

## Revisit trigger
Concrete condition that reopens this decision (e.g. ">10k writes/s", "team > 20 devs").
```

Keep ADRs in-repo (e.g. `docs/adr/`), numbered, immutable once Accepted — supersede, never edit history.

### Architecture styles

| Style | Choose when | Avoid when |
|---|---|---|
| Modular monolith | Small/medium team, one deployable, unclear domain boundaries, need speed | Independent scaling/deploy cadence per domain is a hard requirement |
| Microservices | Many teams needing independent deploys, well-understood domain boundaries, mature ops (observability, CI/CD) | Team < ~20 engineers, domain still shifting, no platform/ops investment |
| Event-driven / pub-sub | Loose coupling between producers/consumers, async workflows, fan-out, audit trails | Strong request/response consistency needed; debugging budget is thin |
| Serverless / FaaS | Spiky or low traffic, glue logic, minimal ops staff, per-request billing wins | Long-running processes, steady high load (cost), tight latency (cold starts), heavy local state |
| CQRS + event sourcing | Audit/history is a product requirement, read and write models diverge sharply | CRUD app; team unfamiliar — this is a high-tax pattern |
| Layered (n-tier) | Simple CRUD, small team, conventional web app | Complex domain logic that leaks across layers; use a domain-centric structure instead |
| Hexagonal / ports-and-adapters | Domain logic must be testable without infra; multiple delivery mechanisms (HTTP, CLI, queue) | Thin CRUD where the abstraction outweighs the logic |
| Monorepo (org style) | Shared code, atomic cross-project refactors, unified tooling | No investment in build tooling (bazel/nx/turbo-class); repos owned by separate orgs |

Trap: microservices adopted to fix a code-organization problem. Distribution adds failure modes; module boundaries inside a monolith solve the same coupling problem without the network.

vs: **monolith vs monorepo** — deployment unit vs repository layout. A monorepo can hold many services; a monolith can live in many repos (don't).

## API Design

### Resource modeling

- Model **nouns** (resources), not verbs. `POST /orders`, not `POST /createOrder`. Verbs that don't map to CRUD become sub-resources or actions: `POST /orders/{id}/cancel`.
- Plural resource names, lowercase, hyphenated: `/purchase-orders/{id}/line-items`.
- Nest at most one level; beyond that, promote to a top-level collection with a filter: `/comments?post_id=…` not `/users/{u}/posts/{p}/comments/{c}`.
- Use the HTTP verb semantics: GET safe+idempotent, PUT/DELETE idempotent, POST neither, PATCH partial update. Return 405 for unsupported verbs, not 404.
- IDs: opaque strings from day one (UUIDv7/ULID for sortability). Never expose auto-increment integers — enumeration risk and shard-hostile.

### Versioning strategies

| Strategy | Form | Notes |
|---|---|---|
| URI version | `/v1/orders` | Most visible, easiest routing/caching. Default choice. |
| Header version | `Accept: application/vnd.api+json;version=2` | Cleaner URIs; harder to test with curl/browsers. |
| Query param | `?api-version=2024-06-01` | Common in cloud APIs; date-based versions age well. |
| No version (additive only) | evolve in place | Works if you enforce a strict additive-change policy; pair with deprecation headers. |

Trap: versioning every endpoint independently. Version the API surface, not routes — mixed versions per route becomes untestable.

### Error envelope

Standardize one error shape across the API (RFC 9457 "problem details" or equivalent):

```json
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Balance 12.50 is below the requested 100.00.",
  "instance": "/transfers/abc123",
  "request_id": "req_9f2c",
  "errors": [{ "field": "amount", "code": "too_large" }]
}
```

Rules: machine-readable `type`/`code`, human-readable `detail`, always a `request_id` for support correlation, field-level errors as an array. Never leak stack traces, SQL, or internal hostnames. 4xx = caller's fault, 5xx = yours — don't return 200 with `{"error": …}`.

### Pagination

- **Cursor-based** (opaque `next_cursor` token): default. Stable under inserts/deletes, index-friendly.
- **Offset/limit**: only for small, admin-facing, or truly random-access lists. Trap: `OFFSET 100000` scans and discards 100k rows; deep pagination degrades linearly.
- Always return `next_cursor` (null when exhausted) and enforce a max page size server-side.
- Keyset pagination = cursor pagination implemented as `WHERE (created_at, id) > (:ts, :id) ORDER BY created_at, id` — the tiebreaker column is mandatory.

### Idempotency keys

- For any non-idempotent mutation that a client may retry (payments, order creation): accept an `Idempotency-Key` header (client-generated UUID).
- Server stores key → result for a TTL (24h typical); replays return the stored response with the same status code.
- Key scope: per endpoint + per principal. A retry with the same key but a **different body** is a 409/422, never a silent replay.
- Trap: implementing idempotency only at the HTTP layer while the handler enqueues jobs — the dedupe must cover the side effect, not just the response.

### Breaking-change policy

- **Additive is safe**: new optional fields, new endpoints, new enum values *only if clients are told to ignore unknown values*.
- **Breaking**: removing/renaming fields, changing types/semantics, tightening validation, changing error codes clients branch on.
- Process: announce → dual-support window with `Deprecation` + `Sunset` headers → monitor usage telemetry → remove only when traffic ~0 or deadline passed.
- Contract tests (consumer-driven, e.g. Pact-style) in CI catch accidental breaks before release.

### Protocol choice

| Criterion | REST/HTTP+JSON | GraphQL | gRPC |
|---|---|---|---|
| Choose when | Public APIs, CRUD, cacheability, broad client compat | Many client shapes (mobile/web) over the same graph; frontend-driven iteration | Internal service-to-service, low latency, streaming, polyglot with strict contracts |
| Avoid when | Client needs highly variable projections (over/under-fetch) | Simple CRUD; team can't invest in query cost control, N+1 resolvers, caching | Browser-facing without a proxy; human debuggability is a priority |
| Contract | OpenAPI | SDL schema | Protobuf |
| Caching | HTTP-native (best) | Hard (POST-everything; needs persisted queries) | App-level only |
| Failure model | Status codes | 200 + `errors[]` (Trap: monitors must inspect body) | Rich status codes + deadlines |

Webhooks: sign payloads (HMAC), deliver at-least-once, require consumer idempotency, retry with exponential backoff, provide a redelivery UI/API.

## Data Modeling & Storage

### Schema design heuristics

- Model the **queries**, then the data. List the top 10 access patterns before drawing tables.
- Normalize until it hurts (3NF default in SQL), denormalize where a measured read pattern demands it — and record the invariant you now maintain by hand.
- Every table: surrogate primary key, `created_at`/`updated_at`, and explicit FKs with declared `ON DELETE` behavior.
- Nullable columns are a tri-state trap (`NULL` vs empty vs missing semantics). Prefer NOT NULL with defaults; make nullability a deliberate choice.
- Enum-like columns: use a lookup table or DB enum with a documented extension policy; free-text status columns rot.
- Soft delete (`deleted_at`) only when un-delete or audit is a real requirement — it taxes every query and index forever. Otherwise hard-delete with an audit log.
- Store money as integer minor units (cents) or `DECIMAL` — never floats. Store timestamps in UTC (`timestamptz`); convert at the edge.

### Migration discipline: expand-migrate-contract

1. **Expand**: add the new column/table/index alongside the old. Deploy code that writes both, reads old.
2. **Migrate**: backfill in batches (throttled, resumable, idempotent). Flip reads to new. Verify parity (counts, checksums).
3. **Contract**: after a soak period, remove old-path code, then drop the old column in a later release.

Rules: every migration is forward-only and reversible-by-new-migration; never edit an applied migration; schema change and destructive data change never ship in the same deploy; large-table `ALTER`s use online/`CONCURRENTLY` variants. Trap: adding a NOT NULL column with a default on a huge table can lock/rewrite — add nullable, backfill, then add the constraint (`NOT VALID` → `VALIDATE`).

### Index strategy

- Index what you filter, join, and sort on — in that combined shape. Composite index column order: equality columns first, then range, then sort.
- Covering indexes (INCLUDE columns) eliminate heap fetches for hot queries.
- Every index taxes every write and consumes cache; audit unused indexes (`pg_stat_user_indexes` or equivalent) quarterly.
- Low-cardinality columns (booleans, small enums) rarely deserve their own index; partial indexes (`WHERE status = 'pending'`) often do.
- Foreign keys are not auto-indexed in most SQL engines — index the FK column on the child side or joins and cascades will scan.
- Verify with the planner (`EXPLAIN ANALYZE`), not intuition.

### Transaction boundaries

- One transaction = one business invariant. Keep them short; never hold a transaction across a network call, user interaction, or queue publish.
- Know your isolation level; most engines default to READ COMMITTED — write-skew and lost updates are possible. Use `SELECT … FOR UPDATE`, optimistic version columns, or SERIALIZABLE (with retry loops) for contended invariants.
- Cross-service consistency: no distributed transactions — use the outbox pattern (write event in same tx, relay asynchronously) and sagas with compensating actions.
- Trap: "transaction" in an ORM often ends at flush, not commit; verify where the boundary actually is.

### Datastore choice

| Store | Choose when | Avoid when |
|---|---|---|
| Relational (SQL) | Default. Invariants, joins, ad-hoc queries, transactions, reporting | Extreme write throughput on trivially partitionable data (rare) |
| Document (Mongo-class) | Aggregate-shaped data read/written whole, schema variance per record | Cross-document invariants, many-to-many relations, heavy ad-hoc joins |
| Key-value (Redis-class) | Caching, sessions, rate limits, queues, leaderboards; µs latency | System of record; anything needing queries beyond get/put |
| Wide-column (Cassandra-class) | Massive write volume, known query patterns, multi-region availability | Evolving query patterns, joins, small datasets |
| Vector | Semantic search / RAG over embeddings; ANN at scale | <~1M vectors — pgvector inside your SQL DB is simpler and transactional |
| Graph | Multi-hop traversals as the core query (fraud rings, recommendations, dependency graphs) | 1–2 hop joins — SQL handles those fine |
| Search (Lucene-class) | Full-text relevance, faceting, typo tolerance | System of record (it's an index — rebuildable, eventually consistent) |

Rule: start with one relational database; add specialized stores only when a measured pattern outgrows it. Every extra store is a consistency boundary and an ops burden.

## Testing Strategy

### Shape of the suite

- **Pyramid** (many unit, some integration, few e2e): fits logic-heavy systems, libraries, backends.
- **Trophy** (emphasis on integration tests through real wiring, thin unit and e2e layers): fits web apps where most bugs live in the seams (HTTP-to-handler, handler-to-DB, component-to-store).
- Either way the invariants hold: fast and deterministic at the bottom, few and forgiving at the top; cost and flake risk grow with scope.

| Level | Test this | Not this |
|---|---|---|
| Unit | Pure logic, branching, edge cases, error paths, algorithms | Framework glue, trivial getters, private internals |
| Integration | Component + real collaborators (real DB in a container, real HTTP routing), contract with schema | Third-party SaaS (fake at the boundary), full user journeys |
| E2E | 5–15 critical user journeys (signup, checkout, core workflow) | Every permutation — push those down the stack |

vs: **sociable vs solitary unit tests** — sociable (real collaborators, fake only I/O) resists refactors better; solitary (mock everything) couples tests to implementation. Prefer sociable; mock at process boundaries only.

### Mechanics

- **Naming**: state behavior and condition, not method name — `rejects_expired_token`, `retries_then_fails_after_3_attempts`. A failing test's name should read as a bug report.
- **Arrange-Act-Assert**: three visible blocks, one logical assertion cluster per test. Multiple `act`s = split the test. Shared setup via builders/factories, not 200-line fixtures.
- **Determinism**: inject clock, RNG, and IDs; no real network; no shared mutable global state; no `sleep(n)` — wait on conditions with timeout. Order-independent: any test runnable alone and first.

### Flakes

Policy: a flaky test is a bug with a deadline. Quarantine (tagged, excluded from merge-blocking, tracked in an issue) → fix within a fixed window → delete if not worth fixing. Never normalize retry-until-green on the whole suite. Diagnose by class: timing (await conditions), isolation (leaked state between tests), infra (containers, ports), true nondeterminism in product code (that's a product bug).

### Coverage

Coverage is a **detector of untested code, not a proof of tested code**. Use it to find dead spots and to ratchet ("patch coverage on changed lines ≥ X%"), never as a target that invites assertion-free tests. Trap: 100% line coverage with zero meaningful assertions is worse than 70% with sharp ones — it manufactures false confidence. Mutation testing is the honest audit when it matters.

### TDD loop

1. **Red**: write the smallest failing test for the next behavior. Run it; watch it fail *for the expected reason*.
2. **Green**: write the minimum code to pass. Ugly is fine.
3. **Refactor**: clean up with the green bar as a safety net. Tests refactor too.
4. Commit on green; repeat in minutes-sized cycles.

Trap: skipping the "watch it fail" step — a test that passes before the code exists tests nothing.

## Git & Collaboration

### Branching

- **Trunk-based**: short-lived branches (< 2 days) merged to `main` continuously; incomplete features behind flags. Choose for teams with solid CI and feature-flag muscle. Pairs with continuous deployment.
- **Feature branches / GitHub flow**: branch per change, PR, merge. Fine at small scale; branches older than ~a week accumulate merge pain and review dread.
- Long-lived `develop`/release-train branches (git-flow): only for versioned, shipped software with parallel maintained releases. Overkill for services.

### Commits

Conventional commits: `type(scope): imperative summary` — types `feat`, `fix`, `docs`, `test`, `refactor`, `perf`, `chore`, `ci`, `build`. `feat`/`fix` drive changelog and semver automation; `!` or `BREAKING CHANGE:` footer marks majors. One logical change per commit; the diff should match the message. Trap: `chore: fixes` — a message that could describe any commit describes none.

### PRs and review

- Size: aim < ~400 changed lines; review quality collapses beyond that. Split by layer or by refactor-then-feature ("make the change easy, then make the easy change"). Stacked PRs for dependent chains.
- Description: what + why + how to verify + screenshots for UI. Link the issue. Self-review the diff before requesting review.
- Reviewer etiquette: review promptly (unreviewed PRs rot); comment on the code, not the author; distinguish blocking issues from `nit:` preferences; approve with nits rather than round-tripping trivia; ask questions instead of issuing verdicts when unsure.
- Author etiquette: respond to every comment (fix, or explain why not); don't force-push over a review in progress (append fixup commits, squash at merge so reviewers can diff the delta).

### Rebase vs merge

- Rebase **your own unshared/PR branches** to keep history linear and CI meaningful. Merge (or squash-merge) into `main`.
- **Never rewrite history others have based work on** — no force-push to `main`/shared branches, ever. `--force-with-lease` only on your own branches.
- Squash-merge: one commit per PR — clean history, loses intra-PR steps. Pick one repo-wide policy and enforce it in the forge settings.
- Trap: resolving the same conflicts repeatedly during a long rebase — enable `git rerere`.

### Releases and semver

- Tag releases: annotated tags `vMAJOR.MINOR.PATCH` on the exact released commit; CI builds from tags, not branch heads.
- Semver: MAJOR = breaking, MINOR = additive, PATCH = fixes. `0.x` = anything may break (minor is the breaking lane). Pre-releases: `1.2.0-rc.1`.
- Semver describes the **public contract**, not effort — a one-line breaking change is a major; a month of internal refactoring is a patch.
- Automate: conventional commits → generated changelog + version bump (release-please / semantic-release class tooling).

## CI/CD

### Pipeline stages, fail-fast order

Order by (cost × probability of failure): cheapest, most-likely-to-fail first.

1. Format/lint check (seconds)
2. Typecheck (seconds–minutes)
3. Unit tests (minutes, parallelized)
4. Build/package (also produces the artifact used by later stages)
5. Integration tests (containers for real deps)
6. Security scans (dependency audit, SAST, secret scan) — parallel with 5
7. E2E on a deployed preview/staging env
8. Deploy (gated: automatic to staging, promotion to prod)

Rules: **build once, promote the same artifact** through environments — never rebuild per environment. Every stage runnable locally with one command. Merge-blocking = stages 1–5; slower stages can gate deploy instead of merge. Keep merge-blocking wall time under ~10 minutes or devs will batch changes.

### Caching

Cache dependency stores (keyed on lockfile hash), compiler/build caches, and container layers (order Dockerfile: deps before source). Trap: caching test results or poorly-keyed build outputs causes "works in CI, broken artifact" — cache inputs aggressively, artifacts conservatively, and key caches on the exact content that invalidates them.

### Flaky-test policy in CI

Auto-detect (pass-on-retry = flake), auto-quarantine with an issue, weekly triage, hard cap on quarantine list size. Retries are diagnostic instrumentation, not a fix. A red main is a stop-the-line event: revert first, investigate after.

### Deployment strategies

| Strategy | Mechanics | Choose when | Cost |
|---|---|---|---|
| Rolling | Replace instances N at a time | Default for stateless services | Two versions live simultaneously during roll |
| Blue-green | Full parallel env, switch traffic atomically | Instant cutover/rollback needed; DB compatible with both | 2× infra during deploy |
| Canary | Route small % to new version, watch metrics, ramp | High-blast-radius services; you have real SLO metrics to compare | Needs traffic splitting + automated analysis |
| Feature flags | Deploy dark, enable per cohort at runtime | Decouple deploy from release; gradual product rollout | Flag debt — expire flags aggressively |

All strategies require **N and N+1 to coexist**: backward-compatible schema (expand-migrate-contract) and API changes. Trap: a "rollback" of code cannot roll back a destructive migration — that's why contract comes last and late.

### Rollback discipline

- Rollback is the **first** mitigation, not the last resort. Practice it; an untested rollback path is a rumor.
- Automate triggers where possible: canary analysis breaching SLO auto-reverts.
- Roll forward only when rollback is impossible (irreversible migration) — and treat that as a process failure to fix.
- Keep deploys small and frequent: rollback scope = deploy scope.

## Security Baseline

### OWASP-driven checklist

- [ ] **Access control**: deny by default; enforce object-level authorization on every request (IDOR is the #1 real-world bug — "can this principal act on this resource?", not just "is logged in?"). No client-side-only enforcement.
- [ ] **Injection**: parameterized queries everywhere; no string-built SQL/shell/LDAP. Template engines auto-escape; `dangerouslySetInnerHTML`-class APIs justified in review.
- [ ] **Authentication**: rate-limit and lock on failures; password hashing with argon2id/bcrypt (never fast hashes); MFA available; session IDs rotated on privilege change.
- [ ] **Crypto failures**: TLS ≥1.2 everywhere (internal too); no homegrown crypto; no secrets/PII in logs or URLs.
- [ ] **Misconfiguration**: prod ≠ debug mode; default creds removed; security headers (CSP, HSTS, X-Content-Type-Options, frame-ancestors); cloud storage private by default.
- [ ] **SSRF**: outbound fetches of user-supplied URLs go through an allowlist/proxy; block link-local/metadata IPs (169.254.169.254).
- [ ] **XXE/deserialization**: XML external entities off; never deserialize untrusted data with polymorphic/native deserializers.
- [ ] **Logging/monitoring**: authn events, authz failures, and admin actions logged and alertable; logs tamper-evident, PII-scrubbed.

### Secrets

- Secrets live in a secret manager (Vault/cloud KMS-backed store), injected at runtime — never in code, config files in git, images, or CI logs.
- Rotation is designed-in (dual-secret windows); any secret that has ever touched git history is compromised — rotate, don't just delete the file. Pre-commit + CI secret scanning (gitleaks-class) as a tripwire.
- Prefer short-lived, machine-identity credentials (OIDC federation from CI to cloud) over long-lived static keys.

### Dependency hygiene

- Lockfiles committed always; installs in CI use the frozen/`ci` mode so builds are reproducible.
- Automated vulnerability audit in CI + scheduled (deps rot even when code doesn't); auto-PR bots (renovate-class) with grouped, tested updates.
- Generate an SBOM (CycloneDX/SPDX) at build; you cannot patch what you cannot enumerate.
- Trap: typosquatting and hallucinated package names — verify a package exists and is the canonical one before adding it; pin new deps to exact versions and review install scripts.

### Design-level

- **Authn vs authz**: authentication = who you are (verify once, at the edge); authorization = what you may do (enforce at every resource access, server-side, close to the data). Never infer authz from authn ("logged in" ≠ "allowed").
- **Validate at trust boundaries**: every input crossing a boundary (user → server, service → service, file/queue → parser) is validated against a schema — type, length, range, allowlist. Canonicalize before validating. Internal services still validate: zero trust between services.
- **Encryption**: at rest — full-disk/volume + application-level encryption for high-value fields (with key rotation via KMS); in transit — TLS externally and mTLS or a service mesh internally. Classify data; encryption scope follows classification.
- Least privilege everywhere: DB accounts per service with minimal grants, scoped tokens, short-lived cloud roles.

## Performance Method

### Method

1. **Measure first.** No optimization without a profile or production metric showing where time/resources actually go. Guessed bottlenecks are usually wrong.
2. Set a target (SLO): "p99 checkout < 400 ms" — otherwise optimization never terminates.
3. Fix the top item, re-measure, repeat. One change at a time; keep the benchmark harness in the repo.

**Percentiles, not averages**: latency is skewed; the mean hides the tail your users feel. Track p50/p95/p99. Trap: averaging percentiles across hosts is meaningless — aggregate from histograms. Tail latency compounds: a page making 10 backend calls hits a backend's p99 far more often than 1% of the time.

### The four resources

| Resource | Saturation signals | Typical fixes |
|---|---|---|
| CPU | High utilization, run-queue depth, throttling (containers) | Algorithmic complexity, batching, caching computed results, parallelism |
| Memory | GC pressure/pauses, RSS growth, OOM kills, swap | Leaks, oversized caches, streaming instead of buffering, pooling |
| Disk IO | iowait, queue depth, fsync latency | Batching writes, better indexes (less scanning), SSD/provisioned IOPS, async |
| Network | Bandwidth, RTT × round-trips, connection churn | Fewer round-trips (batch/pipeline), compression, connection pooling, locality |

USE method per resource: Utilization, Saturation, Errors. For request-serving systems, RED per service: Rate, Errors, Duration.

### Caching

Layers, outermost first: browser/CDN → gateway/edge → application (in-process or Redis-class) → database (buffer pool, materialized views). Cache as close to the user as staleness tolerance allows.

- Every cache entry gets a TTL — TTL is the backstop, invalidation the optimization.
- Invalidation patterns: TTL-only (simplest), write-through, cache-aside with explicit delete-on-write, event-driven purge. Version-keyed caches (`user:123:v42`) dodge invalidation by changing the key.
- Traps: **stampede** (hot key expires, thundering herd rebuilds — use locking/single-flight or jittered TTLs); caching negative results without a short TTL; treating an eventually-consistent cache as the source of truth.

### N+1 hunting

Symptom: query count scales with result rows (1 query for the list + N for children). Detect: per-request query counters/logging in dev, APM traces showing repeated identical-shape queries, assertions in tests capping query count. Fix: eager-load/join, batch by IDs (`WHERE id IN (…)`), or a dataloader (batch + per-request cache) in GraphQL/resolver contexts. The same pathology exists for HTTP calls in loops — batch endpoints or concurrency with limits.

### Profiling workflow

1. Reproduce the load (realistic data volume — dev-sized data hides everything).
2. Profile CPU (sampling profiler → flame graph: widest frames = where time goes), then allocations, then IO/queries (APM trace or query log with timings).
3. Form a hypothesis from the profile, change one thing, re-run the same benchmark.
4. In production: continuous profiling + distributed tracing beat one-off local profiles; wall-clock time ≠ CPU time — a "slow" function may just be waiting.

## Debugging Method

### The loop

1. **Reproduce** — reliably, minimally, ideally as a failing test. No reproduction ⇒ collect evidence (logs, traces, core dumps) until you have one. An intermittent bug reproduced 1-in-10 is still a reproduction — loop it.
2. **Isolate** — shrink the failing case: smallest input, fewest components, shortest path. Half the time isolation reveals the bug by itself.
3. **Bisect** — binary search the difference between working and broken (see below).
4. **Instrument** — add targeted logging/asserts/breakpoints to test the current hypothesis, not scattershot prints.
5. **Fix** — the root cause, not the symptom. Ask "why did this happen?" enough times to reach a process or design cause, and "where else does this same pattern exist?"
6. **Regression-test** — the failing test from step 1 goes into the suite before the fix merges. A bug fixed without a test is a bug scheduled for re-release.

### Binary search over anything

- **Commits**: `git bisect start; git bisect bad; git bisect good <ref>` — with a repro script, `git bisect run ./repro.sh` automates the whole search. log₂(1000 commits) ≈ 10 steps.
- **Config**: diff working vs broken environment; toggle halves of the delta until the culprit flag/env var/version emerges.
- **Data**: split the failing input in half; recurse into the failing half (delta debugging). Works for "which of 10k records breaks the import?"
- **Code path**: disable half the pipeline/middleware/features; recurse.

### Reading stack traces

- Read top-down for *where it blew up*, then find the **deepest frame in your own code** — that's usually where to look; frames in framework/stdlib are consequence, not cause.
- Check for a `Caused by:` / inner exception chain — the last (innermost) cause is the real one; wrappers add context, not information.
- The trace shows where the error was *raised*, not where the bad state was *created* — a `NullPointerException` names the victim, not the culprit. Work backwards to where the value went wrong.
- Async/promise traces are often truncated at the scheduler boundary; enable long/async stack traces in dev.

### Hypothesis discipline

- Keep a written log during nontrivial hunts: hypothesis → experiment → result. Prevents re-testing the same idea at hour three and turns the hunt shareable/resumable.
- Rubber-duck: explain the problem end-to-end, out loud or in writing, stating every assumption — the bug usually lives in an assumption you'd never questioned ("it can't be the cache… have I actually verified the cache?").
- Trap: debugging by changing code until it works. If you don't know *why* the fix works, the bug isn't fixed — it moved.
- Trap: "it can't be X" — the bug is disproportionately often in X, the component everyone trusts. Verify, don't assume; suspect your own newest code first, the platform last.

## Refactoring & Code Health

### Smells table

| Smell | Why it hurts | Refactor |
|---|---|---|
| Long function (> ~40 lines, mixed abstraction levels) | Can't hold it in your head; untestable branches | Extract functions named for intent |
| Duplicated logic (not just similar text) | Fixes must be found N times; drift guarantees bugs | Extract shared function/module — but see Trap below |
| Large class / god object | Every change touches it; merge conflicts; unclear ownership | Split by responsibility; extract collaborators |
| Feature envy (method mostly uses another object's data) | Logic far from data it governs | Move method to the data's home |
| Shotgun surgery (one change → edits in many files) | Cohesion inverted; easy to miss a site | Consolidate the concern behind one module |
| Primitive obsession (`string userId`, `float money`) | Invariants unenforced; wrong-argument bugs typecheck fine | Introduce value types (Money, EmailAddress) |
| Long parameter list | Call sites unreadable; params travel in packs | Parameter object; builder for construction |
| Boolean flag parameter | `f(true, false)` is unreadable; function does two things | Split into two named functions or an enum |
| Deep nesting | Cognitive stack overflow | Guard clauses, early returns, extract predicates |
| Comments explaining *what* the code does | Rot silently; signal unclear code | Rename/extract until the code says it; keep only *why* comments |
| Dead code / commented-out blocks | Reader must decide if it matters; it never does | Delete — git remembers |
| Speculative generality (unused hooks, "might need it") | Complexity paid now for value never claimed | Delete; rebuild when the need is real (YAGNI) |

Trap: DRY applied to coincidental similarity — two pieces of code that look alike but serve different masters will diverge; premature unification couples them. Duplication is cheaper than the wrong abstraction.

### Practices

- **Boy-scout rule**: leave code slightly better than found — a rename, an extracted function, a test — scoped to files you were already touching. Not a license to bundle a rewrite into a feature PR; ship large refactors as separate, behavior-preserving PRs ("refactor" and "behavior change" never share a commit).
- **Strangler fig** (legacy replacement): put a routing facade in front of the legacy system; build new functionality in the new system; migrate slice by slice; the legacy system shrinks until deletable. Never big-bang rewrite a live system — the strangler keeps both running and reversible at every step. Requires: facade seam, comparison/shadow traffic to validate parity, and a kill date per slice.
- **Dead-code deletion**: delete unreachable code, unused flags, expired experiments the moment they're confirmed dead. "Might need it later" is what version control is for. Keep a periodic sweep (coverage + reference analysis + flag-expiry dates); deleted code is a maintenance surface, security surface, and reader tax removed.
- **Comment policy**: comments explain **why** — constraints, non-obvious tradeoffs, links to issues/specs, warnings ("order matters because…"). Code explains *what* via names and structure. A comment paraphrasing the next line is noise; a comment justifying a weird-looking correct thing is gold. `TODO(name, issue#)` or it will outlive you.

## Documentation

### README anatomy

Order matters — answer the reader's questions in the order they ask them: (1) one-sentence what-and-why; (2) quickstart — the copy-paste path from clone to running in minutes; (3) usage examples of the 3 most common tasks; (4) configuration reference or link; (5) how to run tests / contribute (or link CONTRIBUTING); (6) license + support channel. Trap: a README that starts with architecture philosophy — nobody reading a README wants your philosophy before `npm install` works.

### Document types (Diátaxis lens)

| Type | Reader's mode | Form |
|---|---|---|
| Tutorial | Learning, first contact | Guided, guaranteed-success path |
| How-to guide | Working, has a task | Recipe: steps to one goal |
| Reference | Working, needs facts | Complete, structured lookup (this file's style) |
| Explanation | Studying, wants understanding | Context, tradeoffs, "why is it like this" |

Don't mix modes in one page — a reference interrupted by tutorial prose serves neither reader.

### Runbooks

One per alert/operational task: symptom → verify (exact queries/dashboards) → mitigate (copy-paste commands) → escalate (who, when) → related incidents. Written for the 3 a.m. responder who is not the author: zero unexplained context, every command pasteable. Test runbooks in game days; a runbook that has never been executed is fiction.

### Changelog

Human-written summaries per release for humans (`Keep a Changelog` format: Added/Changed/Deprecated/Removed/Fixed/Security) — generated commit lists are an input, not the product. Every breaking change carries a migration instruction. vs release notes: changelog is complete and chronological; release notes are curated highlights for announcement.

### Docs-as-code

Docs live in the repo, reviewed in the same PRs that change behavior ("docs updated?" is a review checklist item), built and link-checked in CI, published on merge. Stale docs are worse than no docs — they teach wrong things with confidence. Delete or mark docs you won't maintain.

### When a diagram earns its place

Draw when structure or flow beats prose: component boundaries and dependencies (C4 context/container level), sequence of a nontrivial cross-service interaction, state machines, data lifecycle. Skip when a list says it as well. Rules: diagram-as-code (Mermaid-class) so it diffs and stays reviewable; every diagram has a title, a scope, and an owner; a wrong diagram is worse than none — fewer diagrams, maintained.

## Incident Response

### Severity levels

| Level | Definition | Response |
|---|---|---|
| SEV1 | Full outage, data loss, security breach, revenue-critical path down | All-hands page, incident commander assigned, exec/status-page comms, 24/7 until mitigated |
| SEV2 | Major feature degraded, significant subset of users affected, no workaround | Page on-call, IC assigned, status page if user-visible |
| SEV3 | Minor degradation, workaround exists, limited blast radius | Ticket + working-hours response |
| SEV4 | Cosmetic / negligible impact | Backlog |

When in doubt, declare high and downgrade — declaring is cheap, delay is not. Anyone may declare an incident; no permission needed.

### Roles

- **Incident Commander (IC)**: owns coordination and decisions; explicitly *not* hands-on-keyboard. Single authority; everyone routes through IC.
- **Ops/subject leads**: hands-on investigation and mitigation.
- **Comms lead**: stakeholder and customer updates, status page — shields responders from "any update?" pings.
- **Scribe**: timeline of events, decisions, and actions in the incident channel (feeds the postmortem).
- Small team: one person wears several hats but the roles still exist. Hand off explicitly, by name, on shift change.

### Comms cadence

Dedicated channel per incident. Updates on a fixed clock even when nothing changed — SEV1 every 30 min, SEV2 hourly (silence reads as chaos). Template per update: current impact, what we know, what we're doing, next update time. External comms: honest, no speculation about cause, no blame, commit only to the next update time.

### Mitigation before root cause

First question: "what's the fastest way to stop user impact?" — rollback, feature-flag off, failover, shed load, scale up. Diagnose fully *after* the bleeding stops. Preserve evidence while mitigating (snapshot logs/metrics/a broken host if possible). Trap: debugging the root cause live on the incident bridge while a clean rollback sits available — mitigate first; curiosity later.

### Blameless postmortem template (inline)

```markdown
# Postmortem: <incident title> (SEV<N>, YYYY-MM-DD)

- Status: Draft | Reviewed
- Owner: <name>  |  Incident channel: <link>
- User impact: who, what, how long, how many (numbers, not adjectives)

## Summary
3–5 sentences: what happened, why, how it was resolved.

## Timeline (UTC)
- HH:MM detection (how: alert/customer/luck)
- HH:MM declared SEV<N>, IC = <name>
- HH:MM key finding / decision …
- HH:MM mitigated  |  HH:MM resolved

## Root cause
Contributing causes and the mechanism of failure — systems and processes,
never a person's name as a cause. "Engineer X pushed bad config" is banned;
"config had no validation and no canary" is the cause.

## What went well / What went poorly / Where we got lucky

## Action items
| # | Action | Type (prevent/detect/mitigate) | Owner | Due | Ticket |
Each action ticketed with an owner and date, tracked to completion —
a postmortem whose actions die in backlog taught nothing.
```

Blameless = assume everyone acted reasonably on the information they had; ask what made the mistake possible, easy, or invisible. Review postmortems in a regular forum; recurring causes across postmortems are your real architecture backlog.

## How to use this volume

- Query by decision, not by keyword: "choosing a datastore" → Data Modeling table; "PR too big" → Git & Collaboration; "prod is down" → Incident Response (mitigate first).
- Decision tables give defaults — the "choose when / avoid when" columns are the argument; cite constraints, not preferences, when recommending one.
- Copy the inline templates (ADR, error envelope, postmortem) verbatim as starting points and adapt fields to the project.
- "Trap:" lines are the highest-value content when reviewing existing designs — scan them before approving an approach.
- This volume is project-agnostic; project rules and language-specific volumes override it on conflict.
