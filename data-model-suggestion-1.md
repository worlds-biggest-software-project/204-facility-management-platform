# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Facility Management Platform · Created: 2026-05-20

## Philosophy

This model follows the classical normalized relational approach used by enterprise IWMS platforms like Archibus, Planon, and IBM TRIRIGA. Every real-world concept — site, building, floor, space, asset, work order, technician, lease, sensor — receives its own table with strict foreign key relationships. Reference data (asset types, priority levels, OmniClass codes) lives in dedicated lookup tables, ensuring consistency and enabling standards-aligned reporting.

The design mirrors the COBie (Construction Operations Building Information Exchange) data schema, which defines 19 worksheets covering Facility, Floor, Space, Zone, Type, Component, System, Job, Resource, Spare, Contact, and Document. By aligning table structures with COBie, the platform can ingest construction handover data directly into its asset register. OmniClass Table 23 product codes and UniFormat element codes provide the classification backbone for assets and building systems.

This is the approach best suited for organisations that need deep cross-entity reporting, complex joins across space-asset-work order-lease domains, and strict referential integrity for regulatory compliance. It trades flexibility for data quality.

**Best for:** Enterprise deployments with stable schemas, complex reporting requirements, and strong data governance needs.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints
- Pro: Natural alignment with COBie, OmniClass, and ISO 55000 asset lifecycle stages
- Pro: Complex cross-domain queries (e.g., "all assets on floor 3 with overdue PMs under active warranty") are straightforward SQL joins
- Pro: Well-understood by enterprise DBAs and integrators
- Con: High table count (~60-70 tables) increases migration and schema evolution complexity
- Con: Adding jurisdiction-specific or tenant-specific fields requires schema migrations
- Con: Multi-tenant implementations need careful row-level security or schema-per-tenant design
- Con: IoT time-series data does not fit naturally into normalized tables at scale

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| COBie (NBIMS-US v3) | Table structure mirrors COBie worksheets: `facilities`, `floors`, `spaces`, `asset_types`, `assets`, `systems`, `documents`, `jobs`, `spares`, `contacts` |
| ISO 55000 / ISO 55001 | Asset lifecycle stages (acquisition, operation, maintenance, decommission) modeled as `asset_lifecycle_stage` enum |
| ISO 41001 | FM management system governance reflected in `compliance_obligations` and `audit_logs` tables |
| OmniClass Table 23 | Asset classification via `omniclass_code` column on `asset_types` |
| UniFormat (OmniClass Table 21) | Building element classification via `uniformat_code` on `building_elements` |
| BOMA 2017 (Z65.1) | Space measurement fields include `gross_area`, `usable_area`, `rentable_area` per BOMA definitions |
| ISO 50001 | Energy management tables with `energy_meters`, `energy_readings`, `energy_baselines` |
| ISO 16739 (IFC4.3) | Spaces and assets can reference `ifc_guid` for BIM synchronisation |
| Project Haystack | IoT points reference `haystack_tags` for semantic interoperability |
| BACnet (ISO 16484-5) | Device integration modeled via `iot_devices` with `protocol` field supporting BACnet, MQTT |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    parent_org_id   UUID REFERENCES organisations(id),
    org_type        VARCHAR(50) NOT NULL DEFAULT 'tenant',  -- 'platform', 'tenant', 'vendor'
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    user_type       VARCHAR(30) NOT NULL,  -- 'admin', 'manager', 'technician', 'occupant', 'vendor'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id),
    role_id         UUID NOT NULL REFERENCES roles(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_users_org ON users(org_id);
CREATE INDEX idx_users_email ON users(email);
```

## Spatial Hierarchy (COBie-aligned)

```sql
CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),           -- ISO 3166-1 alpha-2
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    timezone        VARCHAR(50),        -- IANA timezone
    total_area_sqm  NUMERIC(12, 2),
    site_type       VARCHAR(50),        -- 'office', 'campus', 'hospital', 'warehouse'
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE buildings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    name            VARCHAR(255) NOT NULL,
    building_code   VARCHAR(50),
    ifc_guid        VARCHAR(64),        -- IFC4.3 GlobalId for BIM sync
    year_built      INTEGER,
    gross_area_sqm  NUMERIC(12, 2),     -- BOMA gross area
    usable_area_sqm NUMERIC(12, 2),     -- BOMA usable area
    rentable_area_sqm NUMERIC(12, 2),   -- BOMA rentable area
    floor_count     INTEGER,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE floors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    building_id     UUID NOT NULL REFERENCES buildings(id),
    name            VARCHAR(100) NOT NULL,  -- e.g., 'Ground', 'Level 1', 'B1'
    floor_number    INTEGER,
    elevation_m     NUMERIC(8, 2),
    gross_area_sqm  NUMERIC(12, 2),
    ifc_guid        VARCHAR(64),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    floor_id        UUID NOT NULL REFERENCES floors(id),
    name            VARCHAR(255) NOT NULL,
    space_code      VARCHAR(50),
    space_type      VARCHAR(50) NOT NULL,  -- 'office', 'meeting_room', 'restroom', 'server_room', 'lobby'
    usable_area_sqm NUMERIC(10, 2),
    capacity        INTEGER,
    ifc_guid        VARCHAR(64),
    is_bookable     BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE zones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    name            VARCHAR(255) NOT NULL,
    zone_type       VARCHAR(50),        -- 'hvac', 'fire', 'security', 'cleaning'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE zone_spaces (
    zone_id         UUID NOT NULL REFERENCES zones(id),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    PRIMARY KEY (zone_id, space_id)
);

