# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Facility Management Platform · Created: 2026-05-20

## Philosophy

This model combines relational tables for operational CRUD (work orders, bookings, energy readings) with a property graph layer for relationship-heavy queries. The graph layer models the physical topology of buildings, equipment dependency chains, space containment hierarchies, and IoT sensor-to-asset-to-system relationships as nodes and edges in a generic graph structure.

Facility management is inherently a graph problem. A chiller failure cascades through the cooling system to affect air handling units, which affect temperature in occupied spaces, which triggers comfort complaints. Understanding these dependency chains — and predicting cascade failures — requires traversing relationships that span multiple entity types. In a normalised relational model, this requires joining 6-8 tables with multiple self-referential lookups. In a graph, it is a single traversal query.

This pattern draws on the approach used by building digital twin platforms (Microsoft Azure Digital Twins, DTDL) and semantic building models (Brick Schema, Project Haystack 5). These standards model buildings as graphs of equipment, spaces, and data points connected by typed relationships (`feeds`, `contains`, `hasPoint`, `servedBy`). The graph-relational hybrid keeps the operational simplicity of relational tables while enabling graph-powered analytics for failure propagation, impact analysis, and AI-driven insights.

**Best for:** Large multi-building portfolios with complex mechanical/electrical systems, IoT-heavy deployments needing cascade failure analysis, and organisations wanting digital twin capabilities.

**Trade-offs:**
- Pro: Natural representation of building system dependencies and equipment hierarchies
- Pro: Cascade impact analysis ("if this chiller fails, which spaces are affected?") is a single graph traversal
- Pro: Aligns with Brick Schema and Project Haystack 5 graph-based semantics
- Pro: Enables AI-driven failure propagation prediction across interconnected systems
- Pro: Flexible — new relationship types are data (new edge types), not schema changes
- Con: Graph queries (recursive CTEs or dedicated graph engine) require different developer skills
- Con: Maintaining consistency between relational tables and graph nodes requires careful sync
- Con: Graph edge table can grow large in IoT-heavy deployments (thousands of edges per building)
- Con: PostgreSQL recursive CTEs are less performant than dedicated graph databases for deep traversals
- Con: Two mental models (relational + graph) increase cognitive load for the development team

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Brick Schema | Graph relationship types (`feeds`, `hasPoint`, `isPartOf`, `contains`) align with Brick ontology |
| Project Haystack 5 | Node tags use Haystack semantic tagging; Xeto schema language informs type definitions |
| DTDL (Digital Twins Definition Language) | Node and relationship patterns mirror Azure Digital Twins DTDL models |
| IFC4.3 (ISO 16739) | Spatial containment hierarchy (IfcSite > IfcBuilding > IfcBuildingStorey > IfcSpace) modeled as graph edges |
| COBie | COBie System-Component relationships imported as graph edges |
| ISO 55000 | Asset lifecycle transitions tracked on graph nodes; dependency graphs inform criticality analysis |
| OmniClass | Node classification via OmniClass codes stored as node properties |
| BACnet (ISO 16484-5) | BACnet device-object relationships modeled as graph edges |
| ISO 41001 | Compliance obligation dependencies (which regulations apply to which equipment in which jurisdictions) as graph edges |

---

## Graph Layer

