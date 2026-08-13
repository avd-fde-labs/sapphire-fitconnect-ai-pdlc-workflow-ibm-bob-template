# Tasks: Add Support for Body Temperature Metric Ingestion, Storage, and Reporting

**Branch**: `MVWICA-8`
**Input**: Design documents from `specs/MVWICA-8/`
**Prerequisites**: plan.md ✅ | spec.md ✅ | research.md ✅ | data-model.md ✅ | contracts/ ✅ | quickstart.md ✅

**Child Jira stories**: MVWICA-9 (ingestion) · MVWICA-10 (pipeline) · MVWICA-11 (charting) · MVWICA-12 (bff) · MVWICA-13 (UI) · MVWICA-14 (alerts) · MVWICA-15 (user-service)

**Format**: `- [ ] [TaskID] [P?] [Story?] Description — repo: file path`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Environment variables, Kafka topic provisioning, and PostgreSQL DDL that must exist before any story can be validated end-to-end.

- [x] T001 Create Kafka topic `health-metrics.temperature` with appropriate partition count and retention — repo: `sapphire-kafka-pipeline` (ops/topic config or README)
- [x] T002 [P] Create `temperature_readings` TimescaleDB hypertable per DDL in `specs/MVWICA-8/data-model.md` — repo: `sapphire-kafka-pipeline` migration or init SQL
- [x] T003 [P] Create `temperature_daily_rollups` continuous aggregate per DDL in `specs/MVWICA-8/data-model.md` — repo: `sapphire-kafka-pipeline` migration or init SQL
- [x] T004 [P] Add `temperature_unit_preference` column to `users` table per DDL in `specs/MVWICA-8/data-model.md` — repo: `sapphire-user-service` (Flyway/Liquibase migration)
- [x] T005 [P] Add environment variables `TEMP_RANGE_MIN_C`, `TEMP_RANGE_MAX_C`, `TEMP_RANGE_MIN_F`, `TEMP_RANGE_MAX_F`, `TEMP_HIGH_ALERT_C`, `TEMP_LOW_ALERT_C` to local Docker Compose and service env templates — all affected repos

**Checkpoint**: Topic, hypertable, rollup aggregate, and user table column exist. Env vars defined. Proceed to Foundational.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Avro schema registration and Kafka Connect sink connector — must be in place before any ingestion event can reach PostgreSQL.

**⚠️ CRITICAL**: No user story work past US1 (ingestion) can be validated end-to-end until this phase is complete.

- [x] T006 Register `TemperatureReadingEvent` Avro schema in Confluent Schema Registry using `specs/MVWICA-8/contracts/kafka-avro-schema.json` — repo: `sapphire-event-ingestion-api` (schema registration script or README step)
- [x] T007 Deploy `temperature-sink` Kafka Connect connector using config in `specs/MVWICA-8/contracts/` (create `connectors/temperature-sink.json`) — repo: `sapphire-kafka-pipeline` `connectors/temperature-sink.json`
- [x] T008 [P] Verify end-to-end pipeline: publish test Avro event to topic, confirm record appears in `temperature_readings` within 10 s — repo: `sapphire-kafka-pipeline` (manual quickstart step 6)

**Checkpoint**: Avro schema registered. Connector deployed. Pipeline verified. All user stories can now proceed.

---

## Phase 3: User Story 1 — Smart Device Temperature Ingestion (Priority: P1) 🎯 MVP

**Goal**: `sapphire-event-ingestion-api` accepts single and batch temperature readings via API key auth, validates them, and publishes Avro events to `health-metrics.temperature`.

**Independent Test**: `POST /temperature-readings` with valid API key + valid Celsius reading → HTTP 200 + event on Kafka topic. `POST` with out-of-range value → HTTP 422. `POST` without key → HTTP 401. Batch of 5 → all 5 events on topic. 101st request in 1 min → HTTP 429 with `Retry-After`.

**Jira**: MVWICA-9

### Implementation for User Story 1

