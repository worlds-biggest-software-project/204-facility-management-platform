# Facility Management Platform — Phased Development Plan

> Project: 204-facility-management-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (full-stack) | Dashboard-heavy FM platform with real-time work order tracking; shared types between API and frontend; IoT webhook receivers work well with Node.js event loop |
| Framework | Next.js 15 (App Router) | SSR for dashboard pages; API routes for backend; React Server Components for data-heavy space heatmaps and work order lists |
| Database | PostgreSQL 16 | 60-70 normalised tables from data-model-suggestion-1; JSONB for flexible asset metadata; PostGIS for facility/building locations; TimescaleDB extension for IoT time-series |
| ORM | Drizzle ORM | Type-safe; handles complex FM joins (asset→work_order→technician→space); generates migrations |
| Time-series | TimescaleDB (PostgreSQL extension) | IoT sensor readings (temperature, humidity, occupancy, energy) — hypertables with automatic partitioning |
| Task queue | BullMQ + Redis | Async work order notifications, PM schedule generation, predictive maintenance scoring, energy anomaly detection |
| IoT ingestion | MQTT broker (EMQX) + REST webhooks | MQTT for BACnet/IoT sensor streams; REST webhooks for cloud-connected sensors |
| File storage | S3-compatible (MinIO) | Work order photos, floor plans, BIM files, documents |
| LLM integration | Anthropic TypeScript SDK (Claude) | Work order triage (NLP classification), conversational FM assistant, natural-language reporting |
| PDF/Reports | @react-pdf/renderer | Work order reports, PM compliance reports, energy audits |
| Auth | Better Auth | Multi-org with role-based access (admin, manager, technician, occupant, vendor); OAuth 2.0 for API |
| Maps/Floor plans | Leaflet + custom SVG floor plan renderer | Interactive floor plans with space heatmaps, asset pins, punch locations |
| Containerisation | Docker + docker-compose | PostgreSQL+TimescaleDB, Redis, EMQX, MinIO, Next.js app, BullMQ worker |
| Testing | Vitest + Playwright | Unit/integration (Vitest); E2E (Playwright) |
| Linting | Biome | Fast linter + formatter |
| Type checking | TypeScript strict mode | Enforced in CI |
| Package manager | pnpm | Workspace support |

### Project Structure

