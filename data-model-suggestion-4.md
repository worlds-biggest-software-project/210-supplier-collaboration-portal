# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Supplier Collaboration Portal · Created: 2026-05-20

## Philosophy

This model combines relational tables for operational CRUD (purchase orders, invoices, scorecards) with a property graph layer for relationship-intensive queries. The graph layer models the supply chain as a network of connected entities — organisations, products, locations, risk signals, certifications — enabling queries that are difficult or impossible in pure relational models: "which suppliers share a sub-tier supplier?", "what is the shortest alternative supply path for this component?", "which buyers are exposed if this supplier fails?"

Supply chains are inherently graph-structured. An organisation buys from suppliers who buy from their own suppliers (sub-tier), who source raw materials from specific regions, who hold certifications from specific bodies, who ship through specific logistics providers. These multi-hop relationships are where the most valuable supply chain intelligence lives — and where relational JOINs become prohibitively expensive.

This approach uses PostgreSQL for all operational data (POs, invoices, ASNs) and adds a graph layer implemented either as (a) `graph_nodes` / `graph_edges` tables in PostgreSQL with recursive CTEs, or (b) Apache AGE (a PostgreSQL extension providing native Cypher query support), or (c) a dedicated graph database like Neo4j synchronized from PostgreSQL. The choice depends on scale and query complexity. This document shows the PostgreSQL-native approach with recursive CTEs, which requires no additional infrastructure.

**Best for:** Organisations needing supply chain network analysis, multi-tier visibility, conflict-of-interest detection, alternative supplier discovery, and risk propagation analysis.

**Trade-offs:**
- Pro: Multi-hop relationship queries (sub-tier analysis, risk propagation) are natural and fast
- Pro: Alternative supplier discovery based on capability/certification graph matching
- Pro: Conflict-of-interest and dual-sourcing analysis across the buyer network
- Pro: Supply chain resilience analysis — "what happens if supplier X fails?"
- Pro: AI-native — graph embeddings enable supplier recommendation and risk prediction
- Con: More complex to implement and maintain than pure relational
- Con: Graph queries require different mental model than SQL — steeper learning curve
- Con: Synchronization between relational and graph layers adds operational complexity
- Con: PostgreSQL recursive CTEs are slower than native graph databases for deep traversals
- Con: Overkill for simple buyer-supplier portals without network/sub-tier analysis needs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 17442 (LEI) | Legal Entity Identifier as a standard node property on organisation nodes, enabling cross-network entity resolution |
| GS1 GLN | Global Location Numbers as node identifiers for locations, enabling EPCIS-compatible supply chain mapping |
| GS1 GTIN | Global Trade Item Numbers as product node identifiers |
| GS1 EPCIS 2.0 | Shipment events create edges in the graph (shipped_from → shipped_to), building the supply chain topology automatically |
| ISO 3166 | Country/region codes as properties on location nodes; used for geopolitical risk propagation queries |
| UNSPSC | Commodity codes as category node properties, enabling capability matching across the supplier network |
| ISO 31000 | Risk assessments propagate through graph edges — a risk at a sub-tier supplier propagates up through supply relationships |
| ISO 20400 | ESG scores as node properties; ESG risk propagation through the supply chain graph |

---

## Graph Layer (Supply Chain Network)