CREATE INDEX idx_buildings_site ON buildings(site_id);
CREATE INDEX idx_floors_building ON floors(building_id);
CREATE INDEX idx_spaces_floor ON spaces(floor_id);
CREATE INDEX idx_sites_org ON sites(org_id);
```

## Asset Management (ISO 55000-aligned)

```sql
CREATE TABLE asset_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    omniclass_code  VARCHAR(30),        -- OmniClass Table 23 product code
    uniformat_code  VARCHAR(30),        -- UniFormat element code (OmniClass Table 21)
    parent_id       UUID REFERENCES asset_categories(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE asset_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id     UUID NOT NULL REFERENCES asset_categories(id),
    name            VARCHAR(255) NOT NULL,
    manufacturer    VARCHAR(255),
    model_number    VARCHAR(255),
    expected_life_years INTEGER,
    warranty_duration_months INTEGER,
    specifications  JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    asset_type_id   UUID NOT NULL REFERENCES asset_types(id),
    space_id        UUID REFERENCES spaces(id),
    name            VARCHAR(255) NOT NULL,
    asset_tag       VARCHAR(100),
    serial_number   VARCHAR(255),
    barcode         VARCHAR(255),
    parent_asset_id UUID REFERENCES assets(id),  -- for sub-components
    lifecycle_stage VARCHAR(30) NOT NULL DEFAULT 'operational',
        -- ISO 55000 stages: 'planned', 'procured', 'installed', 'operational', 'degraded', 'decommissioned'
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium',  -- 'critical', 'high', 'medium', 'low'
    install_date    DATE,
    warranty_start  DATE,
    warranty_end    DATE,
    purchase_cost   NUMERIC(12, 2),
    replacement_cost NUMERIC(12, 2),
    condition_score INTEGER CHECK (condition_score BETWEEN 1 AND 5),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE systems (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    name            VARCHAR(255) NOT NULL,
    system_type     VARCHAR(50) NOT NULL,  -- 'hvac', 'electrical', 'plumbing', 'fire', 'elevator', 'lighting'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE system_assets (
    system_id       UUID NOT NULL REFERENCES systems(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    role_in_system  VARCHAR(100),
    PRIMARY KEY (system_id, asset_id)
);

CREATE TABLE spare_parts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    part_number     VARCHAR(100),
    manufacturer    VARCHAR(255),
    unit_cost       NUMERIC(10, 2),
    quantity_on_hand INTEGER NOT NULL DEFAULT 0,
    reorder_point   INTEGER NOT NULL DEFAULT 0,
    storage_location VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE asset_type_spare_parts (
    asset_type_id   UUID NOT NULL REFERENCES asset_types(id),
    spare_part_id   UUID NOT NULL REFERENCES spare_parts(id),
    quantity_per_service INTEGER DEFAULT 1,
    PRIMARY KEY (asset_type_id, spare_part_id)
);

CREATE INDEX idx_assets_org ON assets(org_id);
CREATE INDEX idx_assets_type ON assets(asset_type_id);
CREATE INDEX idx_assets_space ON assets(space_id);
CREATE INDEX idx_assets_tag ON assets(asset_tag);
CREATE INDEX idx_assets_serial ON assets(serial_number);
CREATE INDEX idx_assets_lifecycle ON assets(lifecycle_stage);
```

## Work Order & Maintenance Management

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
    wo_type         VARCHAR(30) NOT NULL,  -- 'corrective', 'preventive', 'emergency', 'inspection', 'project'
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',  -- 'critical', 'high', 'medium', 'low'
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
        -- 'draft', 'open', 'assigned', 'in_progress', 'on_hold', 'completed', 'closed', 'cancelled'
    reported_by     UUID REFERENCES users(id),
    assigned_to     UUID REFERENCES users(id),
    vendor_id       UUID REFERENCES organisations(id),
    estimated_hours NUMERIC(6, 2),
    actual_hours    NUMERIC(6, 2),
    estimated_cost  NUMERIC(10, 2),
    actual_cost     NUMERIC(10, 2),
    due_date        TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    failure_code    VARCHAR(50),
    resolution_code VARCHAR(50),
    resolution_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id),
    sequence_number INTEGER NOT NULL,
    description     TEXT NOT NULL,
    estimated_minutes INTEGER,
    actual_minutes  INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'in_progress', 'completed', 'skipped'
    completed_by    UUID REFERENCES users(id),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order_comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    comment         TEXT NOT NULL,
    is_internal     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order_parts_used (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id),
    spare_part_id   UUID NOT NULL REFERENCES spare_parts(id),
    quantity_used   INTEGER NOT NULL,
    unit_cost       NUMERIC(10, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    trigger_type    VARCHAR(20) NOT NULL,  -- 'time', 'meter', 'condition', 'combined'
    frequency_days  INTEGER,              -- for time-based
    meter_interval  NUMERIC(10, 2),       -- for meter-based
    lead_time_days  INTEGER DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_schedule_assets (
    pm_schedule_id  UUID NOT NULL REFERENCES pm_schedules(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    last_performed  TIMESTAMPTZ,
    next_due        TIMESTAMPTZ,
    meter_last_reading NUMERIC(12, 2),
    PRIMARY KEY (pm_schedule_id, asset_id)
);

CREATE TABLE pm_task_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pm_schedule_id  UUID NOT NULL REFERENCES pm_schedules(id),
    sequence_number INTEGER NOT NULL,
    description     TEXT NOT NULL,
    estimated_minutes INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_org ON work_orders(org_id);
CREATE INDEX idx_wo_site ON work_orders(site_id);
CREATE INDEX idx_wo_asset ON work_orders(asset_id);
CREATE INDEX idx_wo_status ON work_orders(status);
CREATE INDEX idx_wo_assigned ON work_orders(assigned_to);
CREATE INDEX idx_wo_type_status ON work_orders(wo_type, status);
CREATE INDEX idx_wo_due_date ON work_orders(due_date) WHERE status NOT IN ('completed', 'closed', 'cancelled');
CREATE INDEX idx_pm_next_due ON pm_schedule_assets(next_due);
```

## IoT & Sensor Data

```sql
CREATE TABLE iot_devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    asset_id        UUID REFERENCES assets(id),
    space_id        UUID REFERENCES spaces(id),
    device_name     VARCHAR(255) NOT NULL,
    device_type     VARCHAR(50) NOT NULL,  -- 'sensor', 'actuator', 'controller', 'gateway'
    protocol        VARCHAR(30),           -- 'bacnet', 'mqtt', 'modbus', 'rest'
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    serial_number   VARCHAR(255),
    ip_address      INET,
    mqtt_topic      VARCHAR(500),
    bacnet_device_id INTEGER,
    is_online       BOOLEAN DEFAULT false,
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE iot_points (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       UUID NOT NULL REFERENCES iot_devices(id),
    point_name      VARCHAR(255) NOT NULL,
    point_type      VARCHAR(30) NOT NULL,  -- 'temperature', 'humidity', 'occupancy', 'co2', 'power', 'flow'
    unit            VARCHAR(30),           -- 'celsius', 'percent', 'ppm', 'kw', 'lpm'
    haystack_tags   VARCHAR(500),          -- Project Haystack tag string, e.g., 'air,temp,sensor,zone'
    min_value       NUMERIC(12, 4),
    max_value       NUMERIC(12, 4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Time-series readings partitioned by month for query performance
CREATE TABLE iot_readings (
    id              UUID DEFAULT gen_random_uuid(),
    point_id        UUID NOT NULL REFERENCES iot_points(id),
    recorded_at     TIMESTAMPTZ NOT NULL,
    value_numeric   NUMERIC(14, 4),
    value_text      VARCHAR(255),
    quality         VARCHAR(20) DEFAULT 'good',  -- 'good', 'uncertain', 'bad'
    PRIMARY KEY (id, recorded_at)
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions (example)
-- CREATE TABLE iot_readings_2026_05 PARTITION OF iot_readings
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_iot_devices_asset ON iot_devices(asset_id);
CREATE INDEX idx_iot_devices_space ON iot_devices(space_id);
CREATE INDEX idx_iot_points_device ON iot_points(device_id);
CREATE INDEX idx_iot_readings_point_time ON iot_readings(point_id, recorded_at DESC);
```

## Energy Management (ISO 50001-aligned)

```sql
CREATE TABLE energy_meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    building_id     UUID REFERENCES buildings(id),
    name            VARCHAR(255) NOT NULL,
    meter_type      VARCHAR(30) NOT NULL,  -- 'electricity', 'gas', 'water', 'steam', 'chilled_water'
    unit            VARCHAR(20) NOT NULL,  -- 'kwh', 'therms', 'gallons', 'mmbtu'
    is_main_meter   BOOLEAN DEFAULT false,
    parent_meter_id UUID REFERENCES energy_meters(id),  -- sub-metering hierarchy
    iot_device_id   UUID REFERENCES iot_devices(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE energy_readings (
    id              UUID DEFAULT gen_random_uuid(),
    meter_id        UUID NOT NULL REFERENCES energy_meters(id),
    reading_at      TIMESTAMPTZ NOT NULL,
    value           NUMERIC(14, 4) NOT NULL,
    reading_type    VARCHAR(20) NOT NULL DEFAULT 'actual',  -- 'actual', 'estimated', 'audit'
    PRIMARY KEY (id, reading_at)
) PARTITION BY RANGE (reading_at);

CREATE TABLE energy_baselines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    baseline_year   INTEGER NOT NULL,
    energy_type     VARCHAR(30) NOT NULL,
    annual_consumption NUMERIC(14, 2),
    energy_star_score INTEGER CHECK (energy_star_score BETWEEN 1 AND 100),
    source_eui      NUMERIC(10, 2),       -- Source Energy Use Intensity (kBtu/sqft)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_energy_readings_meter_time ON energy_readings(meter_id, reading_at DESC);
```

## Space Booking & Occupancy

```sql
CREATE TABLE space_bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    booked_by       UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(255),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    attendee_count  INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed',  -- 'tentative', 'confirmed', 'cancelled'
    external_cal_id VARCHAR(255),  -- Google Calendar / Microsoft Graph event ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT no_negative_duration CHECK (end_time > start_time)
);

CREATE TABLE occupancy_snapshots (
    id              UUID DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    recorded_at     TIMESTAMPTZ NOT NULL,
    occupant_count  INTEGER NOT NULL,
    capacity        INTEGER,
    source          VARCHAR(30),  -- 'sensor', 'badge', 'wifi', 'manual'
    PRIMARY KEY (id, recorded_at)
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_bookings_space_time ON space_bookings(space_id, start_time, end_time);
CREATE INDEX idx_bookings_user ON space_bookings(booked_by);
CREATE INDEX idx_occupancy_space_time ON occupancy_snapshots(space_id, recorded_at DESC);
```

## Lease & Real Estate

```sql
CREATE TABLE leases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    building_id     UUID REFERENCES buildings(id),
    lease_number    VARCHAR(100),
    landlord_name   VARCHAR(255),
    lease_type      VARCHAR(30),  -- 'gross', 'net', 'modified_gross', 'ground'
    start_date      DATE NOT NULL,
    end_date        DATE,
    monthly_rent    NUMERIC(12, 2),
    annual_escalation_pct NUMERIC(5, 2),
    security_deposit NUMERIC(12, 2),
    asc842_rou_asset NUMERIC(14, 2),      -- ASC 842 Right-of-Use asset value
    asc842_lease_liability NUMERIC(14, 2), -- ASC 842 lease liability
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE lease_spaces (
    lease_id        UUID NOT NULL REFERENCES leases(id),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    PRIMARY KEY (lease_id, space_id)
);

CREATE INDEX idx_leases_org ON leases(org_id);
CREATE INDEX idx_leases_end ON leases(end_date) WHERE status = 'active';
```

## Documents & Attachments

```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(500) NOT NULL,
    doc_type        VARCHAR(50) NOT NULL,  -- 'manual', 'warranty', 'certificate', 'floor_plan', 'photo', 'report'
    file_path       VARCHAR(1000),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE document_links (
    document_id     UUID NOT NULL REFERENCES documents(id),
    entity_type     VARCHAR(50) NOT NULL,  -- 'asset', 'work_order', 'space', 'building', 'lease'
    entity_id       UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (document_id, entity_type, entity_id)
);

CREATE INDEX idx_doc_links_entity ON document_links(entity_type, entity_id);
```

## Vendor & Contractor Management

```sql
CREATE TABLE vendors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    contact_name    VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    address         TEXT,
    vendor_type     VARCHAR(50),  -- 'contractor', 'supplier', 'consultant'
    trade           VARCHAR(100),  -- 'hvac', 'electrical', 'plumbing', 'cleaning', 'security'
    rating          NUMERIC(3, 2) CHECK (rating BETWEEN 0 AND 5),
    is_approved     BOOLEAN NOT NULL DEFAULT false,
    insurance_expiry DATE,
    contract_start  DATE,
    contract_end    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE vendor_sites (
    vendor_id       UUID NOT NULL REFERENCES vendors(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    PRIMARY KEY (vendor_id, site_id)
);

CREATE INDEX idx_vendors_org ON vendors(org_id);
CREATE INDEX idx_vendors_trade ON vendors(trade);
```

## Compliance & Audit

```sql
CREATE TABLE compliance_obligations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    site_id         UUID REFERENCES sites(id),
    name            VARCHAR(255) NOT NULL,
    regulation      VARCHAR(255),         -- e.g., 'ISO 41001', 'OSHA', 'Fire Code'
    frequency_days  INTEGER,
    next_due        DATE,
    assigned_to     UUID REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,  -- 'create', 'update', 'delete', 'status_change'
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_user ON audit_logs(user_id);
```

## Service Requests (Occupant Portal)

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
    category        VARCHAR(50),  -- 'maintenance', 'cleaning', 'temperature', 'lighting', 'pest', 'other'
    priority        VARCHAR(20) DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'submitted',
    work_order_id   UUID REFERENCES work_orders(id),  -- linked when WO is created
    ai_classification JSONB,  -- AI-generated classification and priority suggestion
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sr_org_status ON service_requests(org_id, status);
CREATE INDEX idx_sr_site ON service_requests(site_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organisations, users, roles, user_roles |
| Spatial Hierarchy | 5 | sites, buildings, floors, spaces, zones + 1 junction |
| Asset Management | 6 | asset_categories, asset_types, assets, systems, spare_parts + 2 junctions |
| Work Orders & PM | 6 | work_orders, wo_tasks, wo_comments, wo_parts_used, pm_schedules, pm_task_templates + 1 junction |
| IoT & Sensors | 3 | iot_devices, iot_points, iot_readings (partitioned) |
| Energy Management | 3 | energy_meters, energy_readings (partitioned), energy_baselines |
| Space Booking & Occupancy | 2 | space_bookings, occupancy_snapshots (partitioned) |
| Lease & Real Estate | 2 | leases + 1 junction |
| Documents | 2 | documents, document_links |
| Vendors | 2 | vendors + 1 junction |
| Compliance & Audit | 2 | compliance_obligations, audit_logs |
| Service Requests | 1 | service_requests |
| **Total** | **~42 tables** | Including junction tables and partitioned tables |

---

## Key Design Decisions

1. **COBie-aligned spatial hierarchy** (site > building > floor > space > zone) enables direct ingestion of construction handover data, reducing the time to populate a new building's asset register from weeks to hours.

2. **OmniClass and UniFormat classification on asset types** ensures assets are categorized using industry-standard codes, enabling cross-platform reporting and benchmarking against industry databases.

3. **ISO 55000 lifecycle stages as an enum on assets** rather than a separate lifecycle table keeps queries simple while still tracking where each asset sits in its lifecycle.

4. **Time-series tables (iot_readings, energy_readings, occupancy_snapshots) use PostgreSQL native partitioning** by month, enabling efficient range queries and automatic partition management for data retention policies.

5. **Polymorphic document linking via entity_type + entity_id** avoids creating separate document junction tables for every entity, though it sacrifices foreign key enforcement on the entity_id.

6. **Service requests as a separate table from work orders** reflects the real-world workflow: occupants submit requests, FM managers triage and optionally create work orders. The `ai_classification` JSONB column stores AI-generated triage suggestions.

7. **Multi-tenant via org_id foreign keys** on all major tables with row-level security, rather than schema-per-tenant, to keep operational complexity manageable for mid-market deployments.

8. **BOMA area measurements** (gross, usable, rentable) stored as explicit columns on buildings and spaces rather than in a generic measurements table, because these are the three canonical area types used in every commercial real estate context.

9. **Vendor management integrated with work orders** via the vendor_id foreign key, supporting both in-house technician assignment and outsourced contractor dispatch from the same work order table.

10. **Audit logs capture old/new values as JSONB** providing a flexible audit trail without requiring a dedicated history table for every entity. This is a pragmatic compromise between full event sourcing and no audit trail.
