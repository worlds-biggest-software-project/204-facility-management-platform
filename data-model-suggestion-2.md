# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Facility Management Platform · Created: 2026-05-20

## Philosophy

This model treats every state change in the facility management system as an immutable event. The event store is the single source of truth. Read-optimised materialised views (projections) are rebuilt from events to serve the UI, reporting, and API layers. This is the CQRS (Command Query Responsibility Segregation) pattern: commands write events, queries read projections.

In facility management, this pattern has compelling justification. Regulatory audits (ISO 41001, OSHA) require complete, tamper-proof records of what happened to every asset, who performed each maintenance action, and when. Insurance claims need reconstruction of the exact sequence of events leading to a failure. Predictive maintenance AI benefits from a complete temporal record of every sensor reading, work order transition, and parts replacement — the event stream is the natural training dataset.

Real-world FM systems like IBM Maximo already maintain deep audit trails, but they bolt these onto a mutable relational model, creating inconsistencies. An event-sourced model makes the audit trail the primary data structure, not an afterthought.

**Best for:** Organisations requiring tamper-proof audit trails, regulatory compliance, AI-driven predictive analytics on historical patterns, and temporal queries ("what was the asset's condition on date X?").

**Trade-offs:**
- Pro: Complete, immutable audit trail satisfying ISO 41001, ISO 55001, and OSHA requirements
- Pro: Temporal queries are natural — replay events to reconstruct state at any point in time
- Pro: Event stream is the ideal input for AI/ML models (failure prediction, anomaly detection)
- Pro: Supports event replay for debugging, migration, and analytics
- Pro: Enables real-time event-driven integrations (webhooks, message queues)
- Con: Higher storage requirements (events accumulate indefinitely)
- Con: Read model staleness — projections may lag behind events by milliseconds to seconds
- Con: Schema evolution requires careful event versioning (upcasting)
- Con: More complex to implement correctly than a mutable relational model
- Con: Simple CRUD operations require more code (command handler → event → projection)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 41001 | FM governance events (audits, inspections, compliance checks) recorded as immutable events with full traceability |
| ISO 55000 / ISO 55001 | Asset lifecycle transitions modeled as events: `AssetInstalled`, `AssetDegraded`, `AssetDecommissioned` |
| COBie | Initial data import generates `FacilityImported`, `FloorCreated`, `SpaceCreated`, `AssetRegistered` events from COBie spreadsheets |
| OmniClass / UniFormat | Classification codes stored in `AssetRegistered` and `AssetTypeCreated` event payloads |
| BOMA 2017 | Space area measurements captured in `SpaceMeasured` events with BOMA-compliant fields |
| ISO 50001 | Energy readings captured as `EnergyReadingRecorded` events; baselines as `EnergyBaselineSet` events |
| Project Haystack | IoT point metadata embedded in `IoTPointRegistered` event payloads |
| OWASP API Security | Event store access controlled via JWT with per-tenant claims; events encrypted at rest |

---

## Event Store (Core)

```sql
-- The immutable event store — the single source of truth
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,          -- aggregate root ID (e.g., asset ID, work order ID)
    stream_type     VARCHAR(50) NOT NULL,   -- 'Asset', 'WorkOrder', 'Space', 'Site', 'Lease', 'IoTDevice'
    event_type      VARCHAR(100) NOT NULL,  -- e.g., 'WorkOrderCreated', 'AssetConditionUpdated'
    event_version   INTEGER NOT NULL,       -- monotonically increasing per stream
    org_id          UUID NOT NULL,          -- tenant isolation
    payload         JSONB NOT NULL,         -- event data
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- { "user_id": "...", "ip": "...", "correlation_id": "...", "causation_id": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Optimistic concurrency: unique constraint on (stream_id, event_version) prevents conflicts
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_org ON events(org_id, created_at);
CREATE INDEX idx_events_time ON events(created_at);

-- Outbox table for guaranteed event delivery to projections and external consumers
CREATE TABLE event_outbox (
    event_id        UUID PRIMARY KEY REFERENCES events(event_id),
    published       BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_unpublished ON event_outbox(created_at) WHERE published = false;
```

### Event Type Catalogue

The following event types define the vocabulary of the system. Each event type has a versioned JSON schema for its payload.