```sql
-- ============================================================
-- GRAPH NODES: Every entity in the supply chain network
-- ============================================================

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(30) NOT NULL,
    -- Node types: 'organisation', 'location', 'product', 'certification',
    --             'commodity', 'risk_signal', 'region', 'logistics_provider'
    external_id     VARCHAR(255),          -- LEI, GLN, GTIN, DUNS, ISO code, etc.
    label           VARCHAR(255) NOT NULL, -- human-readable name
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by node type:
    --
    -- organisation:
    -- {"org_type": "supplier", "employee_count": 250, "annual_revenue": 45000000,
    --  "lei": "549300EXAMPLE000001", "duns": "123456789",
    --  "risk_score": 72, "esg_score": 65, "country": "DE"}
    --
    -- location:
    -- {"gln": "4012345000001", "type": "factory", "address": "...",
    --  "country": "CN", "region": "Guangdong", "lat": 23.1291, "lon": 113.2644}
    --
    -- product:
    -- {"gtin": "00012345600012", "name": "Widget Assembly A",
    --  "commodity_code": "31162800", "critical": true}
    --
    -- certification:
    -- {"standard": "ISO 9001:2015", "body": "TUV", "scope": "manufacturing"}
    --
    -- risk_signal:
    -- {"type": "geopolitical", "severity": "high", "description": "Trade sanctions risk",
    --  "source": "ai_scan", "detected_at": "2026-05-15"}
    --
    tenant_id       UUID,                  -- NULL for shared network nodes
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_type ON graph_nodes(node_type);
CREATE INDEX idx_gn_external ON graph_nodes(external_id) WHERE external_id IS NOT NULL;
CREATE INDEX idx_gn_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_gn_properties ON graph_nodes USING GIN (properties);
CREATE INDEX idx_gn_label ON graph_nodes USING gin (label gin_trgm_ops); -- trigram for fuzzy search

-- ============================================================
-- GRAPH EDGES: Relationships between entities
-- ============================================================

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    edge_type       VARCHAR(40) NOT NULL,
    -- Edge types:
    -- 'supplies_to'         organisation → organisation (buyer-supplier)
    -- 'sub_supplies_to'     organisation → organisation (tier-2+ supplier)
    -- 'manufactures_at'     organisation → location
    -- 'ships_from'          organisation → location
    -- 'ships_to'            location → location (shipment route)
    -- 'produces'            organisation → product
    -- 'requires_component'  product → product (BOM relationship)
    -- 'holds_certification' organisation → certification
    -- 'located_in'          location → region
    -- 'affected_by'         organisation → risk_signal
    -- 'classified_as'       product → commodity
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by edge type:
    --
    -- supplies_to:
    -- {"since": "2020-01-15", "spend_annual_usd": 2500000, "category": "direct",
    --  "commodities": ["31162800"], "lead_time_days": 45, "payment_terms": "NET30"}
    --
    -- requires_component:
    -- {"quantity_per_unit": 4, "critical": true, "alternative_sources": 2}
    --
    -- holds_certification:
    -- {"cert_number": "TUV-2026-12345", "valid_from": "2024-06-01", "valid_to": "2027-05-31"}
    --
    -- affected_by:
    -- {"exposure": "high", "impact_description": "Single source in affected region"}
    --
    weight          NUMERIC(10,4) DEFAULT 1.0, -- for graph algorithms (PageRank, shortest path)
    valid_from      DATE,                  -- temporal edge validity
    valid_to        DATE,
    tenant_id       UUID,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_source ON graph_edges(source_node_id);
CREATE INDEX idx_ge_target ON graph_edges(target_node_id);
CREATE INDEX idx_ge_type ON graph_edges(edge_type);
CREATE INDEX idx_ge_source_type ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_ge_tenant ON graph_edges(tenant_id);
CREATE INDEX idx_ge_properties ON graph_edges USING GIN (properties);
CREATE INDEX idx_ge_temporal ON graph_edges(valid_from, valid_to) WHERE valid_from IS NOT NULL;
```

## Operational Relational Tables

These tables handle day-to-day CRUD operations. They reference graph_nodes for organisation identity but are otherwise standard relational.

