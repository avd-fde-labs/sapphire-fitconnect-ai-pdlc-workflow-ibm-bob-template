# Research: MVWICA-8 — Body Temperature Metric

**Branch**: `MVWICA-8` | **Date**: 2026-08-12

---

## RES-001: Kafka Topic Strategy

**Decision**: Dedicated topic `health-metrics.temperature`

**Rationale**: Separate Kafka Connect sink connectors per metric type (inferred from AGENTS.md) imply separate topics. Dedicated topics give independent schema evolution, independent sink parallelism, and clean Avro schema registry entries.

**Alternatives considered**: Shared `health-metrics` topic with `metricType` discriminator — rejected; couples all metric schema changes and complicates per-metric consumer group management.

---

## RES-002: Avro Schema Compatibility

**Decision**: New standalone Avro schema `TemperatureReadingEvent` in the schema registry under subject `health-metrics.temperature-value`. Full schema below in `contracts/kafka-avro-schema.json`.

**Rationale**: Each metric type has a dedicated schema subject following standard Confluent Schema Registry convention. Forward and backward compatibility is enforced by the registry. The temperature schema is additive and does not touch existing metric schemas.

**Alternatives considered**: Reusing a generic `HealthMetricEvent` schema with a JSONB payload field — rejected; sacrifices type safety and Avro's schema-evolution guarantees.

---

## RES-003: TimescaleDB Hypertable Design

**Decision**: New `temperature_readings` hypertable partitioned by `measured_at` (time dimension). Chunk interval: 1 day. Composite index on `(user_id, measured_at DESC)`. Separate `temperature_rollups` continuous aggregate for daily/weekly/monthly aggregates using TimescaleDB `time_bucket`.

**Rationale**: TimescaleDB continuous aggregates materialise rollups automatically on ingestion — no separate rollup job required. The chunk interval of 1 day balances query performance against storage overhead for health telemetry volumes.

**Alternatives considered**: Manual rollup via scheduled PostgreSQL job — rejected; more operational overhead, higher latency for fresh aggregates. Single `health_metrics` JSONB table — rejected; see AD-002 in plan.md.

---

## RES-004: FastAPI Rate Limiting with Redis

**Decision**: FastAPI dependency `RateLimiter` using Redis INCR + EXPIRE (sliding window per API key). Key: `ratelimit:{api_key}:{minute_bucket}`. Limit: 100. On breach: HTTP 429, `Retry-After: {seconds_until_next_window}`.

**Rationale**: Redis is already in the platform stack (AGENTS.md confirms Redis for notification pub/sub). Sliding window prevents burst exploitation at window boundaries. The `Depends` pattern integrates cleanly with FastAPI's DI system per constitution requirements.

**Alternatives considered**: `slowapi` library — valid, but adds a dependency for a simple pattern already achievable with the existing Redis client. Fixed window — simpler but vulnerable to burst at window reset.

---

## RES-005: API Key Validation Strategy

**Decision**: API keys stored as a hashed key-to-user-ID map in a Redis hash (`apikeys:{hashed_key} → user_id`). Validation via FastAPI `Depends(get_api_key_user)`. Keys provisioned out-of-band by Platform Operators via environment variables or an admin script (no UI in this story).

**Rationale**: Redis lookup is sub-millisecond and already available. Hashing the stored key (SHA-256) prevents exposure if Redis is compromised. The user_id extracted from the key is attached to each ingested record.

**Alternatives considered**: PostgreSQL key table — valid but adds a synchronous DB query to every ingestion request. In-memory dict loaded from env var — does not scale horizontally across ingestion service replicas.

---

## RES-006: Unit Preference Persistence

**Decision**: Add `temperature_unit_preference` (VARCHAR(10), default `'CELSIUS'`, nullable) to the existing `users` table in `sapphire-user-service`. Exposed via existing user profile GET endpoint and a new PATCH `/users/{id}/preferences` endpoint (or the existing profile PATCH, if present). BFF adds a `updateTemperatureUnitPreference` mutation delegating to this endpoint.

