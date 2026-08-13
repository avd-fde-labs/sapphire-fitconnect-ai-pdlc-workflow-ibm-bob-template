# Implementation Queue — MVWICA-8

Generated: 2026-08-13

> Each entry is one `speckit.implement` invocation. Entries are processed in order.
> Tick `[x]` only when the corresponding invocation produces a `## Phase Complete` report.

## Queue

- [ ] sapphire-kafka-pipeline / Phase 1: Setup (Shared Infrastructure)
- [ ] sapphire-user-service / Phase 1: Setup (Shared Infrastructure)
- [ ] sapphire-event-ingestion-api / Phase 2: Foundational (Blocking Prerequisites)
- [ ] sapphire-kafka-pipeline / Phase 2: Foundational (Blocking Prerequisites)
- [ ] sapphire-event-ingestion-api / Phase 3: User Story 1 — Smart Device Temperature Ingestion (Priority: P1) 🎯 MVP
- [ ] sapphire-kafka-pipeline / Phase 4: User Story 2 — Temperature Data Persisted to Timeseries Store (Priority: P2)
- [ ] sapphire-charting-api / Phase 5: User Story 3 — Temperature Trend Charts and Reporting (Priority: P3)
- [ ] sapphire-user-service / Phase 6: User Story 4 — Temperature Metric in the Health Dashboard UI (Priority: P4)
- [ ] sapphire-bff-api / Phase 6: User Story 4 — Temperature Metric in the Health Dashboard UI (Priority: P4)
- [ ] Sapphire / Phase 6: User Story 4 — Temperature Metric in the Health Dashboard UI (Priority: P4)
- [ ] sapphire-kafka-streams-consumer / Phase 7: User Story 5 — Temperature Threshold Alerting (Priority: P5)
- [ ] sapphire-playwright / Phase 8: Polish & Cross-Cutting Concerns
- [ ] sapphire-event-ingestion-api / Phase 8: Polish & Cross-Cutting Concerns
- [ ] sapphire-charting-api / Phase 8: Polish & Cross-Cutting Concerns
- [ ] sapphire-kafka-streams-consumer / Phase 8: Polish & Cross-Cutting Concerns
- [ ] sapphire-bff-api / Phase 8: Polish & Cross-Cutting Concerns

## Dependency Notes

```
Phase 1 (Setup)       → no dependencies — start immediately
  sapphire-kafka-pipeline (T001–T003): topic, hypertable DDL, rollup DDL
  sapphire-user-service (T004): temperature_unit_preference column migration

Phase 2 (Foundational) → requires Phase 1
  sapphire-event-ingestion-api (T006): Avro schema registration
  sapphire-kafka-pipeline (T007–T008): sink connector deploy + smoke test

Phase 3 (US1) → requires Phase 2
  sapphire-event-ingestion-api (T009–T020): ingestion endpoint, models, service, tests

Phase 4 (US2) → requires Phase 3
  sapphire-kafka-pipeline (T021–T025): connector JSON, DDL verify, integration test

Phase 5 (US3) → requires Phase 4
  sapphire-charting-api (T026–T033): chart endpoint, repository, service, tests

Phase 6 (US4) → requires Phase 5; sub-tracks A/B/C run in parallel
  sapphire-user-service (T034–T036)    ← Sub-track A
  sapphire-bff-api (T037–T042)         ← Sub-track B
  Sapphire (T043–T049, T062)           ← Sub-track C (depends on A+B for full validation)

Phase 7 (US5) → independent of Phase 4–6; can run from Phase 2 onward
  sapphire-kafka-streams-consumer (T050–T055)

Phase 8 (Polish) → requires all user stories complete
  sapphire-playwright (T059–T060)
  sapphire-event-ingestion-api (T005, T056, T057, T058, T061)
  sapphire-charting-api (T005, T056, T057, T058, T061)
  sapphire-kafka-streams-consumer (T005, T056, T057, T058, T061)
  sapphire-bff-api (T005, T056, T057, T058, T061)
```

## Invocation Template

For each entry above, invoke:
```
/speckit.implement STORY_ID=MVWICA-8 REPO=<repo-name> PHASE=<exact phase label>
```

Phase labels must be copied verbatim from the phase headers in `specs/MVWICA-8/tasks.md`.

### Quick-reference invocations

```
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-kafka-pipeline PHASE="Phase 1: Setup (Shared Infrastructure)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-user-service PHASE="Phase 1: Setup (Shared Infrastructure)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-event-ingestion-api PHASE="Phase 2: Foundational (Blocking Prerequisites)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-kafka-pipeline PHASE="Phase 2: Foundational (Blocking Prerequisites)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-event-ingestion-api PHASE="Phase 3: User Story 1 — Smart Device Temperature Ingestion (Priority: P1) 🎯 MVP"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-kafka-pipeline PHASE="Phase 4: User Story 2 — Temperature Data Persisted to Timeseries Store (Priority: P2)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-charting-api PHASE="Phase 5: User Story 3 — Temperature Trend Charts and Reporting (Priority: P3)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-user-service PHASE="Phase 6: User Story 4 — Temperature Metric in the Health Dashboard UI (Priority: P4)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-bff-api PHASE="Phase 6: User Story 4 — Temperature Metric in the Health Dashboard UI (Priority: P4)"
/speckit.implement STORY_ID=MVWICA-8 REPO=Sapphire PHASE="Phase 6: User Story 4 — Temperature Metric in the Health Dashboard UI (Priority: P4)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-kafka-streams-consumer PHASE="Phase 7: User Story 5 — Temperature Threshold Alerting (Priority: P5)"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-playwright PHASE="Phase 8: Polish & Cross-Cutting Concerns"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-event-ingestion-api PHASE="Phase 8: Polish & Cross-Cutting Concerns"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-charting-api PHASE="Phase 8: Polish & Cross-Cutting Concerns"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-kafka-streams-consumer PHASE="Phase 8: Polish & Cross-Cutting Concerns"
/speckit.implement STORY_ID=MVWICA-8 REPO=sapphire-bff-api PHASE="Phase 8: Polish & Cross-Cutting Concerns"
```

Or skip the queue and implement all at once:
```
/speckit.implement STORY_ID=MVWICA-8
```

When all implementations are done, raise PRs with:
```
/speckit.ship STORY_ID=MVWICA-8
```