```sql
-- ============================================================
-- IDENTITY: Links relational operations to graph nodes
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id   UUID NOT NULL REFERENCES graph_nodes(id), -- link to graph
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    org_type        VARCHAR(20) NOT NULL,
    tax_id          VARCHAR(50),
    lei             VARCHAR(20),
    duns_number     VARCHAR(9),
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(20) NOT NULL DEFAULT 'local',
    auth_subject    VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    role            VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, org_id, role)
);

CREATE TABLE trading_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    graph_edge_id   UUID REFERENCES graph_edges(id),  -- link to graph edge
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_terms   VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (buyer_org_id, supplier_org_id)
);

-- ============================================================
-- TRANSACTIONAL: POs, ASNs, Invoices (standard relational)
-- ============================================================

CREATE TABLE purchase_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    po_number       VARCHAR(50) NOT NULL,
    order_date      DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'issued',
    currency_code   CHAR(3) NOT NULL,
    grand_total     NUMERIC(15,2) NOT NULL DEFAULT 0,
    payment_terms   VARCHAR(20),
    requested_delivery_date DATE,
    source_format   VARCHAR(10),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (buyer_org_id, po_number)
);

CREATE INDEX idx_po_buyer ON purchase_orders(buyer_org_id);
CREATE INDEX idx_po_supplier ON purchase_orders(supplier_org_id);
CREATE INDEX idx_po_status ON purchase_orders(status);

CREATE TABLE po_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_id           UUID NOT NULL REFERENCES purchase_orders(id) ON DELETE CASCADE,
    line_number     INTEGER NOT NULL,
    product_node_id UUID REFERENCES graph_nodes(id), -- link to product in graph
    item_number     VARCHAR(50),
    description     VARCHAR(500) NOT NULL,
    uom             VARCHAR(10) NOT NULL,
    quantity        NUMERIC(12,4) NOT NULL,
    unit_price      NUMERIC(15,4) NOT NULL,
    line_total      NUMERIC(15,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (po_id, line_number)
);

CREATE TABLE ship_notices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_id           UUID NOT NULL REFERENCES purchase_orders(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    asn_number      VARCHAR(50) NOT NULL,
    ship_date       DATE NOT NULL,
    estimated_delivery DATE,
    actual_delivery DATE,
    carrier_name    VARCHAR(100),
    tracking_number VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'shipped',
    origin_location_node_id UUID REFERENCES graph_nodes(id),   -- graph: ship-from location
    dest_location_node_id   UUID REFERENCES graph_nodes(id),   -- graph: ship-to location
    shipment_data   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sn_po ON ship_notices(po_id);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    po_id           UUID REFERENCES purchase_orders(id),
    invoice_number  VARCHAR(50) NOT NULL,
    invoice_date    DATE NOT NULL,
    due_date        DATE,
    currency_code   CHAR(3) NOT NULL,
    grand_total     NUMERIC(15,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'submitted',
    lines           JSONB NOT NULL DEFAULT '[]',
    match_data      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_org_id, invoice_number)
);

CREATE INDEX idx_inv_buyer ON invoices(buyer_org_id);
CREATE INDEX idx_inv_supplier ON invoices(supplier_org_id);
CREATE INDEX idx_inv_status ON invoices(status);

-- ============================================================
-- PERFORMANCE & RISK (relational with graph references)
-- ============================================================

CREATE TABLE supplier_scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    overall_score   NUMERIC(5,2),
    overall_rating  VARCHAR(10),
    scores          JSONB NOT NULL DEFAULT '{}',
    published       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sc_supplier ON supplier_scorecards(supplier_org_id);

CREATE TABLE corrective_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trading_rel_id  UUID NOT NULL REFERENCES trading_relationships(id),
    po_id           UUID REFERENCES purchase_orders(id),
    capa_number     VARCHAR(30) NOT NULL,
    issue_type      VARCHAR(30) NOT NULL,
    severity        VARCHAR(10) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    description     TEXT NOT NULL,
    workflow_data   JSONB NOT NULL DEFAULT '{}',
    due_date        DATE,
    assigned_to     UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);
```

---

## Graph Query Examples

### Find all sub-tier suppliers (recursive, any depth)

```sql
-- "Show me every supplier in the supply chain for Buyer X, including sub-tiers"
WITH RECURSIVE supply_chain AS (
    -- Start: direct suppliers of the buyer
    SELECT
        e.target_node_id AS supplier_node_id,
        n.label AS supplier_name,
        1 AS tier,
        ARRAY[e.source_node_id, e.target_node_id] AS path
    FROM graph_edges e
    JOIN graph_nodes n ON n.id = e.target_node_id
    WHERE e.source_node_id = :buyer_graph_node_id
      AND e.edge_type = 'supplies_to'
      AND e.status = 'active'

    UNION ALL

    -- Recurse: each supplier's suppliers
    SELECT
        e.target_node_id,
        n.label,
        sc.tier + 1,
        sc.path || e.target_node_id
    FROM graph_edges e
    JOIN graph_nodes n ON n.id = e.target_node_id
    JOIN supply_chain sc ON sc.supplier_node_id = e.source_node_id
    WHERE e.edge_type IN ('supplies_to', 'sub_supplies_to')
      AND e.status = 'active'
      AND e.target_node_id <> ALL(sc.path)  -- prevent cycles
      AND sc.tier < 5                        -- max depth
)
SELECT tier, supplier_name, supplier_node_id
FROM supply_chain
ORDER BY tier, supplier_name;
```