```
-- Asset Domain
AssetTypeCreated        { name, omniclass_code, manufacturer, model_number, specs }
AssetRegistered         { asset_tag, serial_number, asset_type_id, space_id, install_date, purchase_cost }
AssetMoved              { from_space_id, to_space_id, moved_by }
AssetConditionUpdated   { old_score, new_score, reason }
AssetLifecycleChanged   { old_stage, new_stage, reason }  -- ISO 55000 stages
AssetDecommissioned     { reason, disposal_method, disposal_date }

-- Work Order Domain
WorkOrderCreated        { title, description, wo_type, priority, asset_id, space_id, reported_by }
WorkOrderAssigned       { assigned_to, assigned_by, estimated_hours }
WorkOrderStarted        { started_by }
WorkOrderPaused         { reason }
WorkOrderCompleted      { actual_hours, resolution_code, resolution_notes, parts_used[] }
WorkOrderClosed         { closed_by }
WorkOrderCancelled      { reason, cancelled_by }
WorkOrderCommentAdded   { comment, is_internal, user_id }
WorkOrderTaskCompleted  { task_id, completed_by }
WorkOrderCostRecorded   { cost_type, amount, description }

-- PM Schedule Domain
PMScheduleCreated       { name, trigger_type, frequency_days, asset_ids[] }
PMScheduleTriggered     { asset_id, generated_wo_id }
PMScheduleUpdated       { old_frequency, new_frequency }

-- Space Domain
SiteCreated             { name, address, country_code, timezone }
BuildingCreated         { site_id, name, year_built, gross_area_sqm }
FloorCreated            { building_id, name, floor_number }
SpaceCreated            { floor_id, name, space_type, usable_area_sqm, capacity }
SpaceMeasured           { gross_area, usable_area, rentable_area }  -- BOMA
SpaceBookingCreated     { booked_by, start_time, end_time, attendee_count }
SpaceBookingCancelled   { cancelled_by, reason }
OccupancyRecorded       { occupant_count, source }

-- IoT Domain
IoTDeviceRegistered     { device_name, device_type, protocol, asset_id, space_id }
IoTPointRegistered      { device_id, point_name, point_type, unit, haystack_tags }
IoTReadingReceived      { point_id, value, quality, recorded_at }
IoTAlertTriggered       { point_id, alert_type, threshold, actual_value }

-- Energy Domain
EnergyMeterRegistered   { site_id, building_id, meter_type, unit }
EnergyReadingRecorded   { meter_id, value, reading_type }
EnergyBaselineSet       { site_id, baseline_year, annual_consumption, energy_star_score }
EnergyAnomalyDetected   { meter_id, expected_value, actual_value, deviation_pct }

-- Lease Domain
LeaseCreated            { building_id, landlord, lease_type, start_date, end_date, monthly_rent }
LeaseRenewed            { new_end_date, new_monthly_rent }
LeaseTerminated         { reason, effective_date }

-- Service Request Domain
ServiceRequestSubmitted { subject, description, category, space_id, requested_by }
ServiceRequestTriaged   { priority, ai_classification, work_order_id }
ServiceRequestResolved  { resolution, resolved_by }

-- Compliance Domain
ComplianceCheckScheduled  { obligation_name, regulation, frequency_days, site_id }
ComplianceCheckCompleted  { result, notes, inspector_id }
ComplianceCheckFailed     { findings, required_actions }

-- User Domain
UserCreated             { email, full_name, user_type, org_id }
UserRoleAssigned        { role_name, permissions }
UserDeactivated         { reason }
```

## Materialised Read Models (Projections)

These tables are rebuilt from events. They can be dropped and reconstructed at any time by replaying the event store.

### Current State Projections

```sql
-- Projection: current asset state (rebuilt from Asset* events)
CREATE TABLE v_assets (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    asset_type_id   UUID,
    space_id        UUID,
    name            VARCHAR(255) NOT NULL,
    asset_tag       VARCHAR(100),
    serial_number   VARCHAR(255),
    lifecycle_stage VARCHAR(30) NOT NULL,
    criticality     VARCHAR(20),
    condition_score INTEGER,
    install_date    DATE,
    warranty_end    DATE,
    purchase_cost   NUMERIC(12, 2),
    last_event_id   UUID NOT NULL,         -- track projection currency
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL       -- optimistic concurrency for projection updates
);

-- Projection: current work order state
CREATE TABLE v_work_orders (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    site_id         UUID NOT NULL,
    asset_id        UUID,
    space_id        UUID,
    wo_number       VARCHAR(30) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    wo_type         VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    reported_by     UUID,
    assigned_to     UUID,
    estimated_hours NUMERIC(6, 2),
    actual_hours    NUMERIC(6, 2),
    total_cost      NUMERIC(10, 2),
    due_date        TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    task_count      INTEGER DEFAULT 0,
    tasks_completed INTEGER DEFAULT 0,
    comment_count   INTEGER DEFAULT 0,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL
);

-- Projection: current space state with occupancy
CREATE TABLE v_spaces (
    id              UUID PRIMARY KEY,
    floor_id        UUID NOT NULL,
    building_id     UUID NOT NULL,
    site_id         UUID NOT NULL,
    org_id          UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    space_type      VARCHAR(50),
    usable_area_sqm NUMERIC(10, 2),
    capacity        INTEGER,
    is_bookable     BOOLEAN DEFAULT false,
    current_occupancy INTEGER DEFAULT 0,
    avg_utilisation_pct NUMERIC(5, 2),
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL
);

-- Projection: current site/building hierarchy
CREATE TABLE v_sites (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2),
    timezone        VARCHAR(50),
    building_count  INTEGER DEFAULT 0,
    total_area_sqm  NUMERIC(12, 2),
    active_wo_count INTEGER DEFAULT 0,
    overdue_wo_count INTEGER DEFAULT 0,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL
);

CREATE INDEX idx_v_assets_org ON v_assets(org_id);
CREATE INDEX idx_v_assets_space ON v_assets(space_id);
CREATE INDEX idx_v_wo_org_status ON v_work_orders(org_id, status);
CREATE INDEX idx_v_wo_assigned ON v_work_orders(assigned_to) WHERE status IN ('assigned', 'in_progress');
CREATE INDEX idx_v_spaces_building ON v_spaces(building_id);
```

