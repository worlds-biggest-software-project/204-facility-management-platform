# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Facility Management Platform · Created: 2026-05-20

## Philosophy

This model uses a relational backbone for core entities (sites, assets, work orders) but leverages PostgreSQL JSONB columns extensively for attributes that vary by jurisdiction, asset type, building type, or tenant configuration. The core tables are lean — typically 8-15 strongly typed columns — with a `properties` or `metadata` JSONB column that absorbs domain-specific variability.

This pattern is inspired by how modern SaaS platforms like Facilio and ServiceNow handle multi-tenant configurability. A hospital's work order needs different fields than a commercial office's. An HVAC asset in Singapore requires different compliance fields than one in Germany. Rather than creating hundreds of nullable columns or dozens of lookup tables, the JSONB approach stores these variations in a single flexible column with GIN indexing for fast containment queries.

The hybrid approach preserves relational integrity where it matters (foreign keys between assets, spaces, and work orders) while enabling rapid feature development without schema migrations. New fields for a specific customer or jurisdiction are added to the JSONB column, not to the DDL.

**Best for:** Multi-tenant SaaS deployments serving diverse industries (offices, hospitals, campuses) across multiple jurisdictions, where tenant-specific customisation must be delivered without schema changes.

**Trade-offs:**
- Pro: Extremely fast to add new fields — no migrations, no downtime
- Pro: Tenant-specific and jurisdiction-specific fields coexist without schema bloat
- Pro: JSONB GIN indexes support efficient containment and existence queries
- Pro: Lower table count (~25-30) compared to fully normalised model
- Pro: Natural fit for IoT device metadata where sensor configurations vary widely
- Pro: Easy to support COBie import by mapping COBie fields into JSONB properties
- Con: JSONB fields lack CHECK constraints — application-layer validation required
- Con: No foreign key enforcement inside JSONB — referential integrity is partial
- Con: Complex aggregation queries on JSONB fields are slower than on indexed columns
- Con: Schema documentation requires discipline — JSONB contents not self-documenting in the DDL
- Con: ORM support for JSONB varies; some frameworks treat it as opaque

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| COBie | COBie worksheets map to core tables; COBie-specific attributes (spare worksheets, job details) stored in `properties` JSONB |
| ISO 55000 | Asset lifecycle stage is a typed column; asset-type-specific ISO 55000 attributes in `properties` |
| ISO 41001 | Compliance obligation details (jurisdiction-specific requirements) stored in `properties` JSONB |
| OmniClass / UniFormat | Classification codes as typed columns on asset types; extended classification metadata in JSONB |
| BOMA 2017 | Core BOMA areas as typed columns; extended measurement details in `properties` |
| ISO 50001 | Energy configuration varies by meter type; meter-specific calibration/configuration in JSONB |
| Project Haystack | Full Haystack tag sets stored as JSONB on IoT points — native fit for tag-based semantics |
| BACnet | BACnet object properties (vendor-specific extensions) stored in device `properties` JSONB |
| IFC4.3 | IFC GUIDs as typed columns; extended IFC property sets imported into `properties` JSONB |

---