- [x] T009 [P] [US1] Create Pydantic v2 models `TemperatureReading` and `TemperatureBatchRequest` — repo: `sapphire-event-ingestion-api` `src/models/temperature.py`
- [x] T010 [P] [US1] Create `TemperatureConfig` dataclass (physiological range, rate limit) bound to env vars via Pydantic `BaseSettings` — repo: `sapphire-event-ingestion-api` `src/config/temperature_config.py`
- [x] T011 [P] [US1] Implement `ApiKeyValidator` FastAPI dependency: Redis hash lookup of hashed API key → user_id; return HTTP 401 if missing/invalid — repo: `sapphire-event-ingestion-api` `src/dependencies/api_key.py`
- [x] T012 [P] [US1] Implement `RateLimiter` FastAPI dependency: Redis sliding window (INCR+EXPIRE per minute bucket per key), raise HTTP 429 with `Retry-After` on breach — repo: `sapphire-event-ingestion-api` `src/dependencies/rate_limiter.py`
- [x] T013 [US1] Implement `TemperatureProducer`: Avro-encode `TemperatureReadingEvent` and publish to `health-metrics.temperature`; on broker failure catch `confluent_kafka.KafkaException` (or `confluent_kafka.KafkaError`) and re-raise as `KafkaUnavailableError`; bare `except:` is FORBIDDEN per constitution §I Python — repo: `sapphire-event-ingestion-api` `src/kafka/temperature_producer.py`
- [x] T014 [US1] Implement `TemperatureService`: validate value against configured range (FR-003), reject future timestamps >5 min (FR-007), accept historical timestamps (FR-008), call producer; emit `temperature_readings_ingested_total` and `temperature_readings_rejected_total` OTEL counters (FR-028/029) — repo: `sapphire-event-ingestion-api` `src/services/temperature_service.py`
- [x] T015 [US1] Implement `POST /temperature-readings` single-record endpoint: `Depends(ApiKeyValidator)`, `Depends(RateLimiter)`, delegate to `TemperatureService`; return HTTP 503 if `KafkaUnavailableError` raised (FR-008a) — repo: `sapphire-event-ingestion-api` `src/routers/temperature.py`
- [x] T016 [US1] Implement `POST /temperature-readings/batch` endpoint: iterate `TemperatureBatchRequest.readings`, call `TemperatureService` per record, collect per-record results (FR-005/006); return partial-success response — repo: `sapphire-event-ingestion-api` `src/routers/temperature.py`
- [x] T017 [US1] Register `temperature` router in app entry point — repo: `sapphire-event-ingestion-api` `src/main.py`
- [x] T018 [P] [US1] Unit tests for `TemperatureService`: valid reading, out-of-range Celsius, out-of-range Fahrenheit, future timestamp, historical timestamp, Kafka failure → 503 — repo: `sapphire-event-ingestion-api` `tests/unit/test_temperature_service.py`
- [x] T019 [P] [US1] Unit tests for `ApiKeyValidator` and `RateLimiter` dependencies — repo: `sapphire-event-ingestion-api` `tests/unit/test_dependencies.py`
- [x] T020 [P] [US1] Integration test (tagged `pytest.mark.integration`): full round-trip `POST /temperature-readings` → Kafka topic (Testcontainers Kafka) — repo: `sapphire-event-ingestion-api` `tests/integration/test_temperature_ingestion.py`

**Checkpoint**: US1 fully functional. Device submitters can authenticate and post temperature readings. Invalid submissions rejected. Events appear on Kafka topic.

---

## Phase 4: User Story 2 — Temperature Data Persisted to Timeseries Store (Priority: P2)

**Goal**: `sapphire-kafka-pipeline` Kafka Connect sink streams temperature events from topic to `temperature_readings` hypertable; `temperature_daily_rollups` continuous aggregate materialises daily min/max/avg.

**Independent Test**: Publish synthetic events to `health-metrics.temperature` topic; within 10 s records appear in `temperature_readings` with all fields correct. After rollup window, `temperature_daily_rollups` contains daily aggregates. Burst of 1,000 events processed without loss.

**Jira**: MVWICA-10

### Implementation for User Story 2

- [x] T021 [US2] Author `connectors/temperature-sink.json`: Kafka Connect JDBC sink config mapping `TemperatureReadingEvent` Avro fields to `temperature_readings` columns — repo: `sapphire-kafka-pipeline` `connectors/temperature-sink.json`
- [x] T022 [US2] Verify `temperature_readings` hypertable created by T002 is present; add connector field-mapping validation for all columns defined in `specs/MVWICA-8/data-model.md` (DDL ownership belongs to T002 in Phase 1) — repo: `sapphire-kafka-pipeline` `sql/001_temperature_schema.sql`
- [x] T023 [US2] Verify `temperature_daily_rollups` continuous aggregate and refresh policy created by T003 is present; confirm refresh interval (target: daily rollup lag ≤ 1 hour; weekly/monthly ≤ 4 hours) (DDL ownership belongs to T003 in Phase 1) — repo: `sapphire-kafka-pipeline` `sql/002_temperature_rollups.sql`
- [x] T024 [US2] Document connector deployment steps and field mapping in `specs/MVWICA-8/quickstart.md` Step 3 (update if needed)
- [x] T025 [P] [US2] Integration test (Docker Compose): publish 5 events to topic, assert all appear in `temperature_readings` within 10 s — repo: `sapphire-kafka-pipeline` `tests/integration/test_temperature_sink.py` (or shell/curl test)

