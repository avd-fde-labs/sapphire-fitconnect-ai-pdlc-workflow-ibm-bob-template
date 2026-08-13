# Feature Specification: Add Support for Body Temperature Metric Ingestion, Storage, and Reporting

**Feature Branch**: `MVWICA-8`
**Created**: 2026-08-12
**Status**: Draft
**Input**: STORY_ID=MVWICA-8

---

## Codebase Snapshot

**Affected repos (candidates)**: sapphire-event-ingestion-api, sapphire-kafka-pipeline, sapphire-charting-api, sapphire-bff-api, Sapphire, sapphire-kafka-streams-consumer

**Key existing artifacts observed** (from AGENTS.md knowledge; sibling repo filesystem access unavailable in this run):
- `sapphire-event-ingestion-api`: Python/FastAPI, Avro schemas, Kafka producer, OTLP tracing. Existing metrics (blood pressure, SpO2, activity) are ingested via REST endpoints and published to Kafka. Temperature will follow the same Avro schema and REST pattern.
- `sapphire-kafka-pipeline`: Kafka Connect JSON configuration. Existing sink connectors stream health metric topics to PostgreSQL hypertables. A new connector configuration is needed for the temperature topic.
- `sapphire-charting-api`: Java 17/Spring Boot. Provides charting and timeseries visualization endpoints consumed by the BFF. Existing chart types cover existing metrics — a temperature chart type must be added.
- `sapphire-bff-api`: Node.js/Apollo Server. GraphQL schema aggregates backend REST APIs. Existing queries expose health charts and metrics — temperature chart/metrics queries will be added following the same resolver pattern.
- `Sapphire`: TypeScript/React/Vite, Apollo GraphQL client, health dashboard. Existing metric chart components exist for other vital signs — a temperature chart component and metrics-list entry will be added to match.
- `sapphire-kafka-streams-consumer`: Java 17/Spring Boot/Kafka Streams. Generates near-real-time alerts by consuming Kafka topics. Currently handles existing metrics — temperature threshold alerting is in-scope but lower priority.

**Conventions to follow**:
- `sapphire-event-ingestion-api`: Pydantic v2 schemas, async FastAPI handlers, `httpx` for HTTP, `structlog` JSON logging, Avro for Kafka events
- `sapphire-kafka-pipeline`: JSON Kafka Connect sink connector configuration per topic
- `sapphire-charting-api`: Spring Controller→Service→Repository layering, `record` DTOs, constructor injection
- `sapphire-bff-api`: JWT-validated resolvers, `DataLoader` for N+1 prevention, typed service clients
- `Sapphire`: Functional React components, Apollo explicit cache policies, co-located feature directories, design token system, shared notification component

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Smart Device Temperature Ingestion (Priority: P1)

A user with a Bluetooth or internet-connected thermometer (e.g., a smart patch or wearable) has their device submit body temperature readings to the platform automatically. The platform accepts each reading — whether submitted individually or in a batch — validates that the value falls within a physiologically acceptable range, converts it to a canonical unit for storage, and records it against the user's account with the device identifier and timestamp. Invalid or out-of-range readings are rejected with a clear error.

**Why this priority**: Without the ability to ingest temperature data, none of the downstream storage, charting, or UI features can function. This is the foundational capability from which all other stories derive their data. Completing it alone provides integration value to device partners even before the UI is ready.

**Independent Test**: Submit a valid single temperature reading via the ingestion API for a known test user and device. Verify the event appears on the correct Kafka topic with the correct Avro-encoded fields. Then submit an out-of-range value and confirm a 422 rejection response with a descriptive error message. Submit a valid batch of five readings and confirm all five appear on the topic.

