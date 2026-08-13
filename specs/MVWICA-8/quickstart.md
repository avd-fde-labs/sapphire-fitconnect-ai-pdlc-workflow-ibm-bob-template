# Quickstart: MVWICA-8 — Body Temperature Metric

**Branch**: `MVWICA-8` | **Date**: 2026-08-12

This guide describes how to run and verify the temperature metric feature locally end-to-end using Docker Compose.

---

## Prerequisites

- Docker Compose (existing Sapphire local stack running: PostgreSQL, Kafka, Redis, Keycloak)
- Python 3.11 + `uv` or `pip` (for ingestion API)
- Java 17 + Maven (for charting API and Kafka Streams consumer)
- Node.js 20 (for BFF)
- An API key provisioned for local testing (see Step 1)

---

## Step 1 — Provision a Test API Key

In `sapphire-event-ingestion-api`, set environment variables:

```bash
export TEMP_API_KEY_TEST="dev-test-key-123"
export TEMP_API_KEY_TEST_USER_ID="00000000-0000-0000-0000-000000000001"
```

The ingestion service reads these at startup and loads them into Redis.

---

## Step 2 — Start Infrastructure

```bash
docker compose up -d postgres kafka redis keycloak
```

---

## Step 3 — Run Kafka Connect Sink

Deploy the temperature sink connector:

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @specs/MVWICA-8/contracts/kafka-avro-schema.json
```

*(The connector config file is `specs/MVWICA-8/contracts/temperature-sink.json` — use that path instead)*

---

## Step 4 — Start the Ingestion API

```bash
cd sapphire-event-ingestion-api
uv run uvicorn src.main:app --port 8001 --reload
```

---

## Step 5 — Submit a Test Temperature Reading

```bash
curl -X POST http://localhost:8001/temperature-readings \
  -H "X-API-Key: dev-test-key-123" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "00000000-0000-0000-0000-000000000001",
    "measured_at": "2026-08-12T10:00:00Z",
    "value": 37.2,
    "unit": "CELSIUS",
    "device_source_id": "device-patch-001",
    "ingestion_source": "api"
  }'
```

**Expected response**: HTTP 200 with `{"status": "accepted", "event_id": "<uuid>"}`

---

## Step 6 — Verify Storage

Wait ~10 seconds, then query PostgreSQL:

```sql
SELECT user_id, measured_at, value, unit, device_source_id
FROM temperature_readings
WHERE user_id = '00000000-0000-0000-0000-000000000001'
ORDER BY measured_at DESC
LIMIT 5;
```

---

## Step 7 — Query the Chart API

```bash
cd sapphire-charting-api
./mvnw spring-boot:run -Dspring-boot.run.arguments="--server.port=8002"
```

```bash
curl "http://localhost:8002/charts/temperature?userId=00000000-0000-0000-0000-000000000001&range=DAY&from=2026-08-12&to=2026-08-12"
```

**Expected response**: JSON with `dataPoints` containing today's min/max/avg.

---

## Step 8 — Verify via BFF GraphQL

Start the BFF, then run in GraphQL Playground (`http://localhost:4000/graphql`):

```graphql
query {
  temperatureChart(
    userId: "00000000-0000-0000-0000-000000000001"
    range: DAY
    from: "2026-08-12"
    to: "2026-08-12"
    unit: CELSIUS
  ) {
    range
    unit
    dataPoints {
      timestamp
      minValue
      maxValue
      avgValue
    }
  }
}
```

---

## Step 9 — Verify Out-of-Range Rejection

```bash
curl -X POST http://localhost:8001/temperature-readings \
  -H "X-API-Key: dev-test-key-123" \
  -H "Content-Type: application/json" \
  -d '{"user_id":"00000000-0000-0000-0000-000000000001","measured_at":"2026-08-12T10:01:00Z","value":60.0,"unit":"CELSIUS","device_source_id":"device-patch-001","ingestion_source":"api"}'
```

**Expected response**: HTTP 422 with message indicating value exceeds configured range.

---

## Step 10 — Verify Rate Limiting

Run the ingestion request 101 times in rapid succession using a loop. The 101st request should return HTTP 429 with a `Retry-After` header.

---

## Running Tests

```bash
# Python ingestion API
cd sapphire-event-ingestion-api
uv run pytest tests/unit/ -v

# Java charting API
cd sapphire-charting-api
./mvnw test

# Java Kafka Streams consumer
cd sapphire-kafka-streams-consumer
./mvnw test

# BFF
cd sapphire-bff-api
npm test

# React UI
cd Sapphire
npm test -- --testPathPattern=body-temperature
```

Integration tests (require Docker Compose infrastructure):
```bash
# Python
uv run pytest tests/integration/ -m integration -v

# Java
./mvnw test -Dgroups=integration
```
