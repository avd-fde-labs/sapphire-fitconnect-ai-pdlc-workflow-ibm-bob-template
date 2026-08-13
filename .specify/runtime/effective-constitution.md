<!-- EFFECTIVE CONSTITUTION
     Generated  : 2026-04-02T00:00:00Z
     Global     : settings.yaml -> constitution.global_source.mode=local, path=org-constitution/constitution.md
     Local      : not applied
     Precedence : global only (no local constitution applied)
-->

# Effective Constitution

**Generated:** 2026-04-02T00:00:00Z
**Mode:** Global-only (local constitution intentionally not applied).

> No local constitution was applied in this run. All rules below are authoritative as-is.

---

# Constitution

## Core Principles

### I. Code Quality

All public functions, methods, and classes MUST have a docstring or Javadoc comment
explaining intent, not implementation. No magic numbers or strings — use named constants
or enums. Functions MUST do one thing; if a block inside a function needs a comment to
explain what it does, that block MUST be extracted into a named function or method.

Cyclomatic complexity MUST NOT exceed 10 per function/method, enforced via static analysis
in CI. Commented-out code MUST NOT appear in any commit; use feature flags or delete it.

**Java / Spring Boot**

- Follow standard Spring layering: Controller → Service → Repository. Business logic in
  controllers or repositories is FORBIDDEN.
- Use `record` types for immutable DTOs. Use `sealed interface` for discriminated domain
  types.
- Prefer constructor injection. Field injection via `@Autowired` is FORBIDDEN in
  non-test production code.
- Define domain-specific checked exceptions for recoverable errors. Unchecked exceptions
  are reserved for programming errors only.
- `Optional<T>` MUST be returned for nullable values. Returning `null` from any public
  method is FORBIDDEN.
- Java compilation MUST pass before merge (for example, `mvn -q -DskipTests compile` or
  `./gradlew compileJava` in the affected repo).

**Python / FastAPI**

- Follow PEP 8. `ruff` MUST be used for linting and formatting (line length 100).
- Pydantic v2 models MUST be used for all request/response schemas and internal data
  contracts.
- All FastAPI route handlers MUST be `async`. `httpx` MUST be used for async HTTP calls;
  the `requests` library is FORBIDDEN in service code.
- Every function signature MUST be fully type-annotated including return types. Bare
  `except:` clauses are FORBIDDEN; catch specific exception types only.
- Shared resources (DB sessions, HTTP clients, configuration) MUST be injected via
  FastAPI `Depends`.
- Python compilation checks MUST pass before merge (for example,
  `python -m compileall -q .` in the affected repo).

**REST API Contracts (all services)**

- All REST API contracts MUST be resource-based. Endpoint paths MUST use plural
  resource nouns (for example, `/users`, `/orders/{orderId}/items`) and MUST NOT
  use verb-style action paths (for example, `/createUser`, `/getOrders`,
  `/calculateScore`).
- CRUD semantics MUST map to standard HTTP methods (`GET`, `POST`, `PUT`, `PATCH`,
  `DELETE`) on resources. RPC-style action tunneling over REST paths is FORBIDDEN
  unless explicitly approved in the feature specification with documented rationale.
- Resource identifiers MUST be path parameters and filtering/pagination inputs MUST
  be query parameters. Request bodies for `GET` endpoints are FORBIDDEN.
- REST contract artifacts (OpenAPI specs, endpoint docs, and tests) MUST reflect the
  same resource model and naming conventions as implemented routes.

**LangGraph / AI Agents (sapphire-wellness-agent, sapphire-wellness-coach)**

- Graph state MUST be defined as a `TypedDict` with fully `Annotated` fields specifying
  the appropriate reducer (e.g., `operator.add` for append-only lists). Raw `dict` state
  is FORBIDDEN.
- Each graph node MUST be a single-responsibility `async` function accepting and
  returning state. Node functions MUST NOT perform DB or HTTP I/O without injected async
  clients (passed via `RunnableConfig` extras or constructor-injected dependencies).
- Conditional routing MUST use named routing functions returning string literals that
  match registered edge targets. Anonymous `lambda` routing callables are FORBIDDEN.
- All production LangGraph agents MUST use a persistent checkpointer
  (`AsyncPostgresSaver` or equivalent). `MemorySaver` is permitted in unit tests and
  local development only.
- Tool definitions MUST use Pydantic v2 `BaseModel` subclasses for their input schema.
  Bare `dict` or untyped `args_schema` is FORBIDDEN.
- Graphs MUST be compiled once at application startup and reused across requests.
  Compiling a new `StateGraph` per request is FORBIDDEN.
- Human-in-the-loop pauses MUST be implemented using LangGraph's `interrupt_before` or
  `interrupt_after` mechanism. Polling loops or arbitrary `asyncio.sleep` for approval
  waiting are FORBIDDEN.
- Node failures MUST be caught and encoded into a typed `error` field in the state.
  Unhandled exceptions that propagate out of a node and crash the graph are FORBIDDEN.
- LangGraph agent execution MUST emit trace data via LangSmith or an OTEL-compatible
  callback. Silent graph execution without observable trace output is FORBIDDEN in
  non-local environments.
