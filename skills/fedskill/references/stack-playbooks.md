# Stack Playbooks

Per-stack quick playbooks: how to detect the stack, which commands build/test/lint it, the idioms that mark competent code, and the traps that mark incidents waiting to happen. Load this volume for language- or tool-specific questions; for cross-cutting method (testing strategy, API design) use `software-engineering.md`, which this volume specializes. Always prefer the project's own configured commands (package scripts, Makefile, CI definition) over the generic ones here.

## JavaScript / TypeScript + Node.js

| Aspect | Default |
|---|---|
| Detection | `package.json`; lockfile decides the manager: `package-lock.json` → npm, `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lock`/`bun.lockb` → bun |
| Install | `npm ci` (CI/reproducible) / `pnpm install --frozen-lockfile` / `yarn install --immutable` |
| Test | `npx vitest run` or `npx jest` — check `package.json` scripts first |
| Lint / format | `npx eslint .` / `npx prettier --check .` |
| Typecheck | `npx tsc --noEmit` |

- Respect the lockfile's manager; mixing managers corrupts resolution state.
- ESM vs CJS is set by `"type"` in `package.json` and file extension (`.mjs`/`.cjs` override). Match the project; don't convert incidentally.
- Enable `"strict": true` in `tsconfig.json`; new code should never widen to `any` — use `unknown` and narrow.
- Prefer built-ins over micro-dependencies: `fetch`, `node:test`, `structuredClone`, `AbortController` are in modern Node.
- Handle promise rejection everywhere: an unawaited floating promise is a silent failure path; lint rule `no-floating-promises` earns its keep.
- Trap: `npm install` in CI mutates the lockfile — always `npm ci`.
- Trap: `undefined` vs `null` vs missing key behave differently under JSON serialization and optional chaining; pick one convention per codebase.
- Trap: top-level `await` works only in ESM; in CJS it's a syntax error that surfaces as a confusing parse failure.
- Trap: `Array.prototype.sort` is in-place and lexicographic by default — `[10, 9].sort()` yields `[10, 9]`.

## React & frontend

| Aspect | Default |
|---|---|
| Detection | `react` in dependencies; framework by config file: `next.config.*`, `vite.config.*`, `remix.config.*` |
| Dev / build | framework CLI (`next dev/build`, `vite`/`vite build`) |
| Test | `vitest` + Testing Library; e2e via Playwright |
| Lint | `eslint-plugin-react-hooks` is non-negotiable |

- Hooks rules are law: call hooks unconditionally at the top level; exhaustive deps on `useEffect` — a suppressed dep warning is a bug you scheduled.
- Colocate state with its consumer; lift only when two consumers genuinely share it. Global stores are for global state, not for avoiding prop drilling twice.
- Server vs client components (RSC frameworks): default to server; add `"use client"` only where interactivity requires it — the boundary is an architectural decision, not a fix for an error message.
- Derive, don't sync: values computable from props/state during render should be computed there, not mirrored into state via effects.
- Keys must be stable identities, never array indices on reorderable lists.
- Measure bundles before optimizing; dynamic-import routes and heavy widgets first.
- Trap: `useEffect` for data fetching without cancellation races on fast navigation; use the framework's loader or a query library with abort support.
- Trap: a11y regressions ship silently — semantic elements, focus management on route change, and labels are review items, not polish (see the Accessibility checklist in `checklists.md`).

## Python

| Aspect | Default |
|---|---|
| Detection | `pyproject.toml` (modern), `requirements.txt`/`setup.py` (legacy); manager: `uv.lock` → uv, `poetry.lock` → poetry |
| Environment | `uv venv` / `python -m venv .venv` — never install into the system interpreter |
| Test | `pytest` (`uv run pytest`) |
| Lint / format | `ruff check .` and `ruff format .` |
| Typecheck | `mypy` or `pyright`; honor the project's chosen one |