```
facility-management/
├── package.json
├── pnpm-workspace.yaml
├── docker-compose.yml
├── Dockerfile
├── packages/
│   ├── shared/
│   │   └── src/types/
│   │       ├── work-order.ts
│   │       ├── asset.ts
│   │       ├── space.ts
│   │       ├── pm-schedule.ts
│   │       ├── sensor.ts
│   │       └── energy.ts
│   └── web/
│       ├── drizzle.config.ts
│       ├── drizzle/migrations/
│       ├── src/
│       │   ├── app/
│       │   │   ├── dashboard/
│       │   │   ├── work-orders/
│       │   │   ├── assets/
│       │   │   ├── spaces/
│       │   │   ├── pm-schedules/
│       │   │   ├── occupant-portal/
│       │   │   ├── energy/
│       │   │   ├── settings/
│       │   │   └── api/v1/
│       │   ├── components/
│       │   │   ├── FloorPlanViewer.tsx
│       │   │   ├── WorkOrderForm.tsx
│       │   │   ├── AssetCard.tsx
│       │   │   ├── SpaceHeatmap.tsx
│       │   │   ├── PMScheduleGrid.tsx
│       │   │   └── EnergyChart.tsx
│       │   ├── db/
│       │   │   ├── schema.ts
│       │   │   └── client.ts
│       │   ├── services/
│       │   │   ├── work-order-manager.ts
│       │   │   ├── pm-scheduler.ts
│       │   │   ├── asset-tracker.ts
│       │   │   ├── space-manager.ts
│       │   │   ├── sensor-ingestion.ts
│       │   │   ├── energy-monitor.ts
│       │   │   ├── wo-triage.ts          # AI work order classification
│       │   │   ├── predictive-maint.ts   # AI failure prediction
│       │   │   └── fm-assistant.ts       # Conversational AI
│       │   ├── workers/
│       │   │   ├── notification.worker.ts
│       │   │   ├── pm-generation.worker.ts
│       │   │   ├── sensor.worker.ts
│       │   │   └── ai.worker.ts
│       │   └── lib/
│       │       ├── auth.ts
│       │       ├── mqtt.ts
│       │       ├── s3.ts
│       │       └── cobie.ts             # COBie import parser
│       └── tests/
├── mobile/                              # PWA for technicians
└── docs/
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database schema from data-model-suggestion-1 (organisations, users, facilities, buildings, floors, spaces, assets), authentication with FM-specific roles, and Docker environment. After this phase, users can create organisations, add facilities with buildings/floors/spaces, and register assets.

### Tasks

#### 1.1 — Project Scaffold and Configuration

**What**: Create pnpm workspace, Next.js app, environment config, Docker with PostgreSQL+TimescaleDB, Redis, EMQX, MinIO.

**Design**:

```typescript
const envSchema = z.object({
  DATABASE_URL: z.string().default("postgresql://fm:fm@localhost:5432/fm"),
  REDIS_URL: z.string().default("redis://localhost:6379"),
  MQTT_BROKER_URL: z.string().default("mqtt://localhost:1883"),
  S3_ENDPOINT: z.string().default("http://localhost:9000"),
  S3_BUCKET: z.string().default("fm-files"),
  S3_ACCESS_KEY: z.string().default("minioadmin"),
  S3_SECRET_KEY: z.string().default("minioadmin"),
  ANTHROPIC_API_KEY: z.string().optional(),
  SITE_URL: z.string().default("http://localhost:3000"),
});
```

**Testing**:
- Unit: config parses valid env
- Integration: `docker-compose up -d` → all 5 services healthy (postgres, redis, emqx, minio, app)

#### 1.2 — Database Schema — Core FM Tables

**What**: Implement organisations, users, roles, facilities, buildings, floors, spaces, and assets from data-model-suggestion-1.

**Design**:

COBie-aligned hierarchy: `facilities` → `buildings` → `floors` → `spaces`. Assets reference spaces and have lifecycle stage (ISO 55000: `acquisition`, `operation`, `maintenance`, `decommission`).

```typescript
export const assets = pgTable("assets", {
  id: uuid("id").defaultRandom().primaryKey(),
  orgId: uuid("org_id").notNull().references(() => organisations.id),
  spaceId: uuid("space_id").references(() => spaces.id),
  assetTypeId: uuid("asset_type_id").references(() => assetTypes.id),
  name: varchar("name", { length: 255 }).notNull(),
  assetTag: varchar("asset_tag", { length: 100 }),
  serialNumber: varchar("serial_number", { length: 100 }),
  manufacturer: varchar("manufacturer", { length: 255 }),
  model: varchar("model", { length: 255 }),
  installDate: date("install_date"),
  warrantyExpiry: date("warranty_expiry"),
  expectedLifeYears: integer("expected_life_years"),
  lifecycleStage: varchar("lifecycle_stage", { length: 30, enum: ["acquisition", "operation", "maintenance", "decommission"] }).default("operation"),
  status: varchar("status", { length: 30, enum: ["active", "inactive", "out_of_service", "decommissioned"] }).default("active"),
  omniclassCode: varchar("omniclass_code", { length: 30 }),
  ifcGuid: varchar("ifc_guid", { length: 50 }),
  metadata: jsonb("metadata").default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow(),
});
```

`asset_types` table with OmniClass Table 23 codes. `spaces` with BOMA measurement fields (`gross_area`, `usable_area`, `rentable_area`).

**Testing**:
- Integration: migrations apply → all tables created including TimescaleDB extension
- Integration: create facility → building → floor → space → asset → FK chain holds
- Unit: asset lifecycle_stage enum rejects invalid values

#### 1.3 — Authentication and Role-Based Access

**What**: Multi-org auth with FM-specific roles: admin, manager, technician, occupant, vendor.

**Design**:

Roles: `admin` (full access), `fm_manager` (work orders, assets, spaces, reports), `technician` (work order completion, asset updates), `occupant` (service requests, space booking), `vendor` (assigned work orders only). Permissions checked per route.

**Testing**:
- Unit: technician can update work order → allowed
- Unit: occupant cannot access asset registry → 403
- Unit: vendor sees only assigned work orders → filtered query

---

## Phase 2: Work Order Management

### Purpose
Implement the core CMMS workflow: work order creation, assignment, status tracking, and completion. This is the primary daily activity for FM teams.

### Tasks

#### 2.1 — Work Order CRUD and Lifecycle

**What**: Create, assign, update, and complete work orders with full audit trail.

**Design**:

State machine: `open` → `assigned` → `in_progress` → `on_hold` | `completed` → `closed` (or `cancelled`).

```typescript
interface WorkOrderCreate {
  title: string;
  description: string;
  priority: "low" | "medium" | "high" | "critical" | "emergency";
  category: "corrective" | "preventive" | "inspection" | "project" | "request";
  facilityId: string;
  spaceId?: string;
  assetId?: string;
  assignedToUserId?: string;
  dueDate?: string;
  estimatedDurationMinutes?: number;
  attachmentIds?: string[];
}
```

Sequential WO numbering per org (WO-00001, WO-00002, ...). Audit trail: every status change, assignment, and comment creates a `work_order_activity` record.

**Testing**:
- Unit: create WO → status = `open`, number assigned
- Unit: assign → status = `assigned`, technician notified
- Unit: complete → status = `completed`, actual_duration recorded
- Unit: close → status = `closed`, closed_at set
- Integration: full lifecycle → audit trail has correct entries
- E2E: create WO in UI → appears in list, assignment notification sent

#### 2.2 — Work Order Dashboard and Analytics

**What**: Dashboard with WO counts by status, priority heatmap, cycle time metrics, and PM compliance rate.

**Design**:

`/dashboard` — cards: Open WOs (count), Overdue (count, red), Avg Cycle Time (hours), PM Compliance (%). Charts: WOs by priority (bar), WOs by category (pie), cycle time trend (line, 30 days). Filterable by facility, building, technician.

**Testing**:
- E2E: 50 WOs in various states → dashboard shows correct counts
- E2E: filter by facility → only that facility's WOs counted
- Unit: PM compliance = (completed PMs / total due PMs) × 100

---

## Phase 3: Preventive Maintenance Scheduling

### Purpose
Schedule recurring maintenance tasks with time-based and meter-based triggers that auto-generate work orders. Reduces unplanned downtime.

### Tasks

#### 3.1 — PM Schedule Definition and Work Order Generation

**What**: Define PM schedules per asset with triggers; auto-generate work orders when due.

**Design**:

```typescript
interface PMScheduleCreate {
  name: string;
  assetId: string;
  triggerType: "time_based" | "meter_based";
  intervalDays?: number;          // for time-based
  meterThreshold?: number;        // for meter-based (e.g., every 500 operating hours)
  taskDescription: string;        // work instructions for the technician
  estimatedDurationMinutes: number;
  priority: "low" | "medium" | "high";
  assignedToUserId?: string;
  checklistItems?: string[];      // inspection checklist
  isActive: boolean;
}
```

BullMQ cron job: `pm-check` runs daily at 01:00 UTC. For each active PM schedule, checks if interval has elapsed since last generated WO. If due, creates a new `preventive` work order with the PM template's task description and checklist.

Meter-based: requires meter readings submitted via API/mobile. When cumulative reading exceeds threshold since last PM, WO auto-generated.

**Testing**:
- Unit: PM with 30-day interval, last WO 31 days ago → new WO generated
- Unit: PM with 30-day interval, last WO 20 days ago → no WO generated
- Unit: meter-based PM at 500 hours, current reading 520 → WO generated
- Integration: create PM schedule → wait for cron → WO appears in work order list
- E2E: define PM schedule in UI → WO auto-created on schedule

---

## Phase 4: Asset Registry and Lifecycle Tracking

### Purpose
Comprehensive asset inventory with maintenance history, warranty tracking, and lifecycle cost analysis. Enables informed replacement and budget decisions.

### Tasks

#### 4.1 — Asset CRUD with Maintenance History

**What**: Asset management with full maintenance history, warranty alerts, and lifecycle cost tracking.

**Design**:

```typescript
// GET /api/v1/assets/{id}
// Response includes: asset details, maintenance_history (all WOs for this asset),
//   total_maintenance_cost_cents, warranty_status (active/expired/expiring_soon),
//   age_years, expected_remaining_life_years, lifecycle_stage