### Analytics Projections

```sql
-- Projection: work order metrics (aggregated from events, rebuilt daily or on-demand)
CREATE TABLE v_wo_metrics_daily (
    org_id          UUID NOT NULL,
    site_id         UUID NOT NULL,
    metric_date     DATE NOT NULL,
    wo_created      INTEGER DEFAULT 0,
    wo_completed    INTEGER DEFAULT 0,
    wo_overdue      INTEGER DEFAULT 0,
    avg_completion_hours NUMERIC(8, 2),
    pm_compliance_pct NUMERIC(5, 2),
    total_cost      NUMERIC(12, 2),
    PRIMARY KEY (org_id, site_id, metric_date)
);

-- Projection: asset failure history (for predictive maintenance ML)
CREATE TABLE v_asset_failure_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL,
    asset_type_id   UUID NOT NULL,
    failure_event_id UUID NOT NULL,
    failure_date    TIMESTAMPTZ NOT NULL,
    age_at_failure_days INTEGER,
    condition_before INTEGER,
    failure_code    VARCHAR(50),
    resolution_code VARCHAR(50),
    repair_hours    NUMERIC(6, 2),
    repair_cost     NUMERIC(10, 2),
    parts_replaced  JSONB,
    sensor_readings_before JSONB  -- snapshot of IoT readings in the 24h before failure
);

-- Projection: energy consumption aggregates
CREATE TABLE v_energy_daily (
    org_id          UUID NOT NULL,
    site_id         UUID NOT NULL,
    meter_type      VARCHAR(30) NOT NULL,
    metric_date     DATE NOT NULL,
    total_consumption NUMERIC(14, 4),
    peak_demand     NUMERIC(14, 4),
    cost_estimate   NUMERIC(10, 2),
    PRIMARY KEY (org_id, site_id, meter_type, metric_date)
);

CREATE INDEX idx_failure_history_asset ON v_asset_failure_history(asset_id, failure_date);
CREATE INDEX idx_failure_history_type ON v_asset_failure_history(asset_type_id, failure_date);
```

## Snapshot Store (Performance Optimisation)

```sql
-- Snapshots reduce replay time for aggregates with many events
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    event_version   INTEGER NOT NULL,     -- version at which snapshot was taken
    state           JSONB NOT NULL,       -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, event_version)
);

-- Keep only the latest snapshot per stream (older ones can be pruned)
CREATE INDEX idx_snapshots_latest ON snapshots(stream_id, event_version DESC);
```

## Subscription & Projection Tracking

```sql
-- Track which events each projection has processed (catch-up subscriptions)
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Identity & Multi-Tenancy (Projection)

```sql
-- These are projections rebuilt from User* events
CREATE TABLE v_users (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    user_type       VARCHAR(30) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    roles           JSONB DEFAULT '[]',
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL
);

CREATE TABLE v_organisations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    parent_org_id   UUID,
    org_type        VARCHAR(50),
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         INTEGER NOT NULL
);

CREATE INDEX idx_v_users_org ON v_users(org_id);
CREATE INDEX idx_v_users_email ON v_users(email);
```

## Example: Temporal Query (Reconstruct Asset State at a Point in Time)

```sql
-- "What was the condition of asset X on January 15, 2026?"
-- Replay all events for the asset up to that date

SELECT
    e.event_type,
    e.payload,
    e.created_at
FROM events e
WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'  -- asset ID
  AND e.stream_type = 'Asset'
  AND e.created_at <= '2026-01-15T23:59:59Z'
