# The FedSkill Glossary

An A-to-Z dictionary of modern software and AI engineering: architecture, web/API, data, infra, security, testing, AI/ML, agentic systems (2026-2027 vocabulary), and process. Load this volume when you need a precise definition, a disambiguation between two commonly confused terms, or the known trap attached to a practice before acting on it. Entries are terse and actionable; scan by letter section, then by bolded term.

## A

**A/B testing** — Serving two variants to disjoint user segments and comparing a pre-registered metric to pick a winner. Trap: peeking at results early and stopping inflates false positives.

**ACID** — Atomicity, Consistency, Isolation, Durability: the transaction guarantees of classical relational databases. vs BASE, which trades consistency for availability.

**ADR (Architecture Decision Record)** — Short versioned document capturing one architectural decision: context, options considered, decision, consequences. Store it in-repo next to the code it governs; never rewrite old ADRs — supersede them.

**adversarial verification** — Having a second model or agent actively try to break, disprove, or find counterexamples to another agent's output before it is accepted. Stronger than a cooperative review pass because the verifier's goal is failure, not approval.

**agent** — An LLM run in a loop with tools, context, and a goal, taking actions against an environment until a stop condition is met. vs chatbot: an agent acts (edits files, calls APIs); a chatbot only replies.

**agent EDR** — Endpoint-detection-and-response applied to AI agents: continuously monitor an agent's tool calls, file and network access, and action sequences; alert on or kill anomalous behavior. Emerging control as agents gain production credentials.

**agentic loop** — The core cycle observe → plan → act (tool call) → observe result, repeated until goal or stop condition. What enters the context window each iteration dominates agent quality more than the base model does.

**aggregate (DDD)** — A cluster of domain objects treated as one consistency unit with a single root entity; all external references and mutations go through the root. Trap: transactions spanning multiple aggregates signal a wrong boundary.

**agile** — Iterative delivery in small increments with continuous feedback, prioritized over big up-front planning. Trap: ceremonies (standups, story points) without short feedback loops are process theater, not agility.

**ANN (approximate nearest neighbor)** — Vector search that trades exactness for sub-linear lookup time (HNSW, IVF, PQ). The retrieval backbone of vector databases; tune recall vs latency explicitly.

**anti-corruption layer (DDD)** — Translation layer between your bounded context and a legacy or external model, so foreign concepts never leak into your domain model.

**API gateway** — Single entry point that routes, authenticates, rate-limits, and shapes traffic to backend services. vs load balancer: a gateway operates on application semantics, not just connections.

**API versioning** — Strategy for evolving an API without breaking clients: URL version (`/v2/`), header, or additive-only changes. Prefer additive evolution; version only on breaking change.

**artifact (build)** — Immutable output of a build (binary, container image, bundle) promoted unchanged through environments. Rebuilding per environment breaks provenance; build once, deploy many.

**assertion quality** — Whether tests verify meaningful behavior rather than merely executing lines. High coverage with weak assertions is false confidence; mutation testing measures assertion quality directly.

**attention** — Transformer mechanism where each token computes weighted relevance over other tokens to build its representation. Its cost grows with sequence length, which is why context windows are bounded and long-context quality degrades.

**autonomous loop** — An agent run without per-step human approval. Must be paired with stop conditions, cost ceilings, sandboxing, and guardrails; never grant production credentials to an unbounded loop.

**autoscaling** — Automatically adjusting replica count or instance size from load signals. Trap: scaling on CPU alone misses queue-depth and latency saturation.

## B

**backpressure** — Signaling upstream producers to slow down when a consumer cannot keep up (bounded queues, HTTP 429, reactive streams). Without it, overload becomes unbounded memory growth and cascading failure.

**BASE** — Basically Available, Soft state, Eventually consistent: the availability-first counterpart to ACID, typical of distributed NoSQL stores.

**batch vs stream processing** — Batch processes bounded datasets on a schedule; streaming processes unbounded events continuously with windowing. Choose by freshness requirement, not fashion.

**BDD (behavior-driven development)** — TDD variant phrasing tests as business-readable behavior specs (Given/When/Then). Value is the shared language with non-engineers; without that audience it is TDD with extra syntax.

**BFF (backend-for-frontend)** — A dedicated backend per client type (web, mobile) that aggregates and shapes downstream APIs for that client's exact needs.

**blameless culture** — Incident analysis that treats failures as systemic (process, tooling, incentives), not individual fault. Precondition for honest postmortems; naming-and-shaming guarantees hidden failures next time.

**blue-green deployment** — Two identical environments; deploy to the idle one, switch traffic atomically, keep the old one warm for instant rollback. vs canary: blue-green switches all traffic at once.

**bounded context (DDD)** — Explicit boundary within which one domain model and one ubiquitous language apply. Different contexts may model "customer" differently; translate at the boundary instead of forcing one global model.

**branch protection** — Repository rules requiring reviews, passing checks, or signed commits before merge to a protected branch. First line of defense for supply-chain integrity.

**bulkhead** — Isolating resource pools per dependency (thread pools, connection pools, pods) so one failing dependency cannot exhaust shared capacity.

**bus factor** — The number of people whose sudden loss would stall the project. Raise it with docs, pairing, and rotation; a bus factor of 1 is an outage waiting for a vacation.

## C

**cache invalidation** — Removing or refreshing stale cached data when the source changes. Trap: prefer short TTLs plus explicit invalidation events; guessing lifetimes causes correctness bugs that look intermittent.

**canary release** — Routing a small percentage of real traffic to a new version, watching SLIs, then ramping or rolling back. vs blue-green: canary is gradual and metric-gated.

**CAP theorem** — Under a network partition, a distributed system must sacrifice either consistency or availability. Trap: CAP applies only during partitions; normal-operation tradeoffs are latency vs consistency (see PACELC).

**cardinality (metrics)** — The number of unique label/tag combinations on a metric. High-cardinality labels (user ID, request ID) explode time-series storage cost; put those in traces or logs instead.

**CDC (change data capture)** — Streaming a database's row-level changes (usually from its write-ahead log) to downstream consumers. The standard way to sync OLTP data into warehouses and caches without dual writes.

**CDN (content delivery network)** — Geographically distributed edge caches serving static (and increasingly dynamic) content close to users. Cache-control headers are the contract; misconfigured ones serve stale or private data.