**Acceptance Scenarios**:
1. **Given** a registered user and a connected device, **When** the device posts a single valid temperature reading (Celsius or Fahrenheit, within physiological range) to the ingestion endpoint, **Then** the system returns HTTP 200, the event is published to the health-metrics Kafka topic with fields: user ID, timestamp, value, unit, device source, ingestion source, and measurement method (if provided).
2. **Given** a connected device, **When** the device posts a batch of up to 100 temperature readings, **Then** each valid reading is published as a separate event on the Kafka topic, and the response contains a per-record result summary.
3. **Given** a temperature value outside the configurable physiological range (e.g., below 25 °C or above 45 °C), **When** the ingestion endpoint receives the reading, **Then** the system returns HTTP 422 with an error message stating the value is out of the acceptable range and the configured bounds.
4. **Given** a Fahrenheit temperature reading, **When** the ingestion endpoint receives it, **Then** the event is stored with the original value and unit preserved, and the canonical storage unit is also recorded.
5. **Given** a request missing required fields (user ID, timestamp, or value), **When** the ingestion endpoint receives it, **Then** the system returns HTTP 422 with a field-level validation error listing each missing field.

---

### User Story 2 - Temperature Data Persisted to Timeseries Store (Priority: P2)

Once temperature events are on the Kafka topic, the platform's data pipeline automatically picks them up and writes each record into the timeseries datastore, where they are indexed for fast time-range queries. The stored records are compatible with the platform's existing retention and aggregation rules. Daily, weekly, and monthly rollup summaries are generated alongside the raw records.

**Why this priority**: Storage is the prerequisite for all charting and reporting. Without persisted records, the charting and UI stories cannot be validated end-to-end. This story is ranked P2 because it depends on P1 but is itself a dependency of all downstream stories.

**Independent Test**: After publishing a set of synthetic temperature events to the Kafka topic (manually or via the ingestion API from Story 1), verify that the records appear in the timeseries datastore within an acceptable lag window. Query the datastore directly for the test user to confirm all fields are present and indexed. Verify that rollup aggregates (daily min/max/avg) are computed.

**Acceptance Scenarios**:
1. **Given** a valid temperature event on the Kafka topic, **When** the pipeline processes it, **Then** the record is written to the timeseries datastore within 10 seconds, containing all fields: user ID, value, unit, timestamp, device source, ingestion source, and measurement method.
2. **Given** temperature records spanning multiple days for a user, **When** the aggregation process runs, **Then** daily rollup records exist containing the minimum, maximum, and average temperature for each day.
3. **Given** the existing retention policy for health metrics, **When** temperature records age past the retention threshold, **Then** they are subject to the same retention and archival rules as other metric types — no separate policy is required.
4. **Given** a burst of 1,000 temperature events within one minute, **When** the pipeline processes them, **Then** all 1,000 records are correctly persisted without data loss or duplication.

---

### User Story 3 - Temperature Trend Charts and Reporting (Priority: P3)

A user can retrieve trend data for their body temperature across selectable time ranges (day, week, month). The platform returns aggregated data points — minimum, maximum, and average — that can be visualised as a line or range chart. Users or partner systems can filter the data by date range and by source device. Temperature metrics are available in analytics export endpoints alongside other metric types.

**Why this priority**: Charting provides the first user-visible value from the stored data. It is ranked P3 because it requires both P1 and P2 to be complete, but it is prioritised above the UI so that the chart API can be validated independently before the front-end consumes it.

**Independent Test**: Call the charting service directly with a test user ID and a one-week date range. Verify the response includes data points for each day with min, max, and average values. Filter by a specific device source and confirm only matching records are returned. Call the analytics export endpoint and confirm temperature records are present in the export payload.

**Acceptance Scenarios**:
1. **Given** a user with stored temperature records, **When** the charting endpoint is called with a date range (day/week/month), **Then** the response contains aggregated temperature data points: timestamp, min value, max value, average value, and unit, for each interval within the range.
2. **Given** a user with data from multiple devices, **When** the charting endpoint is called with a device-source filter, **Then** only records from the specified device are included in the response.
3. **Given** a date range with no temperature records for the user, **When** the charting endpoint is called, **Then** the response returns an empty data array (not an error), with metadata indicating the queried range and unit.
4. **Given** an analytics export request, **When** temperature metrics exist for the requested period, **Then** the export payload includes temperature records with the same schema as other exported metrics.

