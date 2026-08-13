# Implementation Plan: Add Support for Body Temperature Metric Ingestion, Storage, and Reporting

**Branch**: `MVWICA-8` | **Date**: 2026-08-12 | **Spec**: [specs/MVWICA-8/spec.md](spec.md)
**Input**: Feature specification from `/specs/MVWICA-8/spec.md`

---

## Summary

Adds body temperature as a first-class health metric across six Sapphire repositories. A new REST ingestion endpoint (Python/FastAPI, `sapphire-event-ingestion-api`) accepts Celsius/Fahrenheit readings from API-key-authenticated device submitters, validates them against a configurable physiological range, and publishes Avro-encoded events to a Kafka topic. A Kafka Connect sink connector (`sapphire-kafka-pipeline`) streams those events into a PostgreSQL/TimescaleDB hypertable. The charting service (Java/Spring Boot, `sapphire-charting-api`) exposes a trend endpoint returning daily/weekly/monthly min-max-avg aggregates. The BFF (`sapphire-bff-api`) adds GraphQL queries for temperature chart data using JWT-validated resolvers. The React dashboard (`Sapphire`) gains a `BodyTemperatureChart` feature component with time-range selector and persisted Celsius/Fahrenheit unit preference. The Kafka Streams consumer (`sapphire-kafka-streams-consumer`) is extended with an operator-configured temperature threshold rule that emits alerts to the existing alert topic.

---

## Technical Context

**Languages/Versions**:
- Python 3.11 — `sapphire-event-ingestion-api`
- Java 17 — `sapphire-charting-api`, `sapphire-kafka-streams-consumer`
- Node.js 20 / TypeScript — `sapphire-bff-api`
- TypeScript / React 18 — `Sapphire`
- JSON (Kafka Connect configuration) — `sapphire-kafka-pipeline`

**Primary Dependencies**:
- FastAPI, Pydantic v2, `confluent-kafka` (Avro), `structlog`, `opentelemetry-sdk` — ingestion API
- Spring Boot 3, Spring Kafka, TimescaleDB JDBC, `opentelemetry-spring-boot-starter`, Logback+logstash-encoder — charting API + Kafka Streams consumer
- Apollo Server 4, `graphql`, `dataloader`, `pino`, `@opentelemetry/sdk-node` — BFF
- React 18, Apollo Client 3, Recharts (or existing chart library in use), Keycloak-js — UI

**Storage**:
- PostgreSQL 16 + TimescaleDB — `temperature_readings` hypertable, `temperature_rollups` table
- Kafka topic: `health-metrics.temperature` (or shared `health-metrics` topic with metric-type discriminator — see research.md)

**Testing**:
- Python: `pytest` + `pytest-asyncio`, `httpx` test client, Testcontainers for Kafka/PostgreSQL integration tests
- Java: JUnit 5, Mockito, `@WebMvcTest`, `@DataJpaTest`, Testcontainers
- TypeScript/BFF: Jest, mocked backend services for resolver integration tests
- React: Vitest + React Testing Library

**Target Platform**: Linux server (Docker Compose local), Kubernetes production

**Performance Goals**:
- Ingestion: single reading p95 ≤ 200 ms end-to-end (endpoint → Kafka ACK)
- Pipeline lag: event → PostgreSQL within 10 s (SC-001)
- Chart API: p95 ≤ 500 ms for 30-day range query
- UI chart render: ≤ 2 s from user selection (SC-004)
- Alert delivery: ≤ 15 s from ingestion (SC-008)

**Constraints**:
- Batch ingestion: up to 100 records per request, processed within 30 s (SC-002)
- Rate limit: 100 req/min per API key; HTTP 429 + `Retry-After` on breach (FR-001)
- No per-user threshold customisation in this story (FR-026 scope)
- Sibling repository source access unavailable — all decisions based on AGENTS.md patterns; must be validated during implementation