**Checkpoint**: US2 complete. Temperature events flow from Kafka to PostgreSQL automatically. Rollup aggregates materialise on schedule.

---

## Phase 5: User Story 3 — Temperature Trend Charts and Reporting (Priority: P3)

**Goal**: `sapphire-charting-api` exposes `GET /charts/temperature` returning min/max/avg data points for day/week/month ranges with optional device-source filter and analytics export inclusion.

**Independent Test**: `GET /charts/temperature?userId=X&range=WEEK&from=…&to=…` returns data points with correct min/max/avg. With `deviceSourceId` filter only matching records returned. Empty range returns `dataPoints: []` with metadata, not an error. Temperature appears in analytics export endpoint.

**Jira**: MVWICA-11

### Implementation for User Story 3

- [x] T026 [P] [US3] Create `TemperatureChartRequest` and `TemperatureChartResponse` and `TemperatureDataPoint` Java record DTOs per `specs/MVWICA-8/data-model.md` — repo: `sapphire-charting-api` `src/main/java/.../temperature/dto/`
- [x] T027 [P] [US3] Create `ChartRange` enum (DAY, WEEK, MONTH) if not already present — repo: `sapphire-charting-api` `src/main/java/.../temperature/dto/ChartRange.java`
- [x] T028 [US3] Implement `TemperatureChartRepository`: `time_bucket` query against `temperature_readings` for DAY range; query against `temperature_daily_rollups` for WEEK/MONTH; optional `device_source_id` filter; return empty list when no data (FR-018) — repo: `sapphire-charting-api` `src/main/java/.../temperature/TemperatureChartRepository.java`
- [x] T029 [US3] Implement `TemperatureChartService`: delegate to repository, map results to `TemperatureChartResponse`; include temperature in analytics export registry (FR-019) — repo: `sapphire-charting-api` `src/main/java/.../temperature/TemperatureChartService.java`
- [x] T030 [US3] Implement `TemperatureChartController`: `GET /charts/temperature`, validate request params, delegate to service, return response — repo: `sapphire-charting-api` `src/main/java/.../temperature/TemperatureChartController.java`
- [x] T031 [P] [US3] `@WebMvcTest` for `TemperatureChartController`: valid request, empty-range response, missing-param 400 — repo: `sapphire-charting-api` `src/test/java/.../temperature/TemperatureChartControllerTest.java`
- [x] T032 [P] [US3] Unit test `TemperatureChartService` with mocked repository (80% line coverage, 100% service layer) — repo: `sapphire-charting-api` `src/test/java/.../temperature/TemperatureChartServiceTest.java`
- [x] T033 [P] [US3] Integration test (`@IntegrationTest`, Testcontainers PostgreSQL/TimescaleDB): seed data, call service, assert aggregates — repo: `sapphire-charting-api` `src/test/java/.../temperature/TemperatureChartRepositoryIntegrationTest.java`

**Checkpoint**: US3 complete. `GET /charts/temperature` returns correct aggregated trend data independently of the UI.

---

## Phase 6: User Story 4 — Temperature Metric in the Health Dashboard UI (Priority: P4)

**Purpose**: `sapphire-bff-api` adds GraphQL schema extensions; `sapphire-user-service` exposes unit preference; `Sapphire` UI renders the chart component. These three sub-tracks can run in parallel once US3 is complete.

**Independent Test**: Logged-in user navigates to health dashboard, selects Body Temperature, sees trend chart. Toggle unit updates all values. Range selector updates chart without page reload. Loading skeleton shows during fetch. Empty state shown for user with no data.

### Sub-track A: sapphire-user-service (Jira: MVWICA-15)