```sql
-- Generic graph node — represents any entity in the building topology
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Brick Schema aligned types:
    --   'Site', 'Building', 'Floor', 'Room', 'Zone',
    --   'Equipment', 'Sensor', 'Actuator', 'Point',
    --   'System', 'Loop', 'Meter',
    --   'Person', 'Vendor'
    name            VARCHAR(255) NOT NULL,
    entity_id       UUID,               -- FK to the corresponding relational table record
    entity_table    VARCHAR(50),         -- 'sites', 'spaces', 'assets', 'iot_devices', 'users'
    tags            JSONB NOT NULL DEFAULT '{}',
    -- Haystack-style tags:
    -- Equipment: { "equip": true, "ahu": true, "hvac": true, "capacity_tons": 200 }
    -- Sensor:    { "sensor": true, "temp": true, "air": true, "discharge": true, "unit": "celsius" }
    -- Room:      { "space": true, "room": true, "office": true, "area_sqm": 45 }
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Additional properties not covered by tags:
    -- { "omniclass_code": "23-33 11 00", "ifc_guid": "...", "brick_class": "Brick:AHU" }
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Generic graph edge — typed relationships between nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Brick Schema aligned relationship types:
    --   'contains'       — Site contains Building, Building contains Floor
    --   'isPartOf'       — Floor isPartOf Building (inverse of contains)
    --   'feeds'          — Chiller feeds AHU, AHU feeds VAV
    --   'isFedBy'        — VAV isFedBy AHU (inverse of feeds)
    --   'hasPoint'       — Equipment hasPoint Sensor
    --   'isPointOf'      — Sensor isPointOf Equipment (inverse)
    --   'serves'         — AHU serves Room
    --   'isServedBy'     — Room isServedBy AHU (inverse)
    --   'hasLocation'    — Equipment hasLocation Room
    --   'monitors'       — Sensor monitors Equipment
    --   'controls'       — Actuator controls Equipment
    --   'dependsOn'      — Generator dependsOn Fuel_Supply
    --   'assignedTo'     — WorkOrder assignedTo Technician
    --   'maintainedBy'   — Equipment maintainedBy Vendor
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Edge properties: { "capacity_pct": 100, "since": "2024-01-15", "priority": 1 }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_org ON graph_nodes(org_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id, entity_table);
CREATE INDEX idx_graph_nodes_tags ON graph_nodes USING GIN (tags);
CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_org ON graph_edges(org_id);

-- Unique constraint to prevent duplicate edges
CREATE UNIQUE INDEX idx_graph_edges_unique
    ON graph_edges(source_node_id, target_node_id, edge_type);
```

## Relational Layer — Operational Tables

### Sites & Spaces

```sql
CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    graph_node_id   UUID REFERENCES graph_nodes(id),  -- link to graph
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2),
    timezone        VARCHAR(50),
    address         JSONB DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    graph_node_id   UUID REFERENCES graph_nodes(id),
    name            VARCHAR(255) NOT NULL,
    space_level     VARCHAR(20) NOT NULL,
    space_type      VARCHAR(50),
    usable_area_sqm NUMERIC(10, 2),
    rentable_area_sqm NUMERIC(10, 2),
    capacity        INTEGER,
    ifc_guid        VARCHAR(64),
    is_bookable     BOOLEAN DEFAULT false,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spaces_site ON spaces(site_id);
CREATE INDEX idx_spaces_graph ON spaces(graph_node_id);
```

### Assets

```sql
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    graph_node_id   UUID REFERENCES graph_nodes(id),  -- link to graph node
    space_id        UUID REFERENCES spaces(id),
    name            VARCHAR(255) NOT NULL,
    asset_tag       VARCHAR(100),
    serial_number   VARCHAR(255),
    category        VARCHAR(100) NOT NULL,
    omniclass_code  VARCHAR(30),
    manufacturer    VARCHAR(255),
    model_number    VARCHAR(255),
    lifecycle_stage VARCHAR(30) NOT NULL DEFAULT 'operational',
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium',
    condition_score INTEGER CHECK (condition_score BETWEEN 1 AND 5),
    install_date    DATE,
    warranty_end    DATE,
    purchase_cost   NUMERIC(12, 2),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_org ON assets(org_id);
CREATE INDEX idx_assets_space ON assets(space_id);
CREATE INDEX idx_assets_graph ON assets(graph_node_id);
CREATE INDEX idx_assets_category ON assets(category);
CREATE INDEX idx_assets_tag ON assets(asset_tag);
```

### Work Orders