**Scale/Scope**: 6 affected repositories, ~30 functional requirements, 5 user stories

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| # | Gate | Principle | Status |
|---|------|-----------|--------|
| 1 | All public functions/methods/classes have docstrings or Javadoc (intent, not implementation) | I. Code Quality | ✅ Planned — all new FastAPI route handlers, Spring services, and React hooks will have docstrings/Javadoc per constitution |
| 2 | No magic numbers or strings — named constants or enums used | I. Code Quality | ✅ Planned — `TemperatureUnit` enum (CELSIUS/FAHRENHEIT), `MetricType.TEMPERATURE` constant, threshold values via `@ConfigurationProperties` |
| 3 | Cyclomatic complexity ≤ 10 per function/method confirmed via static analysis | I. Code Quality | ✅ Planned — `ruff` for Python, SpotBugs/Checkstyle for Java; validation and batch processing split into small focused functions |
| 4 | No commented-out code committed; feature flags or deletion used instead | I. Code Quality | ✅ Planned — enforced via PR review checklist |
| 5 | Stack-specific rules applied (Spring layering / PEP 8+ruff / strict TS / BFF DataLoader) | I. Code Quality | ✅ Planned — Spring Controller→Service→Repository; `ruff` lint; strict TS; DataLoader for chart data resolver if user has multiple metric types |
| 6 | Coverage gates planned: Java 80%/100% domain, Python 80%, TS/React 70%, BFF resolvers 100% | II. Testing Standards | ✅ Planned — see test strategy per repo in research.md |
| 7 | Contract tests planned for all GraphQL schema changes and Kafka event schema changes | II. Testing Standards | ✅ Planned — Avro schema compatibility test for temperature topic; GraphQL schema contract test for new queries |
| 8 | Test pyramid respected: unit (mocked I/O), integration (Docker Compose, pre-merge only, not validation), E2E | II. Testing Standards | ✅ Planned — unit tests mock Kafka/DB; integration tests tagged `@IntegrationTest` / `pytest.mark.integration`; E2E in `sapphire-playwright` |
| 9 | All data-fetching components handle loading skeleton / error boundary / empty state | III. UX Consistency | ✅ Planned — `BodyTemperatureChart` implements all three states per FR-025 and constitution |
| 10 | Auth path is exclusively Keycloak OIDC/PKCE; no bypass routes in any environment | III. UX Consistency | ✅ Planned — UI uses existing Keycloak OIDC; ingestion uses API key (separate path for Device Submitter persona, not a bypass of Keycloak) |
| 11 | URL state is source of truth for filters, pagination, and selections | III. UX Consistency | ✅ Planned — selected time range (day/week/month) encoded as URL query parameter |
| 12 | Apollo cache policies explicit; no implicit cache-first for mutable health data | I. Code Quality | ✅ Planned — `cache-and-network` policy on temperature chart query; unit preference mutation uses `no-cache` |
| 13 | All services emit structured JSON logs with trace_id and span_id fields | IV. Observability | ✅ Planned — `structlog` JSONRenderer (Python), Logback+logstash-encoder (Java), `pino` JSON (BFF) |
| 14 | OTEL metrics: request count, duration, error rate, in-flight; custom business metrics | IV. Observability | ✅ Planned — FR-028/029/030: `temperature_readings_ingested_total`, `temperature_readings_rejected_total`, `temperature_alerts_generated_total` |
| 15 | Distributed traces via OTEL SDK; W3C traceparent; DB/HTTP/Kafka as child spans | IV. Observability | ✅ Planned — `opentelemetry-instrumentation-fastapi` (Python), `opentelemetry-spring-boot-starter` (Java), `@opentelemetry/sdk-node` (BFF) |
| 16 | OTEL env vars set in every container | IV. Observability | ✅ Planned — `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, `OTEL_DEPLOYMENT_ENVIRONMENT` added to each service's Docker config |
| 17 | LangGraph rules (N/A — no LangGraph involved) | I. Code Quality | N/A |

---

## Project Structure

### Documentation (this feature)

```text
specs/MVWICA-8/
├── plan.md              ← this file
├── research.md          ← Phase 0 output
├── data-model.md        ← Phase 1 output
├── quickstart.md        ← Phase 1 output
├── contracts/
│   ├── ingestion-api.yaml          ← OpenAPI 3.1 for POST /temperature-readings
│   ├── charting-api.yaml           ← OpenAPI 3.1 for GET /charts/temperature
│   ├── bff-schema-additions.graphql ← GraphQL SDL additions
│   └── kafka-avro-schema.json      ← Avro schema for temperature events
└── tasks.md             ← Phase 2 output (created by /speckit.tasks)
```

### Source Code Layout (per affected repo)

```text
# sapphire-event-ingestion-api  (Python/FastAPI)
src/
├── routers/
│   └── temperature.py          ← new: POST /temperature-readings (single + batch)
├── models/
│   └── temperature.py          ← new: TemperatureReading, TemperatureBatchRequest Pydantic v2 models
├── services/
│   └── temperature_service.py  ← new: validation, unit handling, Kafka publish
├── kafka/
│   └── temperature_producer.py ← new: Avro producer for health-metrics.temperature topic
└── config/
    └── temperature_config.py   ← new: PhysiologicalRange, RateLimit settings via env vars

# sapphire-kafka-pipeline  (Kafka Connect JSON)
connectors/
└── temperature-sink.json       ← new: Kafka Connect PostgreSQL sink connector config

# sapphire-charting-api  (Java/Spring Boot)
src/main/java/.../temperature/
├── TemperatureChartController.java  ← new: GET /charts/temperature
├── TemperatureChartService.java     ← new: aggregate query logic
├── TemperatureChartRepository.java  ← new: TimescaleDB time-bucket queries
└── dto/
    ├── TemperatureChartRequest.java   ← new: record DTO
    └── TemperatureChartResponse.java  ← new: record DTO

# sapphire-bff-api  (Node.js/Apollo Server)
src/
├── schema/
│   └── temperature.graphql     ← new: Query.temperatureChart SDL
├── resolvers/
│   └── temperature.resolver.ts ← new: JWT-validated, delegates to charting-api
└── datasources/
    └── TemperatureDataSource.ts ← new: typed HTTP client for charting-api