// GET /api/v1/assets?facility_id=...&lifecycle_stage=operation&sort=age:desc
// Paginated asset list with filters
```

Warranty alerting: BullMQ job checks assets with `warranty_expiry` within 90/60/30 days; creates `ai_suggestion` of type `warranty_expiring`.

**Testing**:
- Unit: asset with 5 completed WOs → maintenance history returns 5 records
- Unit: warranty expiring in 25 days → alert generated
- Unit: total maintenance cost computed from all WO actual_cost_cents
- E2E: asset detail page → shows maintenance history timeline, warranty badge

#### 4.2 — COBie Data Import

**What**: Import asset and space data from COBie spreadsheets for construction-to-FM handover.

**Design**:

```typescript
// POST /api/v1/import/cobie
// Request: multipart form with .xlsx file
// COBie worksheets mapped:
//   Facility → facilities table
//   Floor → floors table
//   Space → spaces table
//   Type → asset_types table
//   Component → assets table
//   System → systems table
//   Document → documents table
```

Parse with SheetJS (xlsx library). Map COBie column names to database fields. Validation: required fields present, FK references resolvable, area values numeric.

**Testing**:
- Unit: valid COBie spreadsheet → 1 facility, 3 floors, 10 spaces, 25 assets created
- Unit: COBie with missing required field → validation error listing missing fields
- Integration: upload COBie file → assets appear in registry with correct space assignments
- Fixture: `tests/fixtures/sample-cobie.xlsx`

---

## Phase 5: Occupant Service Request Portal

### Purpose
Self-service portal for building occupants to report issues, reducing phone calls and email tickets. Feeds directly into work order pipeline.

### Tasks

#### 5.1 — Occupant Portal and Service Request Flow

**What**: Web form for occupants to report issues with location, photo, and description; auto-creates work order.

**Design**:

```typescript
// POST /api/v1/service-requests (occupant-authenticated)
interface ServiceRequestCreate {
  category: "hvac" | "plumbing" | "electrical" | "janitorial" | "security" | "other";
  description: string;          // free-text issue description
  facilityId: string;
  spaceId?: string;             // optional: specific room
  urgency: "low" | "normal" | "urgent";
  photoIds?: string[];
}
// Auto-creates a work order of category="request" with the service request details
// Occupant receives email confirmation with WO number
// Occupant can check status via portal
```

Occupant portal: `/occupant-portal` — submit request form, view my requests with status, no access to full FM dashboard.

**Testing**:
- E2E: occupant submits request → WO created, confirmation email sent
- E2E: occupant views "My Requests" → sees status updates
- Unit: service request → WO with correct category mapping

---

## Phase 6: Space and Occupancy Analytics

### Purpose
Track space utilisation with heatmaps and analytics for hybrid work planning and real estate optimisation. Addresses the post-pandemic demand for data-driven space decisions.

### Tasks

#### 6.1 — Space Utilisation Dashboard

**What**: Heatmap visualisation of space utilisation across floors and buildings; occupancy trends over time.

**Design**:

Data sources: badge access logs, IoT occupancy sensors, desk booking records. Stored in `space_occupancy_readings` TimescaleDB hypertable.

```typescript
// TimescaleDB hypertable
CREATE TABLE space_occupancy_readings (
  time        TIMESTAMPTZ NOT NULL,
  space_id    UUID NOT NULL,
  occupant_count INT NOT NULL,
  capacity    INT NOT NULL,
  source      VARCHAR(30) NOT NULL  -- 'badge', 'sensor', 'booking'
);
SELECT create_hypertable('space_occupancy_readings', 'time');
```

Dashboard: SVG floor plan with rooms coloured by utilisation (green < 50%, yellow 50-80%, red > 80%). Time-series chart of occupancy by hour/day/week. Summary metrics: avg utilisation %, peak occupancy hours, underutilised spaces.

**Testing**:
- Unit: 24 hours of occupancy data → avg utilisation computed correctly
- E2E: floor plan heatmap → rooms coloured by utilisation level
- E2E: date range selector → chart updates

---

## Phase 7: IoT Sensor Integration

### Purpose
Ingest IoT sensor data (temperature, humidity, occupancy, energy) via MQTT and REST; trigger alerts and work orders from threshold breaches.

### Tasks

#### 7.1 — MQTT Sensor Ingestion

**What**: MQTT subscriber receiving sensor readings and storing in TimescaleDB; threshold-based alerting.

**Design**:

```typescript
// src/lib/mqtt.ts
// Subscribe to: sensors/{orgId}/{deviceId}/readings
// Payload: { "metric": "temperature", "value": 28.5, "unit": "celsius", "timestamp": "..." }