---

### User Story 4 - Temperature Metric in the Health Dashboard UI (Priority: P4)

A logged-in user opens their health dashboard and sees body temperature listed alongside other health metrics. They can select the temperature metric to view a trend chart with a selectable time range (day, week, month). The chart displays the values in the user's preferred unit (Celsius or Fahrenheit), with a clearly visible unit label and the ability to toggle between units. Loading and empty states are handled gracefully.

**Why this priority**: The UI is the final delivery layer that makes the feature visible to end users. It depends on all previous stories being complete and is ranked P4 as a result. Once the chart API (P3) is validated, the UI can be built and tested against it independently.

**Independent Test**: Log in as a test user with pre-seeded temperature data. Navigate to the health dashboard. Confirm that body temperature appears in the metrics list. Select it and verify the chart renders with correct data points for the default time range. Switch between day, week, and month views. Toggle the unit between Celsius and Fahrenheit and verify values update. Verify loading skeleton appears during data fetch and an appropriate empty-state message is shown for a user with no temperature data.

**Acceptance Scenarios**:
1. **Given** a logged-in user on the health dashboard, **When** the metrics list loads, **Then** "Body Temperature" appears as a selectable metric alongside other vital signs.
2. **Given** a user selects Body Temperature, **When** the chart view opens, **Then** a trend chart is displayed showing temperature over the default time range (day), with the user's preferred unit labelled on the axis.
3. **Given** the chart is open, **When** the user selects "Week" or "Month" range, **Then** the chart updates to show aggregated data for that range without a full page reload, within 2 seconds.
4. **Given** the chart is open in Celsius mode, **When** the user toggles to Fahrenheit, **Then** all displayed values update immediately to Fahrenheit equivalents, and the unit label updates accordingly.
5. **Given** a user with no temperature records, **When** the temperature chart view loads, **Then** an informative empty-state message is displayed (e.g., "No temperature data available. Connect a compatible device to begin tracking.").
6. **Given** the temperature chart is loading data, **When** the network request is in-flight, **Then** a skeleton screen matching the chart dimensions is displayed, with no partial or undefined values visible.
7. **Given** the chart data request fails, **When** the error is returned, **Then** an error boundary is displayed with a "Try again" action, and the error is not exposed as a stack trace or internal code.

---

### User Story 5 - Temperature Threshold Alerting (Priority: P5)

The platform monitors ingested temperature readings in near-real-time and generates an alert when a reading exceeds a configurable high or low threshold (e.g., fever or hypothermia boundary). Alerts are delivered to the user via the existing notification channel.

**Why this priority**: Alerting adds proactive health value and reuses the existing Kafka Streams alert pipeline. It is the lowest priority because it is an additive capability that does not block any other story, and the threshold configuration mechanism may need separate governance review.

**Independent Test**: Configure a low threshold (e.g., 37.5 °C = alert) for a test user. Submit a temperature reading above that threshold via the ingestion API. Confirm an alert record is created and delivered via the WebSocket notification channel within 15 seconds.

**Acceptance Scenarios**:
1. **Given** a configurable fever threshold is set for the platform, **When** an ingested temperature reading exceeds the threshold, **Then** an alert is generated, associated with the user and the reading timestamp, and published to the alert Kafka topic.
2. **Given** an alert on the alert topic, **When** the notification service processes it, **Then** the user receives a real-time notification via WebSocket with the temperature value, threshold breached, and timestamp.
3. **Given** a temperature reading that is within the normal configured range, **When** it is ingested, **Then** no alert is generated.

---

### Edge Cases