- [x] T034 [P] [US4] Expose `temperature_unit_preference` field on user profile GET response DTO — repo: `sapphire-user-service` (extend existing user profile record/DTO)
- [x] T035 [P] [US4] Add `PATCH /users/{id}/preferences` endpoint (or extend existing profile PATCH) to update `temperature_unit_preference`; validate enum (CELSIUS|FAHRENHEIT) — repo: `sapphire-user-service` (controller + service + repository)
- [x] T036 [P] [US4] Unit test `temperature_unit_preference` PATCH and GET (100% service layer) — repo: `sapphire-user-service` test class for preferences

### Sub-track B: sapphire-bff-api (Jira: MVWICA-12)

- [x] T037 [P] [US4] Add `temperature.graphql` SDL file with `TemperatureUnit`, `ChartRange`, `TemperatureDataPoint`, `TemperatureChartData` types; `temperatureChart` query; `updateTemperatureUnitPreference` mutation per `specs/MVWICA-8/contracts/bff-schema-additions.graphql` — repo: `sapphire-bff-api` `src/schema/temperature.graphql`
- [x] T038 [P] [US4] Implement `TemperatureDataSource`: typed HTTP client calling `GET /charts/temperature` on `sapphire-charting-api`; configured via environment `CHARTING_API_URL` — repo: `sapphire-bff-api` `src/datasources/TemperatureDataSource.ts`
- [x] T039 [US4] Implement `temperature.resolver.ts`: `temperatureChart` query validates JWT claims, extracts `userId`, delegates to `TemperatureDataSource`; `updateTemperatureUnitPreference` mutation calls `sapphire-user-service` PATCH — repo: `sapphire-bff-api` `src/resolvers/temperature.resolver.ts`
- [x] T040 [US4] Register `TemperatureDataSource` in Apollo Server context; register resolver — repo: `sapphire-bff-api` `src/server.ts` or equivalent
- [x] T041 [P] [US4] GraphQL schema contract test: assert no existing query is broken; `temperatureChart` and `updateTemperatureUnitPreference` present in introspection — repo: `sapphire-bff-api` `tests/contract/temperature.contract.test.ts`
- [x] T042 [P] [US4] Resolver unit test: `temperatureChart` with mocked `TemperatureDataSource` (valid JWT, missing JWT → error); `updateTemperatureUnitPreference` (success, invalid enum) — repo: `sapphire-bff-api` `tests/unit/temperature.resolver.test.ts`

### Sub-track C: Sapphire UI (Jira: MVWICA-13)

- [x] T043 [P] [US4] Create co-located feature directory `src/features/body-temperature/`; add `BodyTemperatureChart.types.ts` with `BodyTemperatureChartProps`, `TemperatureUnit`, `ChartRange` interfaces — repo: `Sapphire` `src/features/body-temperature/BodyTemperatureChart.types.ts`
- [x] T044 [P] [US4] Create `temperatureChart.graphql` Apollo query document for `temperatureChart` query and `updateTemperatureUnitPreference` mutation — repo: `Sapphire` `src/features/body-temperature/queries/temperatureChart.graphql`
- [x] T045 [US4] Implement `useBodyTemperature` custom hook: Apollo `useQuery` for `temperatureChart` with `cache-and-network` policy; read `temperature_unit_preference` from user profile; `useMutation` for `updateTemperatureUnitPreference`; encode selected range in URL query param (FR-022) — repo: `Sapphire` `src/features/body-temperature/useBodyTemperature.ts`
- [x] T046 [US4] Implement `BodyTemperatureChart` component: render loading skeleton matching chart dimensions during fetch; render error boundary with "Try again" action on failure; render empty-state message when `dataPoints` is empty; render trend chart with min/max/avg data points and unit label; time-range selector (DAY/WEEK/MONTH) updates URL param; Celsius/Fahrenheit toggle calls `updateTemperatureUnitPreference` mutation and updates display immediately (FR-020 through FR-025) — repo: `Sapphire` `src/features/body-temperature/BodyTemperatureChart.tsx`
- [x] T047 [US4] Add Body Temperature entry to the health dashboard metrics list; wire to `BodyTemperatureChart` route/view — repo: `Sapphire` (existing metrics list component)
- [x] T048 [P] [US4] RTL unit tests for `BodyTemperatureChart`: loading skeleton, error boundary with retry, empty state, chart with data, unit toggle fires mutation (70% coverage minimum) — repo: `Sapphire` `src/features/body-temperature/BodyTemperatureChart.test.tsx`
- [x] T049 [P] [US4] RTL unit tests for `useBodyTemperature` hook: loading, error, data, unit toggle, range change updates URL — repo: `Sapphire` `src/features/body-temperature/useBodyTemperature.test.ts`