// Store in sensor_readings hypertable:
CREATE TABLE sensor_readings (
  time        TIMESTAMPTZ NOT NULL,
  device_id   UUID NOT NULL,
  metric      VARCHAR(50) NOT NULL,
  value       DOUBLE PRECISION NOT NULL,
  unit        VARCHAR(20) NOT NULL
);
SELECT create_hypertable('sensor_readings', 'time');
```

Threshold alerts: configurable per device per metric (e.g., temperature > 30°C → create critical WO for HVAC). Alert rules stored in `alert_rules` table.

**Testing**:
- Unit: MQTT message received → sensor_reading row created
- Unit: temperature 32°C with threshold 30°C → alert triggered, WO auto-created
- Unit: normal reading within threshold → no alert
- Integration: publish MQTT message → reading stored in TimescaleDB within 2s

#### 7.2 — Energy Management Dashboard

**What**: Energy consumption tracking with meter readings, anomaly detection, and ISO 50001-aligned reporting.

**Design**:

Energy meters as a special asset type with sub-meter hierarchy. `energy_readings` hypertable for consumption data. Dashboard: consumption by building/floor/system (line chart), cost breakdown (bar chart), anomaly flags.

Anomaly detection: rolling 7-day average per meter; readings > 2 standard deviations flagged.

**Testing**:
- Unit: energy spike 3σ above average → anomaly flagged
- E2E: energy dashboard → charts show consumption trends by building
- Unit: ISO 50001 baseline calculation correct for a 12-month period

---

## Phase 8: Technician Mobile PWA

### Purpose
Offline-capable mobile app for technicians to receive, update, and complete work orders in the field with photo capture.

### Tasks

#### 8.1 — PWA with Offline Work Order Management

**What**: Service worker caching; offline work order viewing, status updates, and photo capture; background sync.

**Design**:

Offline strategy: cache active work orders in IndexedDB. Status updates and photos queued locally. Background Sync API pushes changes when connectivity returns. Camera access for before/after photos attached to WOs.

**Testing**:
- E2E: airplane mode → view assigned WOs → update status → reconnect → changes synced
- E2E: take photo → attached to WO, uploaded on sync
- Unit: offline queue stores correct WO state changes

---

## Phase 9: AI-Powered Features (v1.1)

### Purpose
Add AI differentiators: automated work order triage from natural-language reports, predictive maintenance, and conversational FM assistant.

### Tasks

#### 9.1 — AI Work Order Triage

**What**: NLP classification of free-text fault reports into category, priority, and recommended technician.

**Design**:

```typescript
class WorkOrderTriageService {
  SYSTEM_PROMPT = `You are a facility management AI. Given a free-text maintenance request,
    classify it into: (1) category (hvac/plumbing/electrical/janitorial/security/other),
    (2) priority (low/medium/high/critical/emergency), (3) suggested trade/skill required.
    Also draft a concise work instruction for the assigned technician.
    Return JSON: {"category": "...", "priority": "...", "trade": "...", "work_instruction": "..."}`;