```sql
CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    site_id         UUID NOT NULL REFERENCES sites(id),
    asset_id        UUID REFERENCES assets(id),
    space_id        UUID REFERENCES spaces(id),
    wo_number       VARCHAR(30) NOT NULL UNIQUE,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    wo_type         VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    reported_by     UUID,
    assigned_to     UUID,
    vendor_id       UUID,
    estimated_hours NUMERIC(6, 2),
    actual_hours    NUMERIC(6, 2),
    estimated_cost  NUMERIC(10, 2),
    actual_cost     NUMERIC(10, 2),
    due_date        TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
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
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    completed_by    UUID,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    trigger_type    VARCHAR(20) NOT NULL,
    frequency_days  INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pm_schedule_assets (
    pm_schedule_id  UUID NOT NULL REFERENCES pm_schedules(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    next_due        TIMESTAMPTZ,
    PRIMARY KEY (pm_schedule_id, asset_id)
);

CREATE INDEX idx_wo_org_status ON work_orders(org_id, status);
CREATE INDEX idx_wo_asset ON work_orders(asset_id);
CREATE INDEX idx_wo_site ON work_orders(site_id);
CREATE INDEX idx_wo_assigned ON work_orders(assigned_to);
```

### IoT & Energy

```sql
CREATE TABLE iot_devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    graph_node_id   UUID REFERENCES graph_nodes(id),
    asset_id        UUID REFERENCES assets(id),
    space_id        UUID REFERENCES spaces(id),
    device_name     VARCHAR(255) NOT NULL,
    device_type     VARCHAR(50) NOT NULL,
    protocol        VARCHAR(30),
    is_online       BOOLEAN DEFAULT false,
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE iot_points (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       UUID NOT NULL REFERENCES iot_devices(id),
    graph_node_id   UUID REFERENCES graph_nodes(id),
    point_name      VARCHAR(255) NOT NULL,
    point_type      VARCHAR(30) NOT NULL,
    unit            VARCHAR(30),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE iot_readings (
    point_id        UUID NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL,
    value_numeric   NUMERIC(14, 4),
    quality         SMALLINT DEFAULT 0,
    PRIMARY KEY (point_id, recorded_at)
) PARTITION BY RANGE (recorded_at);

CREATE TABLE energy_meters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id),
    graph_node_id   UUID REFERENCES graph_nodes(id),
    name            VARCHAR(255) NOT NULL,
    meter_type      VARCHAR(30) NOT NULL,
    unit            VARCHAR(20) NOT NULL,
    parent_meter_id UUID REFERENCES energy_meters(id),
    iot_device_id   UUID REFERENCES iot_devices(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE energy_readings (
    meter_id        UUID NOT NULL,
    reading_at      TIMESTAMPTZ NOT NULL,
    value           NUMERIC(14, 4) NOT NULL,
    PRIMARY KEY (meter_id, reading_at)
) PARTITION BY RANGE (reading_at);

CREATE INDEX idx_iot_devices_graph ON iot_devices(graph_node_id);
CREATE INDEX idx_iot_points_device ON iot_points(device_id);
CREATE INDEX idx_iot_points_graph ON iot_points(graph_node_id);
```

### Bookings, Leases, Documents, Audit

```sql
CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id),
    booked_by       UUID NOT NULL,
    title           VARCHAR(255),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    attendee_count  INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'confirmed',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE leases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    space_id        UUID REFERENCES spaces(id),
    lease_number    VARCHAR(100),
    landlord_name   VARCHAR(255),
    lease_type      VARCHAR(30),
    start_date      DATE NOT NULL,
    end_date        DATE,
    monthly_rent    NUMERIC(12, 2),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    properties      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    doc_type        VARCHAR(50) NOT NULL,
    file_path       VARCHAR(1000),
    mime_type       VARCHAR(100),
    uploaded_by     UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,
    changes         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bookings_space_time ON bookings(space_id, start_time, end_time);
CREATE INDEX idx_docs_entity ON documents(entity_type, entity_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(created_at DESC);
```