### Risk propagation — which buyers are exposed to a risk signal?

```sql
-- "If supplier X is affected by a risk, which buyers are impacted?"
WITH RECURSIVE risk_propagation AS (
    -- Start: the organisation affected by the risk signal
    SELECT
        e.source_node_id AS affected_node_id,
        n.label AS affected_name,
        n.properties->>'org_type' AS affected_type,
        0 AS hops
    FROM graph_edges e
    JOIN graph_nodes n ON n.id = e.source_node_id
    WHERE e.target_node_id = :risk_signal_node_id
      AND e.edge_type = 'affected_by'

    UNION ALL

    -- Propagate upstream: who buys from the affected organisation?
    SELECT
        e.source_node_id,
        n.label,
        n.properties->>'org_type',
        rp.hops + 1
    FROM graph_edges e
    JOIN graph_nodes n ON n.id = e.source_node_id
    JOIN risk_propagation rp ON rp.affected_node_id = e.target_node_id
    WHERE e.edge_type IN ('supplies_to', 'sub_supplies_to')
      AND e.status = 'active'
      AND rp.hops < 4
)
SELECT DISTINCT affected_name, affected_type, hops
FROM risk_propagation
WHERE affected_type = 'buyer'
ORDER BY hops;
```

### Alternative supplier discovery via capability graph

```sql
-- "Find alternative suppliers that produce the same product and hold required certifications"
WITH current_supplier AS (
    SELECT e.source_node_id AS supplier_node
    FROM graph_edges e
    WHERE e.target_node_id = :product_node_id
      AND e.edge_type = 'produces'
      AND e.source_node_id = :current_supplier_node_id
),
required_certs AS (
    SELECT e.target_node_id AS cert_node
    FROM graph_edges e
    WHERE e.source_node_id = :current_supplier_node_id
      AND e.edge_type = 'holds_certification'
      AND e.status = 'active'
),
alternatives AS (
    SELECT DISTINCT e.source_node_id AS alt_supplier_node
    FROM graph_edges e
    WHERE e.target_node_id = :product_node_id
      AND e.edge_type = 'produces'
      AND e.source_node_id <> :current_supplier_node_id
      AND e.status = 'active'
)
SELECT
    n.label AS supplier_name,
    n.properties->>'country' AS country,
    n.properties->>'risk_score' AS risk_score,
    n.properties->>'esg_score' AS esg_score,
    (SELECT COUNT(*)
     FROM graph_edges ce
     WHERE ce.source_node_id = a.alt_supplier_node
       AND ce.edge_type = 'holds_certification'
       AND ce.target_node_id IN (SELECT cert_node FROM required_certs)
    ) AS matching_certs,
    (SELECT COUNT(*) FROM required_certs) AS required_certs_total
FROM alternatives a
JOIN graph_nodes n ON n.id = a.alt_supplier_node
WHERE n.status = 'active'
ORDER BY matching_certs DESC, (n.properties->>'risk_score')::numeric ASC;
```

### Shared sub-tier supplier detection