- A device submits a temperature reading with a timestamp in the past (e.g., readings buffered offline). The system must accept the historical timestamp and store it accurately rather than replacing it with the ingestion time.
- A device submits a temperature reading with a future timestamp (clock skew). The system must reject readings with timestamps more than 5 minutes in the future with a 422 error.
- A batch submission contains a mix of valid and invalid records. The system must process and persist all valid records while returning per-record errors for invalid ones — a partial batch must not be treated as a complete failure.
- Simultaneous submissions from the same user on different devices must not cause duplicate records; each must be stored independently with its device-source identifier.
- The configurable physiological range is updated by an operator while the system is running. Subsequent submissions must use the new range without requiring a service restart.
- A user requests temperature data for a date range that partially overlaps with available data (e.g., one month range where only two weeks have data). The response must return only the intervals with data and indicate the gap rather than returning zeros.
- Unit toggling in the UI: rounding differences between Celsius and Fahrenheit conversions must not cause values to drift visibly (e.g., 37.0 °C → 98.6 °F → 37.0 °C round-trip must be stable).
- A user with no preferred unit set sees a sensible default (platform default unit) without an error.

---

## Requirements *(mandatory)*

### Functional Requirements

**Data Ingestion**
- **FR-001**: The system MUST accept body temperature readings submitted by integrated smart devices and third-party health APIs via a REST endpoint.
- **FR-002**: The ingestion endpoint MUST support both Celsius and Fahrenheit input units and preserve the submitted unit in the stored record.
- **FR-003**: The system MUST validate each temperature value against a configurable physiological range. Values outside the range MUST be rejected with a descriptive error message stating the configured bounds.
- **FR-004**: Each temperature record MUST be associated with: user identifier, measurement timestamp, value, unit, device source identifier, and ingestion source. Measurement method is optional.
- **FR-005**: The ingestion endpoint MUST support both single-record and batch (up to 100 records per request) submission modes.
- **FR-006**: In batch mode, partial failures MUST be handled gracefully: valid records in the batch MUST be processed and a per-record result summary returned.
- **FR-007**: Readings with timestamps more than 5 minutes in the future MUST be rejected with HTTP 422.
- **FR-008**: Historical readings (past timestamps) MUST be accepted and stored with the original measurement timestamp.

**Data Model / Schema**
- **FR-009**: A temperature metric type MUST be added to the existing health metrics catalog, consistent with the schema of existing metric types (blood pressure, SpO2, activity).
- **FR-010**: The temperature event schema MUST define: value (numeric), unit (Celsius/Fahrenheit), timestamp (ISO-8601), device source, ingestion source, and measurement method (optional).
- **FR-011**: The schema MUST be compatible with existing downstream analytics pipelines and the timeseries datastore.
- **FR-012**: All API schema changes MUST be reflected in updated internal and external API documentation delivered as part of this story.

**Storage and Processing**
- **FR-013**: Temperature records MUST be persisted in the timeseries datastore with indexing on user identifier and timestamp to support efficient time-range queries.
- **FR-014**: Temperature data MUST be subject to the same retention and aggregation rules as other health metric types — no separate retention policy is introduced.
- **FR-015**: Daily, weekly, and monthly rollup aggregates (minimum, maximum, average) MUST be computed for stored temperature records.

**Reporting and Charting**
- **FR-016**: A charting endpoint MUST return temperature trend data for a specified user and time range (day, week, month), providing minimum, maximum, and average values per interval.
- **FR-017**: The charting endpoint MUST support filtering by date range and by device source.
- **FR-018**: When no data exists for a requested range, the charting endpoint MUST return an empty data array with metadata indicating the queried range and unit — not an error response.
- **FR-019**: Temperature metrics MUST be available in the platform's analytics export endpoints, with the same schema structure as other exported metric types.

**User Interface**
- **FR-020**: The health dashboard metrics list MUST display Body Temperature as a selectable metric entry.
- **FR-021**: Selecting the Body Temperature metric MUST open a trend chart view with a default time range of "day".
- **FR-022**: The chart MUST support selectable time ranges: day, week, and month. Switching ranges MUST update the chart without a full page reload.
- **FR-023**: The chart MUST display values in the user's preferred unit (Celsius or Fahrenheit), with the unit clearly labelled.
- **FR-024**: The user MUST be able to toggle the display unit between Celsius and Fahrenheit from within the chart view; all displayed values MUST update immediately upon toggle.
- **FR-025**: The chart component MUST implement loading skeleton, error boundary with "Try again" action, and empty-state views as per platform UX standards.