ORDER BY e.event_version ASC;

-- Application code replays these events to reconstruct the asset's state
-- at exactly that point in time, including condition_score, lifecycle_stage,
-- location (space_id), and all maintenance performed up to that date.
```

## Example: Failure Pattern Detection Query

```sql
-- Find all assets of a given type that had condition score drop before failure
-- Used to train predictive maintenance models

SELECT
    fh.asset_id,
    fh.age_at_failure_days,
    fh.condition_before,
    fh.failure_code,
    fh.sensor_readings_before,
    at.name AS asset_type_name,
    at.manufacturer
FROM v_asset_failure_history fh
JOIN v_assets a ON a.id = fh.asset_id
JOIN events e ON e.event_id = fh.failure_event_id
CROSS JOIN LATERAL (
    SELECT payload->>'name' AS name, payload->>'manufacturer' AS manufacturer
    FROM events
    WHERE stream_id = fh.asset_type_id AND event_type = 'AssetTypeCreated'
    LIMIT 1
) at
WHERE fh.asset_type_id = '...'
  AND fh.failure_date >= now() - INTERVAL '2 years'
ORDER BY fh.failure_date DESC;
```

## Example: Event-Driven Work Order Flow

```sql
-- Step 1: Command handler writes events (application code)
-- INSERT INTO events (stream_id, stream_type, event_type, event_version, org_id, payload, metadata)
-- VALUES (
--   gen_random_uuid(),  -- new work order ID
--   'WorkOrder',
--   'WorkOrderCreated',
--   1,
--   :org_id,
--   '{"title": "AHU-3 vibration alarm", "wo_type": "corrective", "priority": "high",
--     "asset_id": "...", "reported_by": "..."}'::jsonb,
--   '{"user_id": "...", "correlation_id": "...", "ip": "10.0.1.42"}'::jsonb
-- );

-- Step 2: Projection handler updates v_work_orders (async, triggered by outbox)
-- Step 3: Analytics projection updates v_wo_metrics_daily
-- Step 4: Notification service sends push notification to assigned technician
-- Step 5: AI service reads event payload and auto-classifies priority
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, event_outbox |
| Snapshots | 1 | snapshots |
| Projection Tracking | 1 | projection_checkpoints |
| Current State Projections | 6 | v_assets, v_work_orders, v_spaces, v_sites, v_users, v_organisations |
| Analytics Projections | 3 | v_wo_metrics_daily, v_asset_failure_history, v_energy_daily |
| **Total** | **~13 tables** | Plus additional projections as needed; projections are disposable and rebuildable |

---

## Key Design Decisions

1. **Single `events` table with JSONB payload** rather than one table per event type. This keeps the event store simple and avoids schema migrations when new event types are added. The `event_type` column combined with `stream_type` provides sufficient discrimination for querying.

2. **Optimistic concurrency via `UNIQUE (stream_id, event_version)`** prevents conflicting writes to the same aggregate. Two concurrent commands targeting the same work order will fail with a unique constraint violation, and the loser retries.

3. **Transactional outbox pattern** ensures events are reliably delivered to projections and external consumers. The outbox poller reads unpublished events and pushes them to message queues (e.g., NATS, RabbitMQ) in order.

4. **Projections are prefixed with `v_`** to clearly distinguish them from the event store. Projections can be dropped and rebuilt from events at any time — they are derived data, not source-of-truth.

5. **Snapshot store for long-lived aggregates** (assets with thousands of events) allows state reconstruction without replaying the full event history. Snapshots are taken at intervals (e.g., every 100 events) and replay starts from the latest snapshot.

6. **IoT readings stored as events** (`IoTReadingReceived`) enables temporal analysis and correlation with maintenance events. However, high-frequency sensors (sub-second) may need a separate time-series store (TimescaleDB) with events sampled or summarised before entering the main event store.

7. **Asset failure history as a dedicated projection** creates a denormalised, ML-ready dataset that includes the asset's age at failure, condition score before failure, and a snapshot of sensor readings in the 24 hours preceding the failure. This is the primary training dataset for predictive maintenance models.

8. **Event versioning via `event_version` per stream** (not global sequence) keeps each aggregate's history self-contained. A global sequence can be added via a PostgreSQL `BIGSERIAL` if needed for ordered cross-aggregate replay.

9. **Metadata captures causation and correlation IDs** enabling distributed tracing across commands. When a PM schedule triggers a work order, the `causation_id` links back to the `PMScheduleTriggered` event, and the `correlation_id` groups the entire chain.

10. **Event type catalogue serves as the domain language** — the ~40 event types listed above define every possible state transition in the system. New features are added by defining new event types, not by altering existing tables. This makes the system extensible without schema migrations.