**Rationale**: Minimal schema change. The user profile service already owns user settings. One new column is less risky than a new preferences service or table.

**Alternatives considered**: Separate `user_preferences` table — future-proof but overkill for a single field. localStorage — not cross-device. User profile JSONB `preferences` column — valid but unstructured; prefer typed column for indexed access.

---

## RES-007: Test Coverage Strategy

| Repo | Coverage Target | Key Test Areas |
|------|----------------|----------------|
| `sapphire-event-ingestion-api` | 80% line (Python) | `TemperatureService` validation logic (100%), Kafka producer mock, rate limiter, API key validation, batch partial-failure handling |
| `sapphire-kafka-pipeline` | N/A (configuration only) | Connector config schema validation test |
| `sapphire-charting-api` | 80% line, 100% service layer (Java) | `TemperatureChartService` aggregate logic, `TemperatureChartRepository` time-bucket query, `@WebMvcTest` for controller |
| `sapphire-bff-api` | 100% resolvers (BFF) | `temperatureChart` resolver with mocked `TemperatureDataSource`, JWT claim validation |
| `Sapphire` | 70% line (React/TS) | `useBodyTemperature` hook (loading/error/data states), `BodyTemperatureChart` render with RTL (loading skeleton, empty state, unit toggle) |
| `sapphire-kafka-streams-consumer` | 80% line, 100% service layer (Java) | `TemperatureAlertProcessor` threshold logic, no-alert scenario, alert publication |
| `sapphire-user-service` | 80% line, 100% service layer (Java) | `temperature_unit_preference` PATCH, GET includes preference field |

**Integration tests** (tagged `@IntegrationTest` / `pytest.mark.integration`, run pre-merge only, not in validation CI):
- Ingestion API → Kafka (Testcontainers Kafka)
- Kafka → PostgreSQL via Kafka Connect (Docker Compose)
- Charting API → PostgreSQL/TimescaleDB (Testcontainers)
- Kafka Streams consumer alert → alert topic (Testcontainers Kafka)

**E2E tests** (added to `sapphire-playwright`):
- End-user views temperature chart after device submits readings
- Unit toggle persists across logout/login

---

## RES-008: GraphQL Schema Evolution

**Decision**: Additive schema change only — new `temperatureChart` query and `TemperatureChartData` types added to the BFF schema. No existing types modified. `updateTemperatureUnitPreference` mutation added. A schema contract test verifies that no existing query is broken.

**Rationale**: Apollo Server 4 supports additive changes without a version bump. Constitution requires contract tests for all schema changes.

---

## RES-009: OTEL Instrumentation per Service

| Service | Instrumentation approach | Custom counters |
|---------|--------------------------|----------------|
| `sapphire-event-ingestion-api` | `opentelemetry-instrumentation-fastapi` auto + manual spans for Kafka publish | `temperature_readings_ingested_total`, `temperature_readings_rejected_total` |
| `sapphire-charting-api` | `opentelemetry-spring-boot-starter` auto | None new (standard request/duration metrics sufficient) |
| `sapphire-bff-api` | `@opentelemetry/sdk-node` auto | None new |
| `sapphire-kafka-streams-consumer` | `opentelemetry-spring-boot-starter` auto + manual span per processed record | `temperature_alerts_generated_total` |

---

## RES-010: Unresolved Items (deferred to implementation)

| Item | Risk | Mitigation |
|------|------|-----------|
| Actual existing Kafka topic naming convention in `sapphire-event-ingestion-api` | Medium — topic name may differ from assumed | Verify in repo source during Step 1 implementation |
| Existing user profile PATCH endpoint signature in `sapphire-user-service` | Medium — may not exist or may have different path | Verify in repo source; create new endpoint if absent |
| Existing chart library used in `Sapphire` (Recharts assumed) | Low — UI chart pattern may differ | Verify in `package.json` of Sapphire repo before Step 5 |
| TimescaleDB version and continuous aggregate syntax | Low — syntax differs between TimescaleDB 2.x versions | Verify version in existing pipeline configs |