```sql
-- "Do any of my suppliers share a common sub-tier supplier?"
SELECT
    n1.label AS supplier_a,
    n2.label AS supplier_b,
    shared.label AS shared_sub_tier,
    shared.properties->>'country' AS sub_tier_country
FROM graph_edges e1
JOIN graph_edges e2 ON e1.target_node_id = e2.target_node_id  -- same sub-tier
                   AND e1.source_node_id <> e2.source_node_id  -- different tier-1
JOIN graph_nodes n1 ON n1.id = e1.source_node_id
JOIN graph_nodes n2 ON n2.id = e2.source_node_id
JOIN graph_nodes shared ON shared.id = e1.target_node_id
WHERE e1.edge_type IN ('supplies_to', 'sub_supplies_to')
  AND e2.edge_type IN ('supplies_to', 'sub_supplies_to')
  AND e1.source_node_id IN (
      -- my tier-1 suppliers
      SELECT target_node_id FROM graph_edges
      WHERE source_node_id = :buyer_graph_node_id
        AND edge_type = 'supplies_to'
  )
  AND e2.source_node_id IN (
      SELECT target_node_id FROM graph_edges
      WHERE source_node_id = :buyer_graph_node_id
        AND edge_type = 'supplies_to'
  );
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Identity | 4 | organisations, users, org_memberships, trading_relationships (linked to graph) |
| Purchase Orders | 2 | headers + lines (lines reference product graph nodes) |
| Ship Notices | 1 | references location graph nodes for origin/destination |
| Invoices | 1 | standard relational with JSONB lines |
| Scorecards | 1 | scores in JSONB |
| Corrective Actions | 1 | workflow in JSONB |
| Audit | 1 | standard append-only |
| **Total** | **13** | Plus the graph layer replaces many relationship tables |

---

## Key Design Decisions

1. **Graph layer as PostgreSQL tables, not a separate database** — using `graph_nodes` and `graph_edges` tables in the same PostgreSQL instance avoids the operational complexity of synchronizing a separate graph database. PostgreSQL recursive CTEs handle most supply chain graph queries adequately for networks up to ~100K nodes. If the network grows beyond that, Apache AGE (PostgreSQL extension for Cypher queries) or a dedicated Neo4j instance can be introduced with the same logical model.

2. **Bidirectional graph-relational links** — `organisations.graph_node_id` links relational operations to the graph, while `trading_relationships.graph_edge_id` links to the graph edge. This allows operational queries (PO lookup) to use relational JOINs while network analysis queries (sub-tier discovery) use graph traversal. Neither layer duplicates the other's data.

3. **Product nodes in the graph** — PO lines reference `graph_nodes` for products. This enables BOM (Bill of Materials) traversal through the graph: "which raw materials go into this finished product, and which suppliers provide them?" This query is impractical in a pure relational model but natural in a graph.

4. **Location nodes for supply chain mapping** — ship notices reference origin and destination location nodes. Over time, these edges build a complete supply chain topology: "which factories ship to which warehouses via which routes?" EPCIS events create these edges automatically.

5. **Risk signals as graph nodes** — rather than storing risk assessments as rows in a relational table, risk signals are nodes connected to affected organisations via `affected_by` edges. This enables risk propagation queries: "if this region has a natural disaster, which suppliers are affected, and which buyers depend on those suppliers?"

6. **Temporal edges with `valid_from` / `valid_to`** — supply chain relationships change over time. Temporal edges allow historical queries ("who supplied us in Q3 2025?") without deleting edges. Active edges have `valid_to IS NULL`.

7. **Edge weight for graph algorithms** — the `weight` column enables weighted graph algorithms. For example, weighting `supplies_to` edges by annual spend allows shortest-path algorithms to find the most critical supply chain dependencies, or PageRank to identify the most strategically important suppliers in the network.

8. **Tenant-scoped and shared nodes** — some graph nodes (major suppliers, standard certifications, countries) are shared across tenants (`tenant_id IS NULL`). Buyer-specific nodes are tenant-scoped. This enables cross-network intelligence while maintaining data isolation for operational data.

9. **JSONB properties on nodes and edges** — rather than separate attribute tables per node type, all variable properties go in JSONB. GIN indexes enable efficient property-based filtering within graph queries.

10. **Graph enables AI-native features** — graph embeddings (Node2Vec, GraphSAGE) trained on the supply chain graph can power supplier recommendation, risk prediction, and anomaly detection. The graph structure is the ideal input for these ML models, providing richer signal than tabular data alone.