**chain-of-thought** — Prompting or training an LLM to produce intermediate reasoning steps before the answer, improving multi-step accuracy. Trap: emitted reasoning is not a faithful trace of computation; do not treat it as an audit log.

**changelog** — Human-readable, versioned record of notable changes per release. Generate the skeleton from conventional commits, then edit for the reader.

**chaos engineering** — Deliberately injecting failures (killed instances, latency, partitions) in controlled experiments to verify resilience assumptions before real incidents test them.

**CI/CD** — Continuous integration (merge and test frequently against main) and continuous delivery/deployment (every green build is releasable/released). The pipeline is production code: version it, review it, test it.

**CI agent** — An AI agent invoked inside a CI pipeline — fixing failing builds, reviewing diffs, updating snapshots — running headless with repo-scoped, short-lived credentials. Constrain writes to branches and PRs, never direct pushes to main.

**circuit breaker** — Wrapper that trips after repeated downstream failures and fails fast (or serves fallback) instead of piling up blocked calls, probing periodically for recovery. Prevents retry storms from finishing off a struggling dependency.

**cohesion** — The degree to which a module's contents belong together for one purpose. Maximize cohesion within modules while minimizing coupling between them.

**cold start** — Latency spike when a serverless function or scaled-to-zero service must initialize before serving. Mitigate with provisioned concurrency, lighter runtimes, or keeping hot paths off serverless.

**compaction (context)** — Summarizing or pruning older conversation/tool history to free context-window space while preserving task-relevant facts. Trap: naive summarization drops exact identifiers (paths, IDs, flags) the agent later needs — preserve them verbatim.

**computer use** — An agent operating a real GUI through screenshots plus mouse/keyboard actions rather than APIs. Slower and less reliable than API/tool access; use only when no programmatic interface exists.

**confused deputy** — A privileged component tricked into using its authority on behalf of a less-privileged requester. The canonical frame for agent security: the agent is the deputy, injected content is the trickster; scope credentials per task.

**container** — OS-level virtualized process with isolated filesystem, network, and resource limits, sharing the host kernel. vs VM: containers isolate processes, VMs isolate whole kernels.

**container image** — Immutable, layered filesystem snapshot a container runs from, identified by digest. Pin by digest for reproducibility; mutable tags like `latest` are a supply-chain risk.

**context engineering** — Deliberately curating everything an LLM sees per step — instructions, retrieved docs, tool results, history — as the primary lever on agent behavior. Successor discipline to "prompt engineering": manage the whole window, not one prompt.

**context rot** — Degradation of agent behavior as the context window fills with stale, contradictory, or irrelevant accumulated content. Countered by compaction, scoped subagents, and re-grounding from source files instead of trusting old turns.

**context window** — Maximum tokens a model can attend to in one request (input plus output). Effective usable quality often degrades before the hard limit; budget context like memory, not like disk.

**contract testing** — Verifying that a service's real responses satisfy the expectations consumers recorded (e.g. Pact), so integrations break in CI instead of production. Cheaper than full end-to-end environment tests for inter-service compatibility.

**conventional commits** — Commit message convention (`feat:`, `fix:`, `docs:`, with optional scope and `!` for breaking) enabling automated changelogs and semver bumps.

**Conway's law** — Systems mirror the communication structure of the organizations that build them. Corollary ("inverse Conway maneuver"): reshape teams to get the architecture you want.

**CORS (cross-origin resource sharing)** — Browser mechanism where servers opt specific foreign origins into reading responses. Trap: CORS protects browser users, not your API — it is not authentication or a server-side control.

**cost ceiling** — Hard budget cap (tokens, dollars, API calls) on an agent run, enforced by the harness. Pair every autonomous loop with one; model-side self-restraint is not enforcement.

**coupling** — The degree to which modules depend on each other's internals. Reduce it via stable interfaces and events; tight coupling turns every change into a multi-module change.

**coverage** — Fraction of code executed by tests (line/branch). A floor metric, not a goal: 100% coverage with weak assertions proves nothing — see assertion quality and mutation testing.

**CQRS (command query responsibility segregation)** — Separate write model (commands) from read model (queries), letting each be shaped and scaled independently. Trap: adds eventual-consistency complexity; do not adopt without a read/write asymmetry that demands it.

**CRDT (conflict-free replicated data type)** — Data structure whose concurrent replicas merge deterministically without coordination, powering offline-first and collaborative editing.

**CSP (Content Security Policy)** — HTTP header whitelisting sources for scripts, styles, and connections, mitigating XSS impact. `unsafe-inline` for scripts largely defeats it.

**CSRF (cross-site request forgery)** — Tricking a logged-in user's browser into sending an authenticated state-changing request. Mitigate with SameSite cookies plus anti-CSRF tokens; token-in-header auth schemes are inherently resistant.

**CVE** — Public identifier (CVE-YYYY-NNNNN) for a specific disclosed vulnerability, the shared key across scanners, advisories, and patches.

**CVSS** — 0-10 severity score for a vulnerability from exploitability and impact metrics. Trap: CVSS is severity in the abstract, not your risk — reachability and exposure in your deployment decide priority.

## D

**DAG (directed acyclic graph)** — Graph with directed edges and no cycles; the shape of build systems, data pipelines, git history, and task schedulers. Acyclicity is what makes topological execution order possible.

**data contract** — Explicit, versioned schema-plus-semantics agreement between a data producer and consumers, enforced in CI/pipelines so upstream changes fail fast instead of silently corrupting downstream tables.

**data exfiltration** — Unauthorized movement of data out of a trust boundary. In agent systems the classic vector is injected instructions making the agent embed secrets in URLs, markdown images, or tool arguments pointing at attacker infrastructure.

**data lake** — Central store of raw, schema-on-read data in open file formats. Cheap to land data, easy to turn into a swamp without cataloging and ownership.

**data lakehouse** — Lake storage plus table formats (Iceberg, Delta, Hudi) adding ACID transactions, schema evolution, and time travel — warehouse semantics over lake economics.

**data warehouse** — Structured, schema-on-write analytical store optimized for large scans and joins (columnar storage). The OLAP counterpart to your OLTP database.

**DDD (domain-driven design)** — Designing software around the business domain using a shared ubiquitous language, bounded contexts, and aggregates. The strategic patterns (contexts, boundaries) matter more than the tactical ones (entities, repositories).