---

## Example: Cascade Impact Analysis

```sql
-- "If Chiller-01 fails, which spaces will be affected?"
-- Traverse the graph: Chiller → feeds → AHU → serves → Room

WITH RECURSIVE impact_chain AS (
    -- Start from the failing equipment
    SELECT
        gn.id AS node_id,
        gn.name AS node_name,
        gn.node_type,
        ge.edge_type,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_id = :chiller_asset_id
      AND gn.entity_table = 'assets'

    UNION ALL

    -- Traverse downstream edges (feeds, serves, contains)
    SELECT
        target.id,
        target.name,
        target.node_type,
        ge.edge_type,
        ic.depth + 1,
        ic.path || target.id
    FROM impact_chain ic
    JOIN graph_edges ge ON ge.source_node_id = ic.node_id
    JOIN graph_nodes target ON target.id = ge.target_node_id
    WHERE ge.edge_type IN ('feeds', 'serves', 'contains')
      AND target.id != ALL(ic.path)  -- prevent cycles
      AND ic.depth < 10              -- limit traversal depth
)
SELECT
    node_id,
    node_name,
    node_type,
    edge_type,
    depth
FROM impact_chain
WHERE node_type IN ('Room', 'Floor')  -- only show affected spaces
ORDER BY depth;
```

## Example: Equipment Dependency Graph for Predictive Maintenance

```sql
-- Find all equipment upstream of a failing asset (what feeds this asset?)
WITH RECURSIVE upstream AS (
    SELECT
        gn.id, gn.name, gn.node_type,
        ge.edge_type,
        0 AS depth
    FROM graph_nodes gn
    WHERE gn.entity_id = :failing_asset_id
      AND gn.entity_table = 'assets'

    UNION ALL

    SELECT
        source.id, source.name, source.node_type,
        ge.edge_type,
        u.depth + 1
    FROM upstream u
    JOIN graph_edges ge ON ge.target_node_id = u.id
    JOIN graph_nodes source ON source.id = ge.source_node_id
    WHERE ge.edge_type IN ('feeds', 'supplies', 'powers')
      AND u.depth < 5
)
SELECT * FROM upstream WHERE node_type = 'Equipment' ORDER BY depth;
```

## Example: Haystack-Style Tag Query

```sql
-- Find all discharge air temperature sensors across all AHUs
-- Equivalent to Haystack filter: discharge and air and temp and sensor

SELECT
    gn.id,
    gn.name,
    gn.tags,
    parent_equip.name AS equipment_name,
    space.name AS location
FROM graph_nodes gn
JOIN graph_edges e_point ON e_point.source_node_id = gn.id
    AND e_point.edge_type = 'isPointOf'
JOIN graph_nodes parent_equip ON parent_equip.id = e_point.target_node_id
LEFT JOIN graph_edges e_loc ON e_loc.source_node_id = parent_equip.id
    AND e_loc.edge_type = 'hasLocation'
LEFT JOIN graph_nodes space ON space.id = e_loc.target_node_id
WHERE gn.tags @> '{"sensor": true, "temp": true, "air": true, "discharge": true}'
  AND gn.org_id = :org_id;
```

## Example: Building Digital Twin Initialisation from COBie