**Checkpoint**: US4 complete. End users can see Body Temperature on the dashboard, view trend charts, switch time ranges, and toggle units. Preference persists across sessions.

---

## Phase 7: User Story 5 — Temperature Threshold Alerting (Priority: P5)

**Goal**: `sapphire-kafka-streams-consumer` monitors `health-metrics.temperature` and publishes a `TemperatureAlertEvent` to the alert topic when a reading exceeds operator-configured high/low thresholds.

**Independent Test**: Configure `TEMP_HIGH_ALERT_C=37.5`. Submit reading of 38.0 °C via ingestion API. Within 15 s a `TemperatureAlertEvent` appears on the alert Kafka topic and the End User receives a WebSocket notification. A reading of 36.5 °C produces no alert.

**Jira**: MVWICA-14

### Implementation for User Story 5

- [x] T050 [P] [US5] Create `TemperatureThresholdConfig` `@ConfigurationProperties` class binding `TEMP_HIGH_ALERT_C`, `TEMP_LOW_ALERT_C` (Celsius; Fahrenheit equivalents derived at runtime) — repo: `sapphire-kafka-streams-consumer` `src/main/java/.../temperature/TemperatureThresholdConfig.java`
- [x] T051 [P] [US5] Create `TemperatureAlertEvent` record DTO per `specs/MVWICA-8/data-model.md` (alert_id, user_id, triggered_at, reading_value, reading_unit, threshold_type, threshold_value, device_source_id) — repo: `sapphire-kafka-streams-consumer` `src/main/java/.../temperature/dto/TemperatureAlertEvent.java`
- [x] T052 [US5] Implement `TemperatureAlertProcessor`: Kafka Streams topology consuming `health-metrics.temperature`; each consumed Kafka message MUST create a root OTEL span (W3C `traceparent` propagation, per constitution §IV); filter readings exceeding `highAlertThreshold` or below `lowAlertThreshold` (converting Fahrenheit inputs to Celsius for comparison); map to `TemperatureAlertEvent`; produce to existing alert topic; emit `temperature_alerts_generated_total` OTEL counter (FR-030) — repo: `sapphire-kafka-streams-consumer` `src/main/java/.../temperature/TemperatureAlertProcessor.java`
- [x] T053 [US5] Register `TemperatureAlertProcessor` topology in the Kafka Streams application startup — repo: `sapphire-kafka-streams-consumer` (existing `StreamsConfig` or application entry point)
- [x] T054 [P] [US5] Unit tests for `TemperatureAlertProcessor`: reading above high threshold → alert produced; reading below low threshold → alert produced; reading within range → no alert; Fahrenheit input converted correctly (80% line coverage, 100% processor class) — repo: `sapphire-kafka-streams-consumer` `src/test/java/.../temperature/TemperatureAlertProcessorTest.java`
- [x] T055 [P] [US5] Integration test (`@IntegrationTest`, Testcontainers Kafka): publish threshold-breaching reading, assert alert on alert topic within 15 s — repo: `sapphire-kafka-streams-consumer` `src/test/java/.../temperature/TemperatureAlertProcessorIntegrationTest.java`

**Checkpoint**: US5 complete. Platform operators can configure thresholds; users receive real-time alerts on fever or hypothermia boundary breaches.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, OTEL env var confirmation, regression protection, and E2E test coverage.