- Type-hint public functions; validate external data at boundaries with `pydantic` or dataclasses + explicit checks — hints alone don't validate.
- Prefer pathlib over os.path, f-strings over `%`/`format`, comprehensions over `map`/`filter` chains when they stay readable.
- Use `with` for anything that opens; context managers are the resource-safety idiom.
- pytest idioms: plain `assert`, fixtures over setUp classes, `parametrize` for case tables.
- Trap: mutable default arguments (`def f(x, acc=[])`) share state across calls — default to `None` and create inside.
- Trap: naming a module after a stdlib/package name (`token.py`, `types.py`) shadows imports with baffling errors.
- Trap: `except Exception: pass` hides the failure you'll debug for a day; catch narrowly, log always.
- Trap: timezone-naive `datetime.now()` in anything persisted or compared — use `datetime.now(timezone.utc)`.

## Go

| Aspect | Default |
|---|---|
| Detection | `go.mod` |
| Build / test | `go build ./...` / `go test ./...` (add `-race` in CI) |
| Lint / format | `gofmt`/`goimports` (non-negotiable), `go vet`, `golangci-lint run` |

- Handle every error at the call site: wrap with `fmt.Errorf("doing X: %w", err)` to keep the chain; check with `errors.Is`/`errors.As`.
- `context.Context` is the first parameter of anything that blocks, calls out, or can be canceled; never store contexts in structs.
- Table-driven tests with subtests (`t.Run`) are the house style everywhere Go is written.
- Accept interfaces, return structs; define interfaces where they're consumed, not where implemented.
- Goroutines need an owner: know how each one ends; `sync.WaitGroup`/`errgroup` over fire-and-forget.
- Trap: loop-variable capture in goroutines (pre-1.22 semantics) — pass the variable as an argument.
- Trap: `nil` map writes panic; `nil` slice appends are fine — the asymmetry bites.
- Trap: an unbuffered channel send with no ready receiver deadlocks silently in tests and hangs in prod.

## Rust

| Aspect | Default |
|---|---|
| Detection | `Cargo.toml` |
| Build / test | `cargo build` / `cargo test` |
| Lint / format | `cargo clippy -- -D warnings`, `cargo fmt --check` |

- Errors: `thiserror` for library error types, `anyhow` for application glue; `?` everywhere, `unwrap`/`expect` only where invariants are documented or in tests.
- Fight the borrow checker with structure, not `clone()` spam: split structs, pass indices/ids, or use interior mutability deliberately.
- Prefer iterators and pattern matching over index loops and flag variables; exhaustive `match` on enums turns future variants into compile errors — that's a feature, keep it by avoiding `_` arms on your own enums.
- `unsafe` requires a `// SAFETY:` comment stating the upheld invariant; reviewers reject bare unsafe.
- Trap: holding a `MutexGuard`/borrow across an `.await` — compile errors at best, deadlocks at worst; scope guards tightly.
- Trap: integer overflow panics in debug and wraps in release — use `checked_*`/`saturating_*` where overflow is reachable.

## Java / Kotlin (JVM)

| Aspect | Default |
|---|---|
| Detection | `build.gradle(.kts)` → Gradle, `pom.xml` → Maven; `.kt` sources → Kotlin |
| Build / test | `./gradlew build` `./gradlew test` / `mvn verify` — use the wrapper, not a global install |
| Lint / format | ktlint/detekt (Kotlin), Checkstyle/SpotBugs or Error Prone (Java) |

- Kotlin null-safety is the point: model absence with `?` types; every `!!` is a defect until proven otherwise.
- Immutability by default: `val`, `List` over `MutableList`, records/data classes for value types.
- Coroutines: structured concurrency only — launch inside a scope that outlives the work; `runBlocking` belongs in main() and tests, never in request paths.
- JUnit 5 + AssertJ/kotest; Testcontainers for real-dependency integration tests.
- Trap: JPA/Hibernate lazy loading outside a transaction throws at render time, far from the cause; fetch explicitly for each use case (and watch N+1 — see the glossary).
- Trap: `equals`/`hashCode` on mutable entities used in sets/maps — identity changes under your feet.

## C# / .NET