# Sapphire  (React/TypeScript)
src/features/body-temperature/
├── BodyTemperatureChart.tsx     ← new: main chart component
├── useBodyTemperature.ts        ← new: Apollo query hook + unit preference logic
├── BodyTemperatureChart.types.ts ← new: prop interfaces
├── BodyTemperatureChart.test.tsx ← new: RTL unit tests
└── queries/
    └── temperatureChart.graphql ← new: Apollo query document

# sapphire-kafka-streams-consumer  (Java/Spring Boot)
src/main/java/.../temperature/
├── TemperatureAlertProcessor.java  ← new: Kafka Streams topology for threshold alerting
├── TemperatureThresholdConfig.java ← new: @ConfigurationProperties for thresholds
└── dto/
    └── TemperatureAlertEvent.java  ← new: record DTO for alert topic
```

---

## Complexity Tracking

No constitution violations requiring justification. All additions follow existing platform patterns.

---

## Architecture Decisions

### AD-001: Kafka Topic Strategy — Dedicated vs Shared

**Decision**: Use a dedicated Kafka topic `health-metrics.temperature` following the existing per-metric naming convention inferred from AGENTS.md (separate Kafka Connect sink connectors per metric type imply separate topics).

**Rationale**: Dedicated topics allow independent scaling, independent schema evolution (each topic has its own Avro schema), and independent sink connector configuration. Shared topics would require a discriminator field and complicate schema registry management.

**Alternatives considered**: Single shared `health-metrics` topic with a `metricType` discriminator — rejected because it couples all metric schema changes and cannot independently control sink connector parallelism.

### AD-002: Storage — New Hypertable vs Extending Existing

**Decision**: Create a new `temperature_readings` hypertable in TimescaleDB following the same pattern as existing metric hypertables, plus a `temperature_rollups` materialised view for aggregates.

**Rationale**: Each metric type has its own schema (different fields — temperature has `unit`, blood pressure has systolic/diastolic). A shared metrics table with JSONB for values would work but sacrifices type safety and index performance. Following the per-metric hypertable pattern is consistent with existing platform design.

**Alternatives considered**: JSONB column in a generic `health_metrics` table — deferred as a future normalisation effort if the number of metric types grows significantly.

### AD-003: API Key Management for Device Submitters

**Decision**: API keys are managed as environment-variable-injected configuration or a simple key-to-user-mapping store in the ingestion service. Key validation is performed in a FastAPI dependency (`Depends`). No new user-facing key management UI in this story.

**Rationale**: Minimal viable implementation. Key issuance and rotation UI is a separate operational concern. The dependency injection pattern aligns with FastAPI conventions (FR-001, constitution).

**Alternatives considered**: Full API key management service — deferred to a follow-on story.

### AD-004: Unit Preference Persistence

**Decision**: The End User's temperature unit preference (Celsius/Fahrenheit) is stored as a new field `temperature_unit_preference` on the existing user profile entity in `sapphire-user-service`, updated via the BFF using a new GraphQL mutation `updateUserPreference`. The UI reads the preference on chart load via the existing user profile query.

**Rationale**: The spec (FR-023/024) requires the preference to persist across sessions. The user profile service already owns user preferences; extending it with one field is the minimal change. Storing it client-side (localStorage) would not synchronise across devices.

**Alternatives considered**: New dedicated preferences service — overkill for a single field. localStorage — not cross-device.

> ⚠️ **Scope note**: This adds `sapphire-user-service` as a sixth affected repo for a minor field addition. The existing user profile PATCH endpoint is assumed to exist; if not, a new endpoint is required.

### AD-005: Rate Limiting Implementation

**Decision**: Rate limiting (100 req/min per API key) is implemented as a FastAPI middleware/dependency in `sapphire-event-ingestion-api` using an in-process sliding window counter backed by a Redis client (existing Redis infrastructure per AGENTS.md). Returns HTTP 429 with `Retry-After` header.

**Rationale**: Redis is already in the platform stack. An in-process counter would not survive restarts or scale horizontally. Redis sliding window is the standard pattern for distributed rate limiting in the Sapphire stack.

**Alternatives considered**: API Gateway-level rate limiting — valid but outside this story's scope (no gateway change specified). Token bucket in-process — does not survive horizontal scaling.

---

## Implementation Sequence

Dependencies flow strictly top-to-bottom. Each step can only begin once its predecessor is complete.

```
Step 1 [sapphire-event-ingestion-api]  Avro schema + Kafka producer + ingestion endpoint
        ↓
Step 2 [sapphire-kafka-pipeline]       Kafka Connect sink connector config
        ↓
Step 3 [sapphire-charting-api]         TimescaleDB hypertable + chart endpoint
        ↓
Step 4 [sapphire-bff-api]              GraphQL schema additions + resolver
        ↓ (parallel with Step 4)
Step 4b [sapphire-user-service]        temperature_unit_preference field + PATCH endpoint
        ↓
Step 5 [Sapphire]                      BodyTemperatureChart component + unit preference toggle
        ↓ (independent — can run in parallel with Steps 3–5)
Step 6 [sapphire-kafka-streams-consumer] Temperature threshold alert rule
```