**dead-letter queue** — Queue where messages land after exhausting processing retries, preserving them for inspection and replay instead of loss or infinite retry loops. Monitor it; a silent DLQ is data loss on a delay.

**defense in depth** — Layered independent controls so one bypassed control does not equal compromise. For agents: input filtering + least-privilege tools + sandboxing + output checks + monitoring, not any single guardrail.

**denormalization** — Deliberately duplicating data to eliminate joins on hot read paths, accepting update-anomaly risk in exchange for read speed. Document every duplication and who owns keeping it consistent.

**dependency injection** — Passing a component's dependencies in from outside instead of constructing them internally, enabling substitution (test fakes, alternate implementations) and making the dependency graph explicit.

**distillation** — Training a smaller "student" model to reproduce a larger "teacher" model's outputs, trading some quality for much cheaper inference.

**DoS / DDoS** — Overwhelming a service with traffic to deny it to legitimate users; distributed DoS uses many sources. Mitigate at the edge (CDN, WAF, rate limits) — origin servers cannot absorb it alone.

**drift (infrastructure)** — Divergence between declared IaC state and actual deployed state, usually from manual console changes. Detect continuously; drift makes your next apply a surprise.

**DRY vs WET** — DRY ("don't repeat yourself") abstracts duplicated knowledge; WET ("write everything twice") tolerates duplication until a real shared abstraction is proven. Trap: the wrong abstraction costs more than duplication — DRY concepts, not coincidentally similar lines.

## E

**edge computing** — Executing compute at CDN points of presence close to users (edge functions/workers), cutting latency for personalization, auth, and routing. Constraints: limited runtime, cold-start sensitivity, and distance from your primary database.

**embedding** — Dense vector representation of text/images/code where semantic similarity maps to vector proximity. Foundation of semantic search and RAG retrieval. Trap: embeddings from different models or versions are not comparable — re-embed on model change.

**encryption in transit / at rest** — TLS for data moving between systems; disk/object/database encryption for stored data. Table stakes; key management and rotation are where real designs differ.

**end-to-end (E2E) test** — Test exercising the full deployed stack through real interfaces (browser, API). Highest confidence per test, highest cost and flake rate — keep few, cover critical journeys only.

**ephemeral environment** — Short-lived full environment spun up per PR/branch for review and testing, destroyed after merge. Kills the shared-staging bottleneck.

**error budget** — The allowed unreliability implied by an SLO (100% minus target). Budget remaining gates release velocity: budget left → ship; budget burned → stop features, fix reliability.

**ETag / conditional request** — Response validator token; clients send `If-None-Match` to get 304 Not Modified (cache validation) or `If-Match` on writes to prevent lost updates (optimistic concurrency over HTTP).

**ETL vs ELT** — Transform before loading (ETL) vs load raw then transform in the warehouse (ELT). ELT dominates modern stacks because warehouse compute is cheap and raw data is reprocessable.

**eval** — A measurable test of model or agent behavior against expected outcomes; the unit test of AI systems. No prompt, model, or tool change ships without evals to catch regressions.

**eval harness** — Infrastructure that runs eval suites: dataset management, environment setup, scoring (exact match, rubric, LLM-as-judge), and run-to-run comparison. Trap: agent evals need multiple runs per case — single-run scores are noise.

**event sourcing** — Persisting state as an append-only sequence of events, deriving current state by replay; gives a complete audit trail and temporal queries. Trap: event schema evolution and migrations are the real long-term cost.

**event-driven architecture** — Services communicating via asynchronous events on a broker rather than synchronous calls, decoupling producers from consumers. Debugging shifts from stack traces to tracing event flows — invest in correlation IDs.

**eventual consistency** — Guarantee that replicas converge given no new writes, with a window where reads may be stale. Design UIs and workflows to tolerate the window (read-your-writes tricks, versioning) instead of pretending it away.

## F

**failover** — Automatic promotion of a standby when the primary fails. Test it deliberately; an unexercised failover path is folklore, not resilience.

**fake** — Working lightweight implementation of a dependency (e.g. in-memory repository) used in tests. vs mock: a fake has real behavior; a mock records and verifies interactions.

**fan-out** — One event or request triggering work to many downstream consumers. Watch amplification: a cheap upstream write can become an expensive downstream storm.

**feature flag** — Runtime toggle decoupling deploy from release: ship dark, enable per segment, kill instantly. Trap: stale flags are tech debt with combinatorial test cost — expire them on a schedule.

**few-shot prompting** — Including worked examples in the prompt so the model infers the pattern. vs zero-shot: instructions only. Examples beat adjectives; three good examples outperform paragraphs of description.

**fine-tuning** — Continuing training of a pretrained model on domain data to shift behavior/style/format. Reach for RAG or prompting first: fine-tuning is for behavior you cannot specify in context, not for injecting facts.

**fixture** — Prepared data or environment state a test depends on, ideally built per-test via factories. Shared mutable fixtures are a top source of order-dependent flaky tests.

**flaky test** — Test that passes and fails without code change (timing, ordering, shared state, real network). Quarantine and fix immediately — tolerated flakes train people to ignore CI, which converts real failures into ignored failures.

**function calling** — Model emitting a structured invocation (name + JSON arguments) of a declared function instead of prose; the mechanism beneath tool use. Validate arguments against the schema before executing — models produce plausible-but-invalid calls.

## G

**GitOps** — Operating infrastructure and deployments from git as the single source of truth, with an in-cluster agent (Argo CD, Flux) continuously reconciling actual state to the declared state. Rollback = `git revert`.

**golden file** — Committed expected-output file that tests compare actual output against, with an explicit regeneration command. vs snapshot testing: same idea, but golden files are hand-reviewed artifacts, not auto-captured blobs.

**graceful degradation** — Continuing to serve reduced functionality when a dependency fails (cached results, defaults, disabled feature) instead of failing the whole request.

**GraphQL** — Query language where clients specify exactly the fields they need against a typed schema, over a single endpoint. Traps: unbounded query depth/complexity needs server-side limits, and naive resolvers breed N+1 queries — use dataloaders.

**grounding** — Anchoring model output to verifiable sources (retrieved documents, tool results, citations) rather than parametric memory. The primary countermeasure to hallucination; require citations and verify them.

**gRPC** — High-performance RPC framework using protobuf schemas over HTTP/2 with streaming and codegen'd typed clients. Standard for service-to-service; browsers need a gateway (gRPC-Web/transcoding).