  async triage(description: string, facilityContext: FacilityContext): Promise<TriageResult> {
    // Call Claude with description + facility context (building type, active assets)
  }
}
```

Applied on service request submission: auto-fills WO fields, routes to appropriate technician based on trade and location.

**Testing**:
- Unit (mocked): "toilet won't stop running in 3rd floor bathroom" → category=plumbing, priority=medium
- Unit (mocked): "smoke detected in server room" → category=security, priority=emergency
- Integration: service request submitted → WO auto-triaged and assigned within 5s

#### 9.2 — Predictive Maintenance Scoring

**What**: Score assets by failure risk using age, maintenance history, and sensor data.

**Design**:

```typescript
interface PredictiveMaintenanceScore {
  assetId: string;
  riskScore: number;          // 0-100, higher = more likely to fail
  riskTier: "low" | "medium" | "high" | "critical";
  riskFactors: {
    factor: string;           // "age_exceeds_expected_life", "increasing_wo_frequency", "sensor_anomaly"
    contribution: number;     // 0-1
  }[];
  recommendedAction: string;
  estimatedFailureWindow: string;  // "2-4 weeks"
}
```

Scoring logic: weighted combination of age vs. expected life (30%), WO frequency trend (25%), sensor anomaly count (25%), time since last PM (20%). Runs weekly for all active assets via BullMQ job.

**Testing**:
- Unit: asset 2 years past expected life + increasing WO frequency → risk > 70
- Unit: new asset with no WOs → risk < 20
- Unit: asset with sensor anomalies in last 30 days → risk increased by anomaly factor
- Integration: weekly scoring → dashboard shows assets ranked by risk

#### 9.3 — Conversational FM Assistant

**What**: Natural-language assistant for occupants to report issues, check WO status, and book spaces via web/Slack/Teams.

**Design**:

```typescript
// POST /api/v1/ai/chat
// Request: { message: "The AC in conference room B on floor 2 isn't working" }
// Response: { reply: "I've created a work order (WO-00142) for HVAC repair...",
//             action: { type: "work_order_created", workOrderId: "..." } }
```

Assistant capabilities: create service request, check WO status by number, find available meeting rooms, report energy issue. Integrated with Slack/Teams via incoming webhook + bot framework.

**Testing**:
- Unit (mocked): "AC broken in room B" → service request created, WO number returned
- Unit (mocked): "What's the status of WO-00142?" → current status and assignment returned
- Unit (mocked): ambiguous message → clarifying question asked

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Work Order Management         ─── requires Phase 1
    │
Phase 3: Preventive Maintenance        ─── requires Phase 2
    │
Phase 4: Asset Registry & COBie        ─── requires Phase 1 (parallel with Phases 2-3)
    │
Phase 5: Occupant Service Portal       ─── requires Phase 2
    │
Phase 6: Space & Occupancy Analytics   ─── requires Phase 1 (parallel with Phases 2-5)
    │
Phase 7: IoT & Energy                  ─── requires Phase 1 (parallel with Phases 2-6)
    │
Phase 8: Mobile PWA                    ─── requires Phase 2
    │
Phase 9: AI Features                   ─── requires Phases 2 + 4 + 7
```

Parallelism opportunities:
- Phases 2, 4, 6, 7 can all begin concurrently after Phase 1
- Phase 3 requires Phase 2 (work order system must exist for PM to generate WOs)
- Phase 5 requires Phase 2 (service requests create WOs)
- Phase 8 requires Phase 2 (mobile app manages WOs)
- Phase 9 requires Phases 2 + 4 + 7 (AI uses WO data, asset data, and sensor data)

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specification.
2. All unit and integration tests pass (`vitest run`).
3. Biome linting passes with zero warnings.
4. TypeScript compiles in strict mode with zero errors.
5. Docker build succeeds.
6. `docker-compose up` brings all services to healthy state (including EMQX and TimescaleDB).
7. Feature works end-to-end (manual or Playwright E2E).
8. New API endpoints documented in auto-generated OpenAPI spec.
9. Database migrations created and tested.
10. Role-based permissions enforced on all new endpoints.
11. File uploads stored in S3 with correct content types.
12. IoT data stored in TimescaleDB hypertables (where applicable).
13. New environment variables documented in config.