```sql
-- Import COBie data as graph nodes and edges
-- Step 1: Create site node
INSERT INTO graph_nodes (org_id, node_type, name, entity_id, entity_table, tags)
VALUES (:org_id, 'Site', 'HQ Campus', :site_id, 'sites', '{"site": true}');

-- Step 2: Create building node and 'contains' edge
INSERT INTO graph_nodes (org_id, node_type, name, entity_id, entity_table, tags)
VALUES (:org_id, 'Building', 'Tower A', :building_space_id, 'spaces',
        '{"building": true, "year_built": 2018}');

INSERT INTO graph_edges (org_id, source_node_id, target_node_id, edge_type)
VALUES (:org_id, :site_node_id, :building_node_id, 'contains');

-- Step 3: Create system node and equipment-to-system edges (COBie System worksheet)
INSERT INTO graph_nodes (org_id, node_type, name, tags)
VALUES (:org_id, 'System', 'Chilled Water Loop', '{"system": true, "chilled_water": true}');

-- Step 4: Link equipment to system
INSERT INTO graph_edges (org_id, source_node_id, target_node_id, edge_type)
VALUES (:org_id, :chiller_node_id, :chw_system_node_id, 'isPartOf');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Sites & Spaces | 2 | sites, spaces |
| Assets | 1 | assets (with graph_node_id FK) |
| Work Orders & PM | 4 | work_orders, work_order_tasks, pm_schedules, pm_schedule_assets |
| IoT & Sensors | 3 | iot_devices, iot_points, iot_readings (partitioned) |
| Energy | 2 | energy_meters, energy_readings (partitioned) |
| Bookings | 1 | bookings |
| Leases | 1 | leases |
| Documents & Audit | 2 | documents, audit_log |
| **Total** | **~18 tables** | Plus graph_nodes/edges which can represent unlimited entity types and relationships |

---

## Key Design Decisions

1. **Generic graph layer (`graph_nodes` + `graph_edges`) rather than a dedicated graph database.** PostgreSQL recursive CTEs handle traversals up to depth 10-15 efficiently. This avoids the operational overhead of running a separate Neo4j or similar graph database alongside PostgreSQL. If traversal performance becomes critical at scale (>1M nodes), the graph layer can be extracted to a dedicated graph database without changing the relational layer.

2. **Bidirectional link between graph and relational tables.** Each relational record (asset, space, device) has a `graph_node_id` FK, and each graph node has `entity_id` + `entity_table` pointing back to the relational record. This allows queries to start from either side: operational queries use relational tables, topology queries use the graph.

3. **Brick Schema relationship types as the edge vocabulary.** Using established relationship types (`feeds`, `contains`, `hasPoint`, `serves`) rather than custom edge types ensures semantic interoperability with Brick Schema-compatible tools and enables automated dashboard generation from graph topology.

4. **Haystack tags as JSONB on graph nodes.** Project Haystack's tag-based classification system maps naturally to JSONB containment queries. A sensor tagged `{"sensor": true, "temp": true, "air": true, "discharge": true}` can be found with `tags @> '{"sensor": true, "temp": true}'`, which is the SQL equivalent of a Haystack filter expression.

5. **Graph edges are data, not schema.** Adding a new relationship type (e.g., `adjacentTo` for space adjacency analysis) requires only inserting new rows into `graph_edges`, not a schema migration. This makes the system extensible without DDL changes.

6. **Cascade impact analysis is the primary graph use case.** When a critical asset fails, the graph traversal instantly identifies all downstream spaces and occupants affected. This information drives automated notification routing, work order prioritisation, and space rebooking.

7. **IoT points are both relational records AND graph nodes.** The relational `iot_points` table supports efficient time-series joins. The graph node representation supports topology queries ("which sensors are on equipment served by this chiller?"). The two representations are linked via `graph_node_id`.

8. **COBie import creates both relational records and graph topology.** The COBie System and Component worksheets naturally map to graph edges (system-contains-component). This dual import means the graph topology is immediately available for analysis after a COBie data handover.

9. **Unique constraint on `(source, target, edge_type)` prevents duplicate edges** while still allowing multiple relationship types between the same pair of nodes (e.g., a sensor can both `monitors` and `isPartOf` the same equipment).

10. **Graph traversal depth is capped at 10** in recursive CTEs to prevent runaway queries. In practice, building system dependency chains rarely exceed depth 5-6 (Site > Building > System > Equipment > Sub-equipment > Sensor), so this limit is conservative.