- Unit tests MUST test each node in isolation with minimal constructed state dicts.
  Integration tests MUST run the compiled graph end-to-end using `MemorySaver` with
  deterministic LLM stubs or recorded cassettes.

**TypeScript / React (Sapphire UI)**

- Strict TypeScript MUST be enabled project-wide. `any` is FORBIDDEN. `ts-ignore`
  requires an explanatory comment and a linked ticket reference.
- Class components are FORBIDDEN; use functional components only.
- Feature code MUST be co-located: one directory per feature containing the component,
  its hook, its types, and its tests.
- Apollo cache policies MUST be explicit on every query. Implicit `cache-first` for
  mutable health data is FORBIDDEN.
- Raw `fetch` calls inside components are FORBIDDEN. All API interaction MUST go through
  the Apollo client or a typed service module.
- TypeScript compilation MUST pass before merge (for example, `tsc --noEmit` or the repo's equivalent compile script).

**Node.js / BFF (sapphire-bff-api)**

- All GraphQL resolvers MUST validate JWT claims before delegating to backend REST calls.
- Inline SQL and raw REST URLs are FORBIDDEN. Use typed resolver helpers and
  environment-configured service clients.
- `DataLoader` MUST be used for any field resolver that could trigger N+1 calls.

---

### II. Testing Standards

**Coverage Gates (enforced in CI)**

- Java services: 80% line coverage minimum; domain service classes require 100% coverage.
- Python services: 80% line coverage minimum; all Pydantic model validators MUST have
  explicit unit tests.
- TypeScript/React: 70% line coverage minimum; all custom hooks MUST have unit tests
  written with React Testing Library.
- BFF (Node.js): all resolvers MUST have integration tests run against mocked backend
  services (run during pre-merge stage, not validation stage).

**Test Pyramid**

- **Unit tests** MUST mock all I/O (DB, HTTP, Kafka) and MUST complete in < 5 seconds
  total per module.
- **Integration tests** MUST use real infrastructure (PostgreSQL, Kafka, Qdrant) via
  Docker Compose. They MUST be tagged `@IntegrationTest` in Java and
  `pytest.mark.integration` in Python.
- **E2E tests** MUST live exclusively in `sapphire-playwright` and cover complete user
  journeys through the UI. They MUST NOT duplicate integration-level assertions.
- **Contract tests** are REQUIRED for every GraphQL schema change in `sapphire-bff-api`
  and every Kafka event schema change. Contracts MUST be verified against all known
  consumers before merge.

**Test Quality Rules**

- String literals appearing in more than one test MUST be extracted to constants.
- Arrange-Act-Assert (AAA) structure is MANDATORY. Each block MUST be visually separated
  with a blank line or comment.
- Tests MUST NOT share mutable state. Static mutable fields in test classes are FORBIDDEN.
- Flaky tests are P1 bugs. A flaky test MUST be quarantined and fixed before the end of
  the sprint in which it is discovered.

**Test Execution Stages**

- **Validation Stage (PR/CI)**: Unit tests only. Integration tests are **explicitly excluded** from validation. Validation includes:
  - Lint checks (ruff for Python, ESLint for TypeScript)
  - Unit tests with coverage gate enforcement
  - Mandatory compilation checks (Java `mvn compile`, Python `python -m compileall`, TypeScript `tsc --noEmit`)
  - Security scanning (SCA, dependency check)
  - Contract tests for schema changes
- **Pre-merge / Integration Tests**: Full test pyramid including integration tests with real infrastructure (Docker Compose). Integration tests run **after** validation passes but **before** merge to main.
- **Post-merge (Nightly/Staging)**: E2E tests in `sapphire-playwright` against staging environment.

Integration tests MUST NOT block CI validation pipelines. They MUST run in a separate, parallel stage.

---

### III. User Experience Consistency

**Loading & Error States**

- Every data-fetching component MUST handle three explicit states: loading skeleton, error
  boundary with retry action, and empty state. Rendering partially loaded data without a
  visual indicator is FORBIDDEN.
- Skeleton screens MUST match the dimensions of the loaded content to prevent layout
  shift.
- Error messages shown to users MUST be actionable (e.g., "Try again" or "Contact
  support"). Exposing stack traces or internal error codes to users is FORBIDDEN.

**Auth & Session**

- Keycloak OIDC/PKCE MUST be the only authentication path. Bypass routes are FORBIDDEN
  in all environments, including local development.
- Token expiry MUST be handled silently via refresh. A login redirect caused by a routine
  token refresh is a UX defect.
- Unauthorized (401) and Forbidden (403) states MUST render distinct, user-friendly
  pages; blank screens are FORBIDDEN.

**Navigation & State**

- URL state MUST be the source of truth for page-level filters, pagination, and selected
  items, so that deep links always work.
- Browser back/forward MUST work correctly for all primary navigation flows.
- All interactive elements MUST have keyboard navigation support and ARIA labels. WCAG AA
  compliance is the minimum accessibility bar.

**Design Consistency**

- The shared design token system MUST be used for all spacing, color, and typography.
  Hardcoded hex values and pixel values outside the token scale are FORBIDDEN.
- Toasts and notifications MUST use the shared notification component. Ad-hoc `alert()`
  calls and custom toast implementations are FORBIDDEN.
- Charts rendered via `sapphire-charting-api` MUST follow the shared color palette for
  health metric categories to ensure visual consistency across all dashboard views.
- Any new UI screen should maintain consistency with the existing screens.
- UI changes MUST preserve the existing product theme, design language, and token system. Introducing a new visual theme or conflicting styling patterns is FORBIDDEN unless explicitly approved in the story/specification.

---

### IV. Observability

Every service MUST be observable via structured logs, metrics, and distributed traces
exported through OpenTelemetry (OTEL). Observability is not optional — it is a
requirement for any service that reaches production.

**Structured JSON Logging**

- All services MUST emit logs as structured JSON to stdout. Human-readable plain-text
  log output is FORBIDDEN in non-local environments.
- Every log record MUST include at minimum: `timestamp` (ISO-8601), `level`, `service`,
  `trace_id`, `span_id`, `message`, and `environment`.
- Log levels MUST follow the standard severity ladder: `DEBUG`, `INFO`, `WARN`, `ERROR`,
  `FATAL`. Using non-standard levels (e.g., `VERBOSE`, `TRACE` as a root level) is
  FORBIDDEN.
- Sensitive data (PII, tokens, passwords, health identifiers) MUST NOT appear in log
  fields. Mask or omit such values before logging.
- Java services MUST use Logback with `logstash-logback-encoder` for JSON output.
  Python services MUST use `structlog` configured with `JSONRenderer`. Node.js/BFF MUST
  use `pino` with JSON output mode.

**Metrics**

- All services MUST expose application metrics via the OTEL Metrics SDK and export to
  the configured OTEL Collector endpoint (`OTEL_EXPORTER_OTLP_ENDPOINT`).
- The following metrics are REQUIRED for every service:
  - Request/operation count (counter)
  - Request/operation duration (histogram with p50/p95/p99 buckets)
  - Error rate (counter, labelled by error type)
  - Active in-flight requests/tasks (up-down counter)
- Java services MUST use `opentelemetry-spring-boot-starter`. Python services MUST use
  `opentelemetry-sdk` + `opentelemetry-instrumentation-fastapi`. Node.js BFF MUST use
  `@opentelemetry/sdk-node` with auto-instrumentation.
- Business-level metrics (e.g., events ingested, recommendations served, summaries
  generated) MUST be emitted as custom OTEL counters — not derived solely from
  infrastructure metrics.

**Distributed Tracing**

- All services MUST instrument distributed traces using the OTEL Traces SDK and export
  spans to the configured OTEL Collector.
- Every inbound HTTP request and Kafka message MUST create a root span. Context
  propagation MUST use the W3C `traceparent` header — proprietary propagation formats
  are FORBIDDEN.
- Spans MUST include `service.name`, `service.version`, and `deployment.environment`
  resource attributes.
- Database queries, outbound HTTP calls, and Kafka produce/consume operations MUST each
  be represented as child spans with the relevant semantic conventions
  (`db.statement`, `http.url`, `messaging.destination`).
- Sampling strategy MUST be configured externally via `OTEL_TRACES_SAMPLER`. Hardcoding
  a sampler in application code is FORBIDDEN.

**OTEL Collector & Pipeline**

- A shared OTEL Collector MUST be the single egress point for all telemetry signal types
  (logs, metrics, traces). Services MUST NOT export directly to backend storage
  (e.g., Prometheus, Jaeger, Loki) — they export only to the Collector.
- Collector pipeline configuration (receivers, processors, exporters) MUST be version-
  controlled alongside infrastructure code.
- The `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, and
  `OTEL_DEPLOYMENT_ENVIRONMENT` environment variables MUST be set for every deployed
  container.

---

## Governance

This constitution supersedes all other development practices for the Sapphire workspace.
Every pull request MUST be reviewed for compliance with all four principles.
Non-compliance that cannot be justified blocks merge.

**Amendment Procedure**

1. Author opens a PR modifying `.specify/memory/constitution.md` with a written rationale.
2. At least two senior engineers (one per affected service domain) MUST approve.
3. The version MUST be incremented per the policy below before merge.
4. A migration note MUST be added to any spec, plan, or task artifact affected by the
   change within one sprint of ratification.

**Versioning Policy**

- **MAJOR** (x.0.0): A principle is removed, fundamentally redefined, or a
  non-negotiable rule is relaxed.
- **MINOR** (x.y.0): A new principle or sub-section is added, or materially expanded
  guidance is introduced.
- **PATCH** (x.y.z): Clarifications, wording improvements, typo fixes, or non-semantic
  refinements with no behavioral impact.

**Compliance Review**

- Compliance is evaluated at PR review time by the reviewing engineer.
- At the start of each sprint, the team lead audits quarantined flaky tests and any
  unresolved TODO constitution items.
- Architecture reviews for new microservices MUST use this constitution as the primary
  evaluation checklist.

**Version**: 2.2.0 | **Ratified**: 2026-03-21 | **Last Amended**: 2026-04-02