- [x] T056 [P] Verify `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, `OTEL_DEPLOYMENT_ENVIRONMENT` are present in Docker Compose configs for `sapphire-event-ingestion-api`, `sapphire-charting-api`, `sapphire-kafka-streams-consumer`, `sapphire-bff-api` — all affected repos
- [x] T057 [P] Update `specs/MVWICA-8/quickstart.md` if any steps changed during implementation (connector config path, env var names, port numbers)
- [x] T058 [P] Update OpenAPI docs `specs/MVWICA-8/contracts/ingestion-api.yaml` and `charting-api.yaml` if any endpoint signatures changed during implementation
- [x] T059 [P] Add E2E Playwright test: End User logs in, navigates to Body Temperature chart, views data, toggles unit — repo: `sapphire-playwright` (new test file)
- [x] T060 [P] Add E2E Playwright test: Unit preference persists across logout and re-login — repo: `sapphire-playwright`
- [x] T061 Run full regression: existing health metric types (blood pressure, SpO2, activity) must pass unchanged test suites (SC-010) — all affected repos
- [x] T062 [P] Run automated WCAG AA accessibility audit on `BodyTemperatureChart` component using `axe-core` (via `@axe-core/react` in RTL test or Playwright axe plugin in T059 E2E test); assert zero violations at AA level (SC-009) — repo: `Sapphire` `src/features/body-temperature/BodyTemperatureChart.test.tsx` (amend T048) and `sapphire-playwright` (amend T059)

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1 (Setup)         → no dependencies — start immediately
Phase 2 (Foundational)  → requires Phase 1
Phase 3 (US1)           → requires Phase 2
Phase 4 (US2)           → requires Phase 3 (needs events on Kafka topic)
Phase 5 (US3)           → requires Phase 4 (needs data in PostgreSQL)
Phase 6 (US4)           → requires Phase 5 (needs chart API)
  Sub-track A (user-service) → can start in parallel with sub-track B and C once Phase 5 done
  Sub-track B (bff)          → can start in parallel with A and C once Phase 5 done
  Sub-track C (Sapphire UI)  → can start in parallel with A and B once Phase 5 done
Phase 7 (US5)           → independent of Phase 4–6; can run in parallel from Phase 3 onward
Phase 8 (Polish)        → requires all user stories complete
```

### Within Each Phase

- `[P]` tasks have no shared file conflicts and can run in parallel
- Models/DTOs before services; services before endpoints/controllers
- Unit tests can be written alongside implementation (TDD encouraged)
- Integration tests run pre-merge only (not in validation CI)

### Parallel Execution Example: Phase 6 (US4)

```
After Phase 5 is complete, launch all three sub-tracks simultaneously:

  Sub-track A: T034, T035, T036  (sapphire-user-service)
  Sub-track B: T037 → T038 → T039 → T040, T041, T042  (sapphire-bff-api)
  Sub-track C: T043, T044 → T045 → T046 → T047, T048, T049  (Sapphire)

Sub-tracks A and B must both be complete before T045/T046 (UI hook + component)
can be fully validated against a running stack.
```

### Parallel Execution Example: Phase 7 (US5)

```
T050 and T051 can start as soon as Phase 2 is complete (independent of Phases 3–6):

  Parallel: T050, T051
  Then:     T052 (depends on T050 + T051)
  Then:     T053
  Parallel: T054, T055
```

---

## Implementation Strategy

### MVP Scope (US1 only — MVWICA-9)

1. Complete Phase 1 + Phase 2 (Setup + Foundational)
2. Complete Phase 3 (US1 — ingestion endpoint)
3. **STOP and VALIDATE**: Device submitters can post readings; events appear on Kafka topic; rejections work correctly
4. Demonstrate to integration partners

### Incremental Delivery

1. Phase 1 + 2 → Foundation ready
2. Phase 3 → US1 ✅ → Device integration MVP
3. Phase 4 → US2 ✅ → Data persisted to PostgreSQL
4. Phase 5 → US3 ✅ → Chart API usable
5. Phase 6 → US4 ✅ → End users see temperature on dashboard
6. Phase 7 → US5 ✅ → Alert delivery complete
7. Phase 8 → Polish + E2E ✅ → Production-ready

### Parallel Team Strategy (3 engineers)

- After Phase 2: Engineer A owns US1→US2→US3 (pipeline track); Engineer B owns US4 sub-tracks A+B; Engineer C owns US4 sub-track C + US5 (can start Kafka Streams work independently from Phase 2)

---

## Task-to-Repo Mapping

| Repo | Tasks | Jira |
|------|-------|------|
| `sapphire-event-ingestion-api` | T006, T009–T020 | MVWICA-9 |
| `sapphire-kafka-pipeline` | T001–T003, T007–T008, T021–T025 | MVWICA-10 |
| `sapphire-charting-api` | T026–T033 | MVWICA-11 |
| `sapphire-bff-api` | T037–T042 | MVWICA-12 |
| `Sapphire` | T043–T049 | MVWICA-13 |
| `sapphire-kafka-streams-consumer` | T050–T055 | MVWICA-14 |
| `sapphire-user-service` | T004, T034–T036 | MVWICA-15 |
| `sapphire-playwright` | T059–T060 | (E2E) |
| All repos | T005, T056–T058, T061 | (cross-cutting) |