| Aspect | Default |
|---|---|
| Detection | `*.csproj` / `*.sln` |
| Build / test | `dotnet build` / `dotnet test` |
| Lint / format | `dotnet format`; analyzers on with `TreatWarningsAsErrors` |

- `async`/`await` all the way down: no `.Result`/`.Wait()` (deadlock classic); suffix async methods with `Async`; `ConfigureAwait(false)` in libraries.
- Enable nullable reference types (`<Nullable>enable</Nullable>`) and fix, don't suppress, the warnings.
- DI via the built-in container; constructor injection; scoped lifetimes for per-request state — resolving scoped from singleton is a runtime trap.
- xUnit + FluentAssertions; `WebApplicationFactory` for in-memory API integration tests.
- Trap: LINQ deferred execution — a query enumerated twice hits the database twice; materialize with `ToList()` deliberately.
- Trap: `DateTime.Now` vs `DateTime.UtcNow` — persist UTC or use `DateTimeOffset`.

## Mobile: Swift/iOS & Kotlin/Android essentials

| Aspect | iOS | Android |
|---|---|---|
| Detection | `*.xcodeproj`/`Package.swift` | `build.gradle` with `com.android.application` |
| Build/test | `xcodebuild test` / `swift test` | `./gradlew assembleDebug` `./gradlew test` |
| UI toolkit default | SwiftUI (UIKit interop where needed) | Jetpack Compose (View interop where needed) |

- Both platforms: keep business logic out of view controllers/activities — lifecycle owners are entry points, not homes; view models + unidirectional data flow.
- Never block the main thread; async work via Swift concurrency (`async/await`, actors) or Kotlin coroutines with lifecycle-aware scopes (`viewModelScope`).
- Trap (iOS): retain cycles via `self` capture in closures — `[weak self]` at escaping boundaries.
- Trap (Android): holding a `Context`/View reference past its lifecycle leaks the Activity; use application context for long-lived objects.
- Trap (both): shipping without testing process death/state restoration — the top source of "works on my phone" crashes.

## SQL & databases (Postgres-first)

| Aspect | Default |
|---|---|
| Detection | `migrations/` dir, ORM config, `docker-compose` service, `DATABASE_URL` |
| Migrations | the project's tool (Flyway, alembic, prisma migrate, golang-migrate…) — versioned files, forward-only mindset |
| Inspect | `EXPLAIN (ANALYZE, BUFFERS)` before and after any index/query change |

- Parameterized queries only — string-built SQL is an injection and a plan-cache miss at once.
- Index for the query shape: leftmost-prefix rule for composite indexes; every foreign key gets an index; partial indexes for hot filtered subsets.
- Keep transactions short; never hold one across a network call or user interaction.
- Use the database's strengths: constraints (`NOT NULL`, `CHECK`, `UNIQUE`, FK) are the last line of data integrity — application-only validation drifts.
- Pool connections; serverless callers need a pooler (e.g., pgbouncer) or they exhaust `max_connections` on the first spike.
- Trap: `SELECT *` in application code couples you to schema width and breaks covering indexes.
- Trap: `NULL` propagates through comparisons — `col != 'x'` excludes NULL rows silently; reach for `IS DISTINCT FROM`.
- Trap: an ORM default of lazy loading in a loop is the canonical N+1; log queries in dev and count them.

## Shell / Bash

| Aspect | Default |
|---|---|
| Detection | `*.sh`, shebang lines, `Makefile` targets |
| Lint | `shellcheck` on every script; `shfmt` for format |

- Open every script with `set -euo pipefail` and understand what each flag does before relying on it.
- Quote every expansion: `"$var"`, `"$@"` — unquoted expansion is the number-one shell defect class.
- `[[ ]]` over `[ ]` in bash; `$(cmd)` over backticks; `printf` over `echo` for anything with escapes or user data.
- Use `trap 'cleanup' EXIT` for temp files and locks; `mktemp` for temp paths.
- Past ~100 lines or any real data structure, move to Python/Node — shell is glue, not an application language.
- Trap: `cmd | while read` runs the loop in a subshell — variables set inside vanish; use process substitution or a for-loop.
- Trap: `set -e` does not fire inside `if cmd; then`, `cmd || true`, or command substitution in some shells — error handling still needs thought.
- Trap: parsing `ls` output breaks on spaces; glob or `find -print0 | xargs -0`.