**guardrail** — Automated check constraining agent/model inputs or outputs: schema validation, content filters, policy engines, action allowlists. Guardrails are enforcement outside the model; instructions inside the prompt are requests, not guarantees.

## H

**hallucination** — Model output that is fluent, confident, and false — fabricated APIs, citations, facts. Mitigate with grounding, structured verification against sources, and making "I don't know" an acceptable output.

**harness** — The non-model machinery an agent runs inside: loop control, tool dispatch, context assembly, permissions, budgets, checkpointing. Most agent failures are harness failures, not model failures — debug the loop before blaming the weights.

**HATEOAS** — REST constraint where responses embed hypermedia links to available next actions, letting clients navigate by link relations instead of hardcoded URLs. Rarely fully implemented; useful selectively for workflow APIs.

**headless agent** — Agent running without interactive UI or human present — cron jobs, CI, queue consumers. Since nobody will answer clarifying questions, it needs complete standalone instructions, strict stop conditions, and result reporting out-of-band.

**health check** — Endpoint reporting service status: liveness (restart me if failing) vs readiness (don't route traffic yet). Trap: a liveness check that depends on a downstream service turns that dependency's outage into a restart storm.

**hexagonal architecture (ports and adapters)** — Domain core exposing ports (interfaces), with adapters translating to the outside world (HTTP, DB, queues) at the edges. Dependencies point inward; the core imports no framework.

**HNSW (hierarchical navigable small world)** — Graph-based ANN index with layered skip-list-like structure; the default high-recall, low-latency index in most vector databases. Memory-hungry; parameters (M, efSearch) trade recall vs speed.

**hook** — Deterministic automation triggered by a lifecycle event (pre/post tool call, session start, pre-commit) that runs code outside the model's discretion. Use hooks for anything that must always happen — models forget, hooks don't.

**human-in-the-loop** — Requiring human review or approval at designated points in an automated flow — typically before irreversible or high-privilege actions. Design the checkpoint list explicitly; "human somewhere nearby" is not a control.

## I

**IaC (infrastructure as code)** — Declaring infrastructure in versioned code (Terraform, Pulumi, CloudFormation) applied through a plan/review/apply pipeline. Manual console changes create drift; treat them as incidents.

**idempotency** — Property that applying an operation multiple times equals applying it once. Essential wherever retries exist (queues, webhooks, payments); implement via idempotency keys and upserts. Any at-least-once delivery system without idempotent consumers has a duplication bug by design.

**IdP (identity provider)** — Service that authenticates users and issues assertions/tokens to relying applications (SSO via OIDC/SAML). Centralizes credentials, MFA, and offboarding.

**immutable infrastructure** — Never patching running servers; every change ships as a new image/instance replacing the old. Eliminates config drift and snowflake servers.

**index (database)** — Auxiliary structure for fast lookups: B-tree (range/equality, the default), hash (equality), GIN/inverted (arrays, full-text), partial (subset predicate), covering (includes queried columns, skips table fetch). Every index taxes writes — index for measured query patterns, then verify with EXPLAIN.

**indirect prompt injection** — Malicious instructions planted in content the agent will process — web pages, emails, file contents, tool results — rather than typed by the user. The top practical attack on tool-using agents: treat all fetched content as data, never as directives.

**inference** — Running a trained model to produce output (vs training). Production concerns: batching, KV-cache reuse, quantization, and the latency/throughput tradeoff.

**integration test** — Test verifying multiple components work together across real boundaries (service + real database via testcontainers, module + module). Sits between unit and E2E in cost and confidence.

**isolation levels** — Transaction visibility guarantees: read uncommitted → read committed → repeatable read → serializable, trading anomalies (dirty/non-repeatable/phantom reads, write skew) for concurrency. Trap: most databases default to read committed — code assuming serializable behavior has latent race conditions.

## J

**jailbreak** — Crafted input designed to make a model bypass its safety or policy constraints (role-play framing, encoding tricks, many-shot pressure). vs prompt injection: jailbreak attacks the model's own rules; injection hijacks the application's instructions.

**JSON Schema** — Standard for declaring JSON structure and constraints; the lingua franca for tool/function parameter definitions and structured-output contracts. Validate at runtime — a schema nobody enforces is documentation.

**judge panel** — Multiple LLM judges (different models or prompts) scoring an output independently, aggregated by vote or consensus, reducing single-judge bias in evals and adversarial verification.

**JWT (JSON Web Token)** — Signed (optionally encrypted) token carrying claims, verifiable statelessly by any holder of the key. Traps: cannot be revoked before expiry without a denylist — keep TTLs short; never accept `alg: none`; and a JWT is authorization data, not encryption — payloads are readable.

## K

**kanban** — Flow-based process: visualize work stages, limit work-in-progress, pull when capacity frees. WIP limits are the mechanism — a kanban board without them is just a to-do list.

**key rotation** — Regularly replacing cryptographic keys and credentials with overlapping validity so compromise windows stay short. Automate it; manual rotation means never.

**KISS** — "Keep it simple": prefer the simplest design meeting current requirements. Complexity must be paid for by demonstrated need, not anticipated need.

**Kubernetes** — Container orchestration platform reconciling declared desired state to actual state: scheduling, scaling, self-healing, service discovery, rolling updates.

**Kubernetes primitives** — Pod: smallest deployable unit, one or more co-located containers. Deployment: manages replicated pods with rolling updates. Service: stable virtual IP/DNS over ephemeral pods. Ingress: HTTP routing from outside. ConfigMap/Secret: injected configuration (Secrets are base64, not encrypted by default — enable encryption at rest). Namespace: soft multi-tenancy boundary.

## L

**latency percentiles (p50/p95/p99)** — Distribution-based latency measures; p99 is what your heaviest users feel and what times out. Trap: never average latencies — means hide the tail that hurts.

**latency vs throughput** — Latency: time for one request; throughput: requests per unit time. Optimizations trade one for the other (batching raises throughput and latency); state which you are optimizing before touching code. In LLM serving: time-to-first-token vs tokens/sec.

**least privilege** — Granting the minimum access needed for the task, no more, scoped in time where possible. For agents: per-task tokens over broad standing credentials — assume the agent can be tricked and size the blast radius accordingly.

**LLM-as-judge** — Using a model to score outputs against a rubric, enabling scalable evaluation of open-ended tasks. Calibrate against human labels; known biases include position preference, verbosity preference, and self-preference for its own model family.

**load balancer** — Distributes traffic across healthy instances (round-robin, least-connections) at L4 (connections) or L7 (HTTP-aware). Health-check integration is what turns it from a splitter into a failover mechanism.

**logs** — Timestamped discrete event records; one of the three observability pillars. Emit structured (JSON) logs with correlation/trace IDs, not printf prose — logs you cannot query are logs you do not have.

**LoRA (low-rank adaptation)** — Fine-tuning method training small low-rank matrices alongside frozen base weights, cutting cost and memory by orders of magnitude; adapters are swappable per task at serve time.

## M

**MCP (Model Context Protocol)** — Open protocol standardizing how AI applications connect to external tools, data, and prompts; clients (agents/IDEs) discover and invoke capabilities from MCP servers uniformly instead of via bespoke integrations.

**MCP connector** — A packaged, user-facing MCP integration to a specific service (mail, drive, database) attached to an AI client, bundling the server, auth flow, and tool surface.

**MCP server** — Process exposing tools, resources, and prompts over MCP. Treat third-party servers as supply-chain dependencies: pin versions, review tool descriptions (see tool poisoning), and scope their credentials — a malicious server sits inside your agent's trust boundary.

**memory (agent)** — Persistence of information across agent sessions, beyond the context window. Retrieval quality, not storage, is the hard part — memory that surfaces the wrong fact is worse than none.

**memory, episodic** — Records of specific past events and sessions ("what happened when"): past runs, decisions, outcomes. Enables learning from precedent.

**memory, procedural** — Stored know-how: learned workflows, rules, skills — "how to do X here". In Claude Code terms: CLAUDE.md, rules, and skills are procedural memory.

**memory, semantic** — Stored facts and knowledge independent of when learned ("what is true"): user preferences, domain facts, architecture decisions.

**message queue** — Broker buffering messages between producers and consumers, decoupling availability and absorbing bursts. Know your delivery guarantee: at-least-once demands idempotent consumers; exactly-once is usually at-least-once plus deduplication.

**metrics** — Numeric time series (counters, gauges, histograms) cheap to store and alert on; one of the three observability pillars. Watch label cardinality; alert on symptoms (SLIs), not causes.

**microservices** — Independently deployable services, each owning its data, communicating over the network. Buys independent scaling and team autonomy at the cost of distributed-systems complexity everywhere. Trap: a "distributed monolith" — services that must deploy together — has the costs of both and benefits of neither.

**MLOps** — Operational discipline for ML systems: data/model versioning, reproducible training, deployment, monitoring for drift, and retraining loops. The model file is the small part; the pipeline around it is the product.

**mock** — Test double that records calls and verifies expected interactions (arguments, counts, order). Trap: over-mocking couples tests to implementation; mock your own boundaries, not third-party internals, and prefer fakes for complex behavior.

**modular monolith** — Single deployable with strictly enforced internal module boundaries (separate schemas, no cross-module imports except via interfaces). The pragmatic default: monolith operability with an extraction path if a module later needs independence.

**monolith** — Single deployable containing all functionality. Simple to operate, test, and refactor; scales further than fashion admits. The problem is unstructured internals, not the single deployable.

**monorepo** — All projects in one repository: atomic cross-project changes, one toolchain, shared visibility. Needs investment in selective CI (build only what changed) as it grows.

**mTLS (mutual TLS)** — Both sides authenticate with certificates, giving every service a cryptographic identity. Foundation of zero-trust service-to-service traffic; typically automated by a service mesh.

**multi-agent orchestration** — Coordinating multiple agents on one goal — parallel fan-out, pipelines, planner-executor, judge panels. Costs context-transfer overhead and compounding errors: use for parallelizable or role-separated work, not as a default.

**mutation testing** — Automatically introducing small code mutations and checking tests fail; surviving mutants expose untested behavior. The honest measure of test-suite strength, unlike coverage.

**MVP (minimum viable product)** — Smallest release that tests the core value hypothesis with real users. It is an experiment, not phase one of the full spec — be prepared to discard it.

## N

**N+1 query** — Fetching a list, then issuing one query per item (1 + N round trips). Fix with joins, batched `IN` queries, or dataloaders. The most common ORM-induced performance bug; it hides until data grows.

**normalization** — Structuring relational data to eliminate redundancy so every fact lives in one place (3NF as the practical default). Normalize for write correctness; denormalize deliberately and locally for read speed.

**NoSQL** — Non-relational databases trading relational guarantees for specific shapes: document (MongoDB), key-value (Redis), wide-column (Cassandra), graph (Neo4j). Chosen by access pattern; most CRUD apps still belong on PostgreSQL.

## O

**OAuth 2.0** — Delegated authorization framework: a user grants a client scoped access to a resource server via tokens from an authorization server, without sharing credentials. Trap: OAuth is authorization, not authentication — that is OIDC's job. Use authorization-code flow with PKCE for public clients.

**observability** — Ability to explain a system's internal state from its outputs (logs, metrics, traces) well enough to debug conditions you did not predict. vs monitoring: monitoring watches known failure modes; observability handles unknown ones.

**OIDC (OpenID Connect)** — Authentication layer on OAuth 2.0 adding an ID token (JWT) with verified identity claims; the standard behind "Sign in with X" and workload identity federation in CI.

**OLAP** — Analytical processing: large scans and aggregations over history, columnar storage, batch/streaming ingest. Never point dashboards at your OLTP database; replicate via CDC.

**OLTP** — Transactional processing: many small concurrent reads/writes, row-oriented storage, indexes on hot paths, ACID. Your application database.

**OpenAPI** — Machine-readable REST API contract (schemas, endpoints, auth) driving codegen, validation, and docs. Adopt design-first or generate from code, but the spec must be CI-verified against reality — a stale spec is worse than none.

**OpenTelemetry** — Vendor-neutral standard APIs, SDKs, and wire protocol (OTLP) for traces, metrics, and logs. Instrument once against OTel, export anywhere; the default choice — avoid vendor-proprietary instrumentation.

**optimistic locking** — Detecting write conflicts at commit via a version column and retrying, instead of holding locks. vs pessimistic locking (lock first): optimistic wins under low contention; pessimistic under high contention or costly retries.

**orchestration** — Automated management of workload lifecycle across machines: placement, scaling, restarts, rollouts (Kubernetes for containers; Airflow/Temporal for workflows; harnesses for agents). Common thread: declare desired state, reconcile continuously.

**outbox pattern** — Writing domain change and outgoing event in one local transaction (event to an "outbox" table), with a relay publishing to the broker afterward. Solves the dual-write problem — never write DB and publish to a broker as two separate steps.

**OWASP Top 10** — Periodically updated list of the most critical web application security risks (broken access control, injection, misconfiguration, SSRF, etc.); the baseline checklist for web security review. A companion OWASP Top 10 for LLM applications covers prompt injection, insecure output handling, and excessive agency.

## P

**pagination** — Splitting large result sets across requests. Offset/limit is simple but drifts under concurrent writes and degrades at depth; cursor/keyset (opaque token from last item's sort key) is stable and O(1) — default to cursor for APIs.

**planner-executor** — Multi-agent pattern: a planner decomposes the goal into tasks; executors (often cheaper/scoped agents) run them; the planner reviews and re-plans. Keeps the big context in one place while parallelizing the work.

**polyrepo** — One repository per project/service: independent versioning and access control, at the cost of painful cross-repo changes and dependency version skew. vs monorepo: choose by how often changes cross project boundaries.

**postmortem** — Structured incident write-up: timeline, impact, contributing causes, and tracked action items. Blameless by policy; an unwritten or action-item-free postmortem means the incident will repeat.

**PRD (product requirements document)** — Document defining what to build and why: problem, users, goals, success metrics, scope and non-goals. The non-goals section prevents more waste than the goals section.

**progressive disclosure** — Structuring information in layers loaded on demand — summary first, detail on request. In agent skills: a short SKILL.md that points to reference volumes like this one, so context is spent only when needed.

**prompt caching** — Server-side reuse of computation for a repeated prompt prefix, cutting cost and latency substantially. Structure prompts with stable content (system, tools, references) first and variable content last; a changed early byte invalidates the cache.

**prompt injection** — Adversarial input that overrides an application's instructions to the model ("ignore previous instructions and..."). No reliable model-level fix exists: rely on privilege separation, output validation, and human gates on dangerous actions. See indirect prompt injection for the fetched-content variant.

**property-based testing** — Asserting invariants over generated inputs ("decode(encode(x)) == x") with automatic shrinking of failing cases to minimal counterexamples. Finds edge cases example-based tests never enumerate; seed found failures back as regression examples.

**pub/sub** — Messaging where publishers emit to topics and any number of subscribers receive independently, decoupling producers from consumer count and identity. vs queue: a queue delivers each message to one consumer; pub/sub to all subscribers.

## Q

**quantization** — Reducing model weight precision (FP16 → INT8/INT4) to shrink memory and speed inference with modest quality loss. Evaluate quality on your task after quantizing — degradation is task-dependent and uneven.

**quorum** — Minimum replica votes for a distributed operation (typically majority: `N/2+1`). With read and write quorums overlapping (R + W > N), reads see the latest write; the mechanism beneath consensus systems like Raft.

## R

**RAG (retrieval-augmented generation)** — Retrieving relevant documents (vector/keyword/hybrid search) and placing them in context so the model answers from evidence rather than parametric memory. Retrieval quality bounds answer quality — evaluate retrieval separately from generation.

**rate limiting** — Capping request rates per client/key/IP (token bucket, sliding window), returning 429 with `Retry-After`. Protects capacity and blunts abuse; apply at the edge and per-tenant, not just globally.

**red teaming** — Authorized adversarial exercise attacking a system as a real adversary would — for AI systems: jailbreaks, injection, exfiltration attempts, tool abuse — to find failures before attackers do. Turn every successful attack into a regression eval.

**release train** — Fixed-schedule releases that ship whatever is ready ("the train leaves on time"); missed features catch the next train. Removes deadline-driven merge pressure and makes releasing routine.

**replication** — Maintaining data copies across nodes: leader-follower (single writer, read replicas — mind replication lag), multi-leader (conflicts to resolve), leaderless/quorum. Trap: reading your own write from a lagging replica looks like data loss to users — route read-after-write to the leader.

**REST** — API style using HTTP semantics over resources: nouns in URLs, verbs from methods (GET/POST/PUT/PATCH/DELETE), status codes, statelessness, cacheability. Trap: RPC-style endpoints (`/getUser`, `/doThing`) with JSON are not REST — which is fine, but then apply RPC conventions consistently.

**retry with backoff** — Retrying failed calls with exponentially growing, jittered delays and a retry cap. Jitter prevents synchronized retry storms; only retry idempotent operations, and respect circuit breakers.

**RFC (request for comments)** — Written design proposal circulated for structured feedback before significant work begins. Scales design review beyond meeting size; the written record becomes the "why" future maintainers need.

**RLHF (reinforcement learning from human feedback)** — Aligning a model by training a reward model on human preference comparisons, then optimizing the LLM against it. The step that turns raw next-token predictors into instruction-following assistants.

**rollback** — Reverting to the last known-good version on failure. Design deploys to be roll-back-able: backward-compatible schema migrations (expand-migrate-contract), versioned artifacts, and feature flags as the instant lever.

**rug-pull (MCP)** — Supply-chain attack where an MCP server behaves benignly through review/adoption, then a later update swaps in malicious tool definitions or behavior. Pin server versions, diff tool schemas on update, and re-review before upgrading.

**rule** — Always-on instruction file loaded into an agent's context governing behavior (security constraints, code style, workflow requirements). vs skill: rules always apply; skills load on demand for specific tasks.

**runbook** — Step-by-step operational procedure for a specific scenario (alert response, failover, rotation), executable by someone without full system context — increasingly, by an agent. Test runbooks in game days; stale runbooks fail during real incidents.

## S

**saga** — Distributed transaction as a sequence of local transactions, each with a compensating action to semantically undo it on later failure. Choreographed (events) or orchestrated (coordinator). Trap: compensations are not rollbacks — design for the visible intermediate states.

**sandboxing** — Executing untrusted or agent-generated code in an isolated environment (container, microVM, WASM) with restricted filesystem/network/syscalls. Mandatory for agents that execute code; the sandbox boundary, not the prompt, is the security control.

**SBOM (software bill of materials)** — Machine-readable inventory (SPDX, CycloneDX) of every component in a build. Enables answering "are we exposed to CVE-X?" in minutes; generate per build in CI, not annually by hand.

**schema migration** — Versioned, ordered, automated database schema changes applied via tooling. Use expand-migrate-contract for zero-downtime: add new alongside old, backfill and dual-write, then remove old only after full rollout.

**scratchpad** — Working memory an agent writes intermediate reasoning, plans, and partial results into — a notes file or dedicated context region — surviving beyond a single turn without polluting the main conversation.

**secret scanning** — Automated detection of credentials in code, commits, and configs, blocking at pre-commit/CI and revoking on detection. Any secret that touched git history is compromised — rotate it; deleting the commit is not remediation.

**semver (semantic versioning)** — MAJOR.MINOR.PATCH: breaking change / new backward-compatible feature / fix. The contract that makes dependency ranges safe. Trap: semver describes the API surface, not effort — a one-line breaking change is still a major bump.

**serverless** — Running code as managed, event-triggered functions that scale to zero, billed per invocation. Fits spiky and glue workloads; watch cold starts, execution limits, and per-invocation cost at sustained high volume.

**service mesh** — Infrastructure layer (sidecars or node proxies: Istio, Linkerd) handling service-to-service mTLS, retries, timeouts, traffic splitting, and telemetry uniformly, outside application code. Adopt for fleet-wide policy needs, not because it exists.

**session auth vs token auth** — Session: server stores state, browser holds an opaque cookie — instantly revocable, ideal for first-party web. Token (JWT): stateless, self-contained — scales across services but cannot be revoked before expiry without extra state. Pick revocability or statelessness per surface.

**sharding** — Horizontal partitioning of data across nodes by a shard key. Choose the key by access pattern and growth: cross-shard queries and rebalancing are the ongoing costs, and a hot key defeats the whole scheme.

**sidecar** — Helper container deployed alongside an application container in the same pod, adding capabilities (proxying, TLS, log shipping) without touching app code.

**skill** — Packaged, on-demand instruction set an agent loads for a specific task type — workflow steps, conventions, reference material (SKILL.md plus resources). Skills are procedural memory: they encode "how we do X here" once, instead of re-explaining per session.

**slash command** — User-invoked named prompt/workflow in an agent interface (`/tdd`, `/code-review`, `/plan`) that expands to a full instruction set. vs skill: commands are explicit user invocations; skills can auto-load on relevance.

**SLA (service level agreement)** — Contract with external consequences (credits, penalties) if service falls below stated levels. Set SLAs looser than internal SLOs so the SLO breach alarm fires before the contract breach does.

**SLI (service level indicator)** — The measured signal of service health: success rate, p99 latency, freshness. Measure as close to user experience as possible — server-side success rate misses what the CDN and network did to users.

**SLO (service level objective)** — Internal target for an SLI over a window ("99.9% of requests succeed over 30 days") defining the error budget. Set from user needs, not current performance; more nines than users notice is money burned.

**snapshot testing** — Auto-capturing output (rendered component, API response) and failing when it changes, with one-command updates. Trap: reflexive snapshot-updating turns the suite into a change detector that approves everything — review snapshot diffs like code.

**speculative decoding** — Inference acceleration where a small draft model proposes tokens and the large model verifies them in parallel, yielding identical output faster.

**spy** — Test double wrapping a real implementation, recording calls while preserving behavior — verify interactions without replacing logic. vs mock: a mock replaces behavior; a spy observes it.

**SQL injection (SQLi)** — Attacker input altering query structure due to string-concatenated SQL. Fully prevented by parameterized queries/prepared statements — string-building SQL from user input is never acceptable, including in LLM-generated queries.

**SSE (server-sent events)** — One-way server-to-client streaming over plain HTTP (`text/event-stream`) with built-in reconnection. The standard transport for LLM token streaming. vs WebSocket: SSE is server-push only but simpler and proxy-friendly.

**SSRF (server-side request forgery)** — Tricking a server into making requests to attacker-chosen URLs — classically cloud metadata endpoints (169.254.169.254) or internal services. Any "fetch this URL" feature (including agent web tools) needs allowlists, IP-range blocking, and no-redirect-following-into-private-space.

**stop condition** — Explicit rule ending an agent loop: goal achieved, budget exhausted, iteration cap, error threshold, or human-abort. Every autonomous loop needs one enforced by the harness; "the model will know when to stop" is not a stop condition.

**strangler fig pattern** — Incremental legacy migration: route traffic through a façade, replace functionality piece by piece behind it, retire the old system when nothing remains. The alternative to big-bang rewrites, which mostly fail.

**structured output** — Constraining model output to a schema (JSON mode, grammar-constrained decoding, tool-call shape) so downstream code parses it reliably. Still validate: schema conformance does not guarantee semantic correctness.

**stub** — Test double returning canned answers to calls made during the test, with no interaction verification. Simplest double; use when the test only needs the dependency to answer, not to be observed.

**subagent** — Scoped child agent spawned for a subtask with its own fresh context window, returning a distilled result to the parent. Isolates context (research, review, parallel work) and contains context rot; costs handoff fidelity — pass complete instructions, not references to parent context.

**supply-chain attack** — Compromising software via its dependencies, build pipeline, or distribution: typosquatted packages, hijacked maintainers, poisoned build steps, malicious model weights or MCP servers. Defend with lockfiles, pinning, provenance attestation, SBOMs, and minimal dependency surface.

**system prompt** — The instruction block establishing an agent's role, constraints, tools, and behavior before any user input. Highest-priority steerable context, but not a security boundary — anything truly forbidden must be enforced outside the model.

## T

**TDD (test-driven development)** — Red-green-refactor: write a failing test, write minimal code to pass, refactor with the safety net. The design pressure (testable seams, small units) is as valuable as the tests; particularly effective for keeping LLM-generated code honest — the test defines done.

**tech debt** — Future cost accepted for present speed: shortcuts, missing tests, outdated dependencies, wrong abstractions. Manage it explicitly (track, budget paydown); unacknowledged debt compounds into velocity collapse.

**temperature** — Sampling parameter scaling randomness: low (~0-0.3) → deterministic-ish, good for extraction, code, tool use; high (~0.8+) → diverse, good for ideation. Trap: temperature 0 still does not guarantee reproducibility across runs or model versions.

**test pyramid** — Test-portfolio heuristic: many fast unit tests, fewer integration tests, few E2E tests — cost and flakiness rise with scope. vs test trophy: the trophy (favoring integration tests as the bulk) fits systems where units are trivial glue and boundaries are where bugs live.

**test trophy** — Portfolio shape emphasizing integration tests as the largest layer, with static analysis at the base, some unit, and few E2E. Choose pyramid vs trophy by where your defects actually occur, not by slogan.

**threat model** — Structured analysis of what you are protecting, from whom, via which attack surfaces, and which mitigations apply (e.g. STRIDE). For agent systems, enumerate: injected content, tool misuse, credential scope, and exfiltration channels. Do it at design time; retrofitted security is patchwork.

**token** — The subword unit models read and emit (~4 English characters on average; code and non-English tokenize less efficiently). Everything is priced and budgeted in tokens: context limits, API cost, latency.

**token budget** — Explicit allocation of context/spend across an agent task — system prompt, references, history, retrieval, output — enforced by the harness. Budgeting forces prioritization; unbudgeted context fills with the loudest content, not the most useful.

**tool poisoning** — Malicious instructions hidden in a tool's own metadata — descriptions, parameter docs, error messages — which the model reads as trusted context. Review third-party tool schemas as untrusted input; see also rug-pull (MCP).

**tool schema tax** — The context-window cost of declaring tools: every tool's name, description, and JSON schema is paid on every request. Dozens of attached tools degrade both budget and tool-selection accuracy — curate per task, or defer/lazy-load schemas.

**tool use** — An agent invoking external capabilities (search, file edit, API calls) via function calling in a loop: model requests call → harness executes → result returns to context. The harness owns execution and permissions; the model only ever proposes.

**top-p (nucleus sampling)** — Sampling from the smallest token set whose cumulative probability exceeds p, truncating the long tail. Adjust temperature or top-p, not both aggressively — they interact.

**traces (distributed tracing)** — Request-scoped records of causally-linked spans across services, showing where time went and what failed; one of the three observability pillars. Propagate trace context (W3C traceparent) across every hop, including queues and agent tool calls.

**trajectory** — The full recorded sequence of an agent run: prompts, reasoning, tool calls, results, final state. The unit of agent debugging and evaluation — store trajectories; scoring only final answers hides how the agent got there.

**transformer** — Neural architecture built on self-attention, processing sequences in parallel rather than recurrently; the foundation of modern LLMs. Its attention cost is why context length is a first-class engineering constraint.

**trunk-based development** — Everyone merges small changes to main frequently (at least daily); long-lived branches are avoided; incomplete features hide behind flags. Pairs with CI/CD and feature flags; the alternative — long-lived branches — trades merge pain later for isolation now.

**TTL (time to live)** — Expiry duration on cached or stored data (cache entries, DNS records, tokens, queue messages). The blunt-but-reliable staleness bound when explicit invalidation is impractical.

**twelve-factor app** — Methodology for deployable services: config in environment, stateless processes, backing services as attached resources, logs as event streams, disposability, dev-prod parity. Still the baseline checklist for anything running in containers.

## U

**ubiquitous language (DDD)** — Shared, precise vocabulary between domain experts and code within a bounded context — the same terms in conversation, models, and class names. When code words diverge from business words, translation bugs follow.

**unit test** — Fast, isolated test of a small unit's behavior through its public interface, with external dependencies doubled. Test behavior, not implementation: a refactor that preserves behavior should not break unit tests.

## V

**vector database** — Store optimized for embedding similarity search via ANN indexes (HNSW/IVF), with metadata filtering and hybrid keyword+vector queries. Trap: for modest scale, pgvector in your existing Postgres beats operating a new database.

**vertical vs horizontal scaling** — Vertical: bigger machine — simple, has a ceiling, restart to resize. Horizontal: more machines — elastic, but demands statelessness or partitioning. Scale vertically until the ceiling or availability needs force horizontal.

**vibe coding** — Building software primarily by directing an AI agent in natural language, reviewing outcomes over reading every line. Viable for prototypes; for production, pair it with tests, review gates, and CI evals — accountability does not transfer to the model.

## W

**WAF (web application firewall)** — Edge filter inspecting HTTP traffic against attack patterns (SQLi, XSS signatures) and rate rules. A mitigation layer, not a fix — the vulnerable code is still vulnerable behind it.

**webhook** — Provider-to-consumer HTTP callback on events (push, payment, deploy). Consumers must verify signatures, respond fast (enqueue, then process), and be idempotent — providers retry, so duplicates are guaranteed.

**WebSocket** — Persistent bidirectional TCP-based connection upgraded from HTTP, for real-time two-way traffic (chat, collaboration, live games). vs SSE: choose WebSocket only when the client must push frequently; otherwise SSE is simpler.

**write-ahead log (WAL)** — Append-only log of changes written before applying them to data structures, guaranteeing crash recovery by replay. Also the tap point for CDC and the shipping mechanism for replication.

## X

**XSS (cross-site scripting)** — Injecting attacker script into pages other users view (stored, reflected, or DOM-based), hijacking sessions and actions. Defend with context-aware output encoding, framework auto-escaping left on, and CSP as backstop — sanitizing input alone is insufficient.

## Y

**YAGNI ("you aren't gonna need it")** — Do not build for speculative future requirements; build for demonstrated ones. Applies doubly to speculative abstraction layers and premature microservices; the future you guessed rarely arrives in the shape you built for.

## Z

**zero-day** — Vulnerability exploited before a patch exists. You cannot patch ahead of one; defense in depth, least privilege, and detection determine blast radius.

**zero-downtime deployment** — Releasing without user-visible interruption: rolling/blue-green/canary rollout, connection draining, backward-compatible schema and API changes across the overlap window where old and new run simultaneously.

**zero-shot** — Prompting a model to perform a task from instructions alone, no examples. Baseline to try first; add few-shot examples when format or judgment must be demonstrated.

**zero trust** — Security model with no trusted network zone: every request is authenticated, authorized, and encrypted regardless of origin ("never trust, always verify"). For agents: authenticate the agent per request and authorize per action — network position proves nothing.

## How to use this volume

- Look up terms by letter section, then by bolded term; entries are alphabetical within each letter.
- Read the "Trap:" note before acting on any practice — it encodes the failure mode seen in the wild.
- Use the "vs" disambiguations verbatim when a user conflates two terms; they are written to be quotable.
- When a term is absent here, say so rather than inventing a definition; this volume is the reference of record for the fedskill set.