## Core Identity & Configuration

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    parent_org_id   UUID REFERENCES organisations(id),
    org_type        VARCHAR(50) NOT NULL DEFAULT 'tenant',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "default_timezone": "America/New_York",
    --   "work_order_number_prefix": "WO",
    --   "wo_number_sequence": 1001,
    --   "enabled_modules": ["maintenance", "space", "energy", "iot"],
    --   "custom_fields": {
    --     "work_order": [
    --       {"key": "cost_center", "label": "Cost Center", "type": "string", "required": true},
    --       {"key": "gl_account", "label": "GL Account", "type": "string"}
    --     ],
    --     "asset": [
    --       {"key": "regulatory_class", "label": "Regulatory Class", "type": "select",
    --        "options": ["Class I", "Class II", "Class III"]}
    --     ]
    --   },
    --   "compliance_jurisdictions": ["US-NY", "US-CA", "SG"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    user_type       VARCHAR(30) NOT NULL,
    roles           JSONB NOT NULL DEFAULT '[]',
    -- roles example: ["admin", "site_manager:site-uuid-1", "technician:site-uuid-2"]
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences: { "timezone": "Asia/Singapore", "language": "en", "notifications": {...} }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

CREATE INDEX idx_users_org ON users(org_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_roles ON users USING GIN (roles);
```

## Spatial Hierarchy

```sql
CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2),
    timezone        VARCHAR(50),
    coordinates     POINT,                  -- PostGIS-compatible
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "address": { "line1": "123 Main St", "city": "New York", "state": "NY", "postal": "10001" },
    --   "site_type": "campus",
    --   "total_area_sqm": 45000,
    --   "operating_hours": { "mon-fri": "06:00-22:00", "sat": "08:00-18:00" },
    --   "emergency_contacts": [
    --     { "name": "Security", "phone": "+1-555-0100" },
    --     { "name": "Facilities", "phone": "+1-555-0200" }
    --   ],
    --   "certifications": ["LEED Gold", "ENERGY STAR"],
    --   "weather_station_id": "KJFK"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    parent_space_id UUID REFERENCES spaces(id),  -- self-referential hierarchy: building > floor > room
    name            VARCHAR(255) NOT NULL,
    space_level     VARCHAR(20) NOT NULL,    -- 'building', 'floor', 'room', 'zone', 'desk'
    space_type      VARCHAR(50),             -- 'office', 'meeting_room', 'lobby', 'server_room'
    capacity        INTEGER,
    ifc_guid        VARCHAR(64),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example (varies by space_level):
    -- Building: {
    --   "year_built": 2015, "floor_count": 12,
    --   "gross_area_sqm": 15000, "usable_area_sqm": 12500, "rentable_area_sqm": 13200,
    --   "building_code": "HQ-01", "construction_type": "steel_frame"
    -- }
    -- Floor: { "floor_number": 3, "elevation_m": 12.5, "gross_area_sqm": 1250 }
    -- Room: {
    --   "usable_area_sqm": 45.5, "is_bookable": true,
    --   "amenities": ["whiteboard", "video_conference", "phone"],
    --   "max_noise_db": 45, "lighting_lux": 500
    -- }
    -- Zone: { "zone_type": "hvac", "controlled_by": "ahu-3" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Materialised path for efficient subtree queries
-- e.g., '/site-uuid/building-uuid/floor-uuid/room-uuid'
ALTER TABLE spaces ADD COLUMN path TEXT;

CREATE INDEX idx_spaces_site ON spaces(site_id);
CREATE INDEX idx_spaces_parent ON spaces(parent_space_id);
CREATE INDEX idx_spaces_level ON spaces(space_level);
CREATE INDEX idx_spaces_type ON spaces(space_type);
CREATE INDEX idx_spaces_props ON spaces USING GIN (properties);
CREATE INDEX idx_spaces_path ON spaces(path text_pattern_ops);
CREATE INDEX idx_sites_org ON sites(org_id);
CREATE INDEX idx_sites_props ON sites USING GIN (properties);
```

## Asset Management

```sql
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    space_id        UUID REFERENCES spaces(id),
    parent_asset_id UUID REFERENCES assets(id),
    name            VARCHAR(255) NOT NULL,
    asset_tag       VARCHAR(100),
    serial_number   VARCHAR(255),
    category        VARCHAR(100) NOT NULL,     -- 'hvac', 'electrical', 'plumbing', 'elevator', 'fire'
    omniclass_code  VARCHAR(30),
    lifecycle_stage VARCHAR(30) NOT NULL DEFAULT 'operational',
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium',
    condition_score INTEGER CHECK (condition_score BETWEEN 1 AND 5),
    install_date    DATE,
    warranty_end    DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example (varies by category):
    -- HVAC: {
    --   "manufacturer": "Carrier", "model": "30XA-400", "capacity_tons": 400,
    --   "refrigerant": "R-410A", "rated_efficiency": 0.65,
    --   "purchase_cost": 85000, "replacement_cost": 92000,
    --   "expected_life_years": 20, "operating_hours": 45000,
    --   "last_refrigerant_charge_kg": 120,
    --   "compliance": {
    --     "epa_section_608": true, "ashrae_90_1": true,
    --     "next_inspection_date": "2026-09-15"
    --   }
    -- }
    -- Elevator: {
    --   "manufacturer": "Otis", "model": "Gen2", "capacity_kg": 1600,
    --   "floors_served": [1,2,3,4,5], "speed_mps": 2.5,
    --   "inspection_authority": "NY-DOB",
    --   "last_inspection": "2026-03-10", "certificate_number": "EL-2026-0042"
    -- }
    -- Fire: {
    --   "manufacturer": "Siemens", "model": "FC2060",
    --   "system_type": "fire_alarm_panel", "zones_covered": 12,
    --   "last_test_date": "2026-04-20", "nfpa_code": "NFPA 72"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_org ON assets(org_id);
CREATE INDEX idx_assets_space ON assets(space_id);
CREATE INDEX idx_assets_category ON assets(category);
CREATE INDEX idx_assets_tag ON assets(asset_tag);
CREATE INDEX idx_assets_lifecycle ON assets(lifecycle_stage);
CREATE INDEX idx_assets_props ON assets USING GIN (properties);
-- Partial index for warranty tracking
CREATE INDEX idx_assets_warranty ON assets(warranty_end) WHERE warranty_end IS NOT NULL AND status = 'active';
```

## Work Order Management

```sql
CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    asset_id        UUID REFERENCES assets(id),
    space_id        UUID REFERENCES spaces(id),
    wo_number       VARCHAR(30) NOT NULL UNIQUE,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    wo_type         VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    reported_by     UUID REFERENCES users(id),
    assigned_to     UUID REFERENCES users(id),
    vendor_id       UUID REFERENCES organisations(id),
    due_date        TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "estimated_hours": 4.5, "actual_hours": 3.0,
    --   "estimated_cost": 500, "actual_cost": 375,
    --   "failure_code": "MECH-BEARING", "resolution_code": "REPLACED",
    --   "resolution_notes": "Replaced main bearing, aligned motor shaft",
    --   "parts_used": [
    --     { "part_number": "BRG-6208", "name": "Bearing 6208-2RS", "qty": 2, "cost": 45.00 }
    --   ],
    --   "tasks": [
    --     { "seq": 1, "desc": "Lock out power", "status": "completed", "completed_by": "..." },
    --     { "seq": 2, "desc": "Remove old bearing", "status": "completed" },
    --     { "seq": 3, "desc": "Install new bearing", "status": "completed" }
    --   ],
    --   "photos": ["wo-12345-before.jpg", "wo-12345-after.jpg"],
    --   "ai_triage": {
    --     "suggested_priority": "high",
    --     "suggested_category": "mechanical",
    --     "confidence": 0.92,
    --     "suggested_technician": "user-uuid"
    --   },
    --   "custom_fields": {
    --     "cost_center": "FAC-NYC-01",
    --     "gl_account": "6200-MAINT"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(50) NOT NULL,  -- 'created', 'assigned', 'status_changed', 'commented', 'parts_added'
    old_value       JSONB,
    new_value       JSONB,
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    trigger_type    VARCHAR(20) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "frequency_days": 90,
    --   "meter_interval": null,
    --   "lead_time_days": 7,
    --   "asset_ids": ["uuid-1", "uuid-2"],
    --   "task_template": [
    --     { "seq": 1, "desc": "Inspect belt tension", "est_minutes": 15 },
    --     { "seq": 2, "desc": "Check refrigerant levels", "est_minutes": 30 },
    --     { "seq": 3, "desc": "Clean condenser coils", "est_minutes": 45 }
    --   ],
    --   "last_generated": { "uuid-1": "2026-04-15", "uuid-2": "2026-04-16" },
    --   "next_due": { "uuid-1": "2026-07-14", "uuid-2": "2026-07-15" }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_org_status ON work_orders(org_id, status);
CREATE INDEX idx_wo_site ON work_orders(site_id);
CREATE INDEX idx_wo_asset ON work_orders(asset_id);
CREATE INDEX idx_wo_assigned ON work_orders(assigned_to);
CREATE INDEX idx_wo_due ON work_orders(due_date) WHERE status NOT IN ('completed', 'closed', 'cancelled');
CREATE INDEX idx_wo_props ON work_orders USING GIN (properties);
CREATE INDEX idx_wo_history_wo ON work_order_history(work_order_id);
```

## IoT & Sensors

```sql
CREATE TABLE iot_devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    asset_id        UUID REFERENCES assets(id),
    space_id        UUID REFERENCES spaces(id),
    device_name     VARCHAR(255) NOT NULL,
    protocol        VARCHAR(30),
    is_online       BOOLEAN DEFAULT false,
    last_seen_at    TIMESTAMPTZ,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "device_type": "sensor_gateway",
    --   "manufacturer": "Schneider Electric", "model": "SmartX IP",
    --   "serial_number": "SX-2024-00123",
    --   "ip_address": "10.0.5.42",
    --   "bacnet_device_id": 40001,
    --   "mqtt_topic_prefix": "building/hq/floor3/ahu3",
    --   "firmware_version": "4.2.1",
    --   "points": [
    --     {
    --       "point_id": "sat", "name": "Supply Air Temp",
    --       "type": "temperature", "unit": "celsius",
    --       "haystack_tags": ["air", "temp", "sensor", "discharge"],
    --       "min": 10, "max": 35, "alarm_low": 12, "alarm_high": 30
    --     },
    --     {
    --       "point_id": "occ", "name": "Occupancy",
    --       "type": "occupancy", "unit": "boolean",
    --       "haystack_tags": ["occupied", "sensor", "zone"]
    --     }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Time-series readings — high-volume, partitioned by month
CREATE TABLE iot_readings (
    device_id       UUID NOT NULL,
    point_id        VARCHAR(50) NOT NULL,   -- references the point within the device's JSONB
    recorded_at     TIMESTAMPTZ NOT NULL,
    value_numeric   NUMERIC(14, 4),
    value_text      VARCHAR(100),
    quality         SMALLINT DEFAULT 0,     -- 0=good, 1=uncertain, 2=bad
    PRIMARY KEY (device_id, point_id, recorded_at)
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_iot_devices_asset ON iot_devices(asset_id);
CREATE INDEX idx_iot_devices_space ON iot_devices(space_id);
CREATE INDEX idx_iot_devices_props ON iot_devices USING GIN (properties);
CREATE INDEX idx_iot_readings_time ON iot_readings(recorded_at DESC);
```

## Energy Management

```sql
CREATE TABLE energy_meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    space_id        UUID REFERENCES spaces(id),
    name            VARCHAR(255) NOT NULL,
    meter_type      VARCHAR(30) NOT NULL,
    unit            VARCHAR(20) NOT NULL,
    parent_meter_id UUID REFERENCES energy_meters(id),
    iot_device_id   UUID REFERENCES iot_devices(id),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties: { "is_main_meter": true, "ct_ratio": 400, "utility_account": "ACC-12345" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE energy_readings (
    meter_id        UUID NOT NULL,
    reading_at      TIMESTAMPTZ NOT NULL,
    value           NUMERIC(14, 4) NOT NULL,
    reading_type    SMALLINT NOT NULL DEFAULT 0,  -- 0=actual, 1=estimated, 2=audit
    PRIMARY KEY (meter_id, reading_at)
) PARTITION BY RANGE (reading_at);

CREATE INDEX idx_energy_readings_time ON energy_readings(reading_at DESC);
```

## Space Bookings & Occupancy

```sql
CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    booked_by       UUID NOT NULL REFERENCES users(id),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties: { "title": "Sprint Review", "attendee_count": 12, "external_cal_id": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT valid_duration CHECK (end_time > start_time)
);

-- Exclude overlapping bookings on the same space (PostgreSQL exclusion constraint)
-- Requires btree_gist extension
-- CREATE EXTENSION IF NOT EXISTS btree_gist;
-- ALTER TABLE bookings ADD CONSTRAINT no_overlap
--     EXCLUDE USING GIST (space_id WITH =, tstzrange(start_time, end_time) WITH &&)
--     WHERE (status != 'cancelled');

CREATE TABLE occupancy_readings (
    space_id        UUID NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL,
    occupant_count  SMALLINT NOT NULL,
    source          SMALLINT NOT NULL,  -- 0=sensor, 1=badge, 2=wifi, 3=manual
    PRIMARY KEY (space_id, recorded_at)
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_bookings_space_time ON bookings(space_id, start_time, end_time);
CREATE INDEX idx_bookings_user ON bookings(booked_by);
```

## Leases & Vendors

```sql
CREATE TABLE leases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    space_id        UUID REFERENCES spaces(id),  -- references a building-level space
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example:
    -- {
    --   "lease_number": "LSE-2024-001",
    --   "landlord": "Manhattan Properties LLC",
    --   "lease_type": "modified_gross",
    --   "start_date": "2024-01-01", "end_date": "2029-12-31",
    --   "monthly_rent": 45000, "annual_escalation_pct": 3.0,
    --   "security_deposit": 135000,
    --   "asc842": {
    --     "rou_asset": 2450000, "lease_liability": 2380000,
    --     "discount_rate_pct": 5.25, "classification": "operating"
    --   },
    --   "renewal_options": [
    --     { "term_years": 5, "notice_months": 12, "rent_adjustment": "market" }
    --   ],
    --   "spaces_covered": ["space-uuid-1", "space-uuid-2"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE vendors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    vendor_type     VARCHAR(50),
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties: {
    --   "contact_name": "...", "email": "...", "phone": "...",
    --   "trade": "hvac", "rating": 4.5,
    --   "insurance_expiry": "2026-12-31",
    --   "contract": { "start": "2025-01-01", "end": "2027-12-31", "value": 120000 },
    --   "sites_serviced": ["site-uuid-1", "site-uuid-2"],
    --   "certifications": ["EPA 608", "NFPA Certified"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leases_org ON leases(org_id);
CREATE INDEX idx_leases_props ON leases USING GIN (properties);
CREATE INDEX idx_vendors_org ON vendors(org_id);
CREATE INDEX idx_vendors_props ON vendors USING GIN (properties);
```

## Service Requests & Compliance

```sql
CREATE TABLE service_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    space_id        UUID REFERENCES spaces(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    request_number  VARCHAR(30) NOT NULL UNIQUE,
    subject         VARCHAR(500) NOT NULL,
    description     TEXT,
    category        VARCHAR(50),
    priority        VARCHAR(20) DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'submitted',
    work_order_id   UUID REFERENCES work_orders(id),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties: { "ai_classification": {...}, "photos": [...], "custom_fields": {...} }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    doc_type        VARCHAR(50) NOT NULL,
    file_path       VARCHAR(1000),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    uploaded_by     UUID REFERENCES users(id),
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,
    changes         JSONB,      -- { "field": {"old": "...", "new": "..."} }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sr_org_status ON service_requests(org_id, status);
CREATE INDEX idx_docs_entity ON documents(entity_type, entity_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(created_at DESC);
```

## Example: JSONB Containment Query

```sql
-- Find all HVAC assets with EPA Section 608 compliance due
SELECT id, name, asset_tag,
       properties->>'manufacturer' AS manufacturer,
       properties->'compliance'->>'next_inspection_date' AS next_inspection
FROM assets
WHERE org_id = :org_id
  AND category = 'hvac'
  AND properties @> '{"compliance": {"epa_section_608": true}}'
  AND (properties->'compliance'->>'next_inspection_date')::date <= now() + INTERVAL '30 days'
ORDER BY (properties->'compliance'->>'next_inspection_date')::date;
```

## Example: Custom Fields Query

```sql
-- Find work orders for a specific cost center (tenant-defined custom field)
SELECT id, wo_number, title, status, priority
FROM work_orders
WHERE org_id = :org_id
  AND properties->'custom_fields'->>'cost_center' = 'FAC-NYC-01'
  AND status IN ('open', 'assigned', 'in_progress')
ORDER BY
    CASE priority WHEN 'critical' THEN 1 WHEN 'high' THEN 2 WHEN 'medium' THEN 3 ELSE 4 END;
```

## Example: Space Hierarchy Query Using Materialised Path

```sql
-- Find all rooms under a specific building (using path prefix)
SELECT s.id, s.name, s.space_level, s.space_type,
       s.properties->>'usable_area_sqm' AS area,
       s.properties->>'is_bookable' AS bookable
FROM spaces s
WHERE s.path LIKE '/site-uuid/building-uuid/%'
  AND s.space_level = 'room'
  AND s.status = 'active'
ORDER BY s.path;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Config | 2 | organisations (with config JSONB), users (with roles JSONB) |
| Spatial Hierarchy | 2 | sites, spaces (self-referential with path) |
| Asset Management | 1 | assets (category-specific properties in JSONB) |
| Work Orders & PM | 3 | work_orders, work_order_history, pm_schedules |
| IoT & Sensors | 2 | iot_devices (points in JSONB), iot_readings (partitioned) |
| Energy | 2 | energy_meters, energy_readings (partitioned) |
| Bookings & Occupancy | 2 | bookings, occupancy_readings (partitioned) |
| Leases & Vendors | 2 | leases, vendors |
| Service Requests | 1 | service_requests |
| Documents & Audit | 2 | documents, audit_log |
| **Total** | **~19 tables** | Plus partitioned tables for time-series data |

---

## Key Design Decisions

1. **Self-referential `spaces` table replaces separate buildings/floors/rooms tables.** A single table with `space_level` discriminator and `parent_space_id` supports arbitrary nesting depths. The `path` column (materialised path pattern) enables efficient subtree queries without recursive CTEs.

2. **Organisation-level `config` JSONB stores custom field definitions.** When a tenant defines custom fields for work orders or assets, those definitions live in the org config. The application renders dynamic forms based on this configuration, and custom field values are stored in each entity's `properties` JSONB.

3. **IoT device points are embedded in the device's JSONB** rather than in a separate `iot_points` table. Since points are always queried in the context of their device, colocation reduces joins. Project Haystack tags are stored as arrays within each point definition.

4. **Work order tasks, parts used, and photos are JSONB arrays** inside the work order's `properties` column rather than separate tables. This eliminates 3 junction tables and keeps the work order as a self-contained document. The trade-off is that per-task or per-part reporting requires JSONB array unpacking (`jsonb_array_elements`).

5. **Lease financials (ASC 842 calculations) are in JSONB** because they vary by accounting standard (ASC 842 in US, IFRS 16 internationally) and not every tenant needs them. Storing these in typed columns would create many nullable fields.

6. **GIN indexes on all `properties` JSONB columns** enable fast containment queries (`@>`), existence checks (`?`), and key-value lookups. These indexes support the custom-field query pattern without full table scans.

7. **User roles stored as a JSONB array** (e.g., `["admin", "site_manager:site-uuid"]`) rather than in junction tables. This simplifies role checks for most queries while sacrificing the ability to do efficient "find all users with role X" queries at scale. A GIN index mitigates this.

8. **Table count is roughly half the normalized model** (~19 vs ~42), which reduces migration complexity, simplifies backup/restore, and makes the schema easier for new developers to understand. The cost is moved to application-layer validation.

9. **Vendor site assignments are inside vendor JSONB** (`sites_serviced` array) rather than a junction table. Acceptable because vendor-site relationships rarely change and the cardinality is low.

10. **Audit log captures changes as JSONB diffs** with old/new values. Combined with the `work_order_history` table (which tracks status transitions specifically), this provides a practical audit trail without the complexity of full event sourcing.