## Docker & Kubernetes

| Aspect | Default |
|---|---|
| Detection | `Dockerfile`, `compose.yaml`/`docker-compose.yml`, `k8s/`/`helm/` dirs |
| Build | `docker build` with BuildKit; multi-stage always |
| Local stack | `docker compose up` |

- Multi-stage builds: build stage with toolchain, runtime stage minimal (distroless/alpine-class); copy artifacts only.
- Run as non-root (`USER`), pin base images by digest or at least minor version, `.dockerignore` mirrors `.gitignore` plus secrets and `.git`.
- Order Dockerfile layers by change frequency: dependency manifests + install first, source last — that's the cache.
- Kubernetes: every container declares resource requests/limits; liveness probe answers "should this be restarted", readiness answers "should this get traffic" — conflating them causes restart storms.
- Config via env/ConfigMap, secrets via Secret + external manager; never bake either into images.
- Trap: `latest` tags make deploys non-reproducible and rollbacks fictional.
- Trap: PID 1 without signal handling ignores SIGTERM — use exec-form `ENTRYPOINT`, tini, or handle signals; otherwise every stop is a 30s kill.
- Trap: missing memory limits → node-level OOM kills that look like random pod deaths.

## CI systems (GitHub Actions idioms)

| Aspect | Default |
|---|---|
| Detection | `.github/workflows/*.yml`; alternatives: `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/` |
| Local sanity | run the same commands the workflow runs — CI must mirror the local fast-check set |

- Order jobs fail-fast: lint/typecheck before unit before integration/e2e; cheap failures first.
- Cache by lockfile hash (`actions/cache` or setup-* built-in caching); a cold cache should slow CI, never break it.
- Matrix across the versions you actually support — each matrix cell is paid compute, prune it.
- Security is part of the playbook: pin third-party actions to a commit SHA, set least-privilege `permissions:` per workflow, never expose secrets to `pull_request_target`-triggered code from forks, and don't echo secrets (masking is best-effort).
- Make CI required: branch protection on the checks that matter; a non-required check is documentation.
- Trap: `pull_request` vs `push` triggers run against different merge states; test the merge result (`pull_request`) for correctness gates.
- Trap: a workflow that regenerates files (lockfiles, registries) without committing or failing produces permanent drift — check in the regeneration or fail the build on diff.

## Stack detection order

When entering an unknown repository, detect the stack in this precedence order — stop at the first decisive signal per layer, and expect polyglot repos to hit several:

1. **Lockfiles** (decisive for both language and package manager): `package-lock.json`/`pnpm-lock.yaml`/`yarn.lock`/`bun.lock`, `uv.lock`/`poetry.lock`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, `composer.lock`.
2. **Manifests**: `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`/`build.gradle*`, `*.csproj`, `Package.swift`.
3. **Tool/config files**: `tsconfig.json`, `vite.config.*`/`next.config.*`, `Dockerfile`/`compose.yaml`, `.github/workflows/`, migration directories, `Makefile`.
4. **File extensions** (weakest signal — only when nothing above resolves): predominant source extensions under the main source roots.

Then read `package.json` scripts / `Makefile` targets / CI workflow steps to learn the project's *own* command set, which overrides every default table above.

## How to use this volume

- Jump straight to the stack's `##` section; the table gives commands, the bullets give idioms, the "Trap:" lines give review ammunition.
- Run "Stack detection order" first in any unfamiliar repo, and prefer the project's configured scripts over this volume's generic commands.
- When reviewing code, scan the stack's Trap lines against the diff — they are the highest-hit-rate checks here.
- For method-level questions (how much to test, how to design the API), escalate to `software-engineering.md`; this volume covers the how-in-this-stack layer.