**Alerting**
- **FR-026**: The platform MUST generate an alert when a temperature reading exceeds a configurable high or low threshold.
- **FR-027**: Temperature threshold alerts MUST be published to the existing alert Kafka topic and delivered to the user via the existing WebSocket notification channel.

### Key Entities

- **Temperature Reading**: A single body temperature measurement event. Attributes: user ID, timestamp, value (numeric), unit (Celsius | Fahrenheit), device source ID, ingestion source, measurement method (optional). This is the atomic unit flowing from ingestion through to storage and charting.
- **Temperature Rollup**: An aggregated summary of temperature readings for a given user and time interval (day/week/month). Attributes: user ID, interval start, interval end, minimum value, maximum value, average value, unit, record count.
- **Temperature Threshold Configuration**: A platform-level (or future per-user) configuration defining the acceptable physiological range and alert thresholds. Attributes: metric type (temperature), low threshold, high threshold, unit, effective date.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A valid single temperature reading submitted via the ingestion endpoint appears in the timeseries datastore within 10 seconds of submission under normal operating conditions.
- **SC-002**: A batch of 100 temperature readings submitted in a single request is fully processed (all valid records persisted) within 30 seconds.
- **SC-003**: Invalid or out-of-range temperature readings are rejected 100% of the time with an appropriate error response; no invalid record reaches the datastore.
- **SC-004**: Temperature trend charts load and render within 2 seconds when a user selects or switches a time range on the health dashboard.
- **SC-005**: All five user stories are independently testable and pass their defined acceptance scenarios before the feature is considered complete.
- **SC-006**: Unit coverage meets the platform minimums: 80% line coverage for Java and Python services, 70% line coverage for TypeScript/React components.
- **SC-007**: API schema documentation for all changed endpoints is published and verified as accurate against the implemented contract before release.
- **SC-008**: A temperature threshold alert is generated and delivered to the user within 15 seconds of an out-of-threshold reading being ingested, under normal load.
- **SC-009**: The UI temperature chart component passes WCAG AA accessibility standards as verified by an automated accessibility audit.
- **SC-010**: No regressions are introduced to existing health metric types (blood pressure, SpO2, activity) as verified by the existing test suite passing without modification.

---

## Assumptions

- The existing Avro event schema for health metrics is extensible by adding a new metric type without requiring a breaking schema change or separate schema registry migration.
- The Kafka topic for health metrics is partitioned such that a new temperature metric type can share the same topic (or a dedicated topic following the same naming convention) without pipeline reconfiguration beyond adding a new sink connector.
- The configurable physiological range defaults are: 25 °C – 45 °C (77 °F – 113 °F), representing the outer bounds of survivable human body temperature; normal fever threshold is 38.0 °C (100.4 °F).
- The platform's existing unit preference system (used for other metrics) is sufficient to store and recall a per-user temperature unit preference (Celsius/Fahrenheit); no new preference model is required.
- The timeseries datastore (PostgreSQL with TimescaleDB hypertables) already has hypertable infrastructure for health metrics; adding temperature requires a new hypertable or a partition/column extension of the existing one, following the same pattern as existing metrics.
- The BFF's GraphQL schema can be extended with new temperature query types without requiring a full schema version bump; additive changes are backward-compatible.
- Celsius-to-Fahrenheit conversion on the UI will use the standard formula (°F = °C × 9/5 + 32) and round to one decimal place to avoid visible drift on round-trips.
- The analytics export endpoint already supports a pluggable metric type registry; adding temperature is a configuration/registration step, not a structural change to the export pipeline.
- `sapphire-kafka-streams-consumer` already has a pattern for configuring threshold rules per metric type; temperature alerting will follow that same configuration mechanism.
- Sibling repository source code was not accessible from the workspace during spec generation; all assumptions about existing patterns are based on AGENTS.md descriptions and must be validated during planning.
