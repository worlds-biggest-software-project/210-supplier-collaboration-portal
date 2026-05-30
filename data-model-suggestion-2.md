# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Supplier Collaboration Portal · Created: 2026-05-20

## Philosophy

This model treats every change to every business object as an immutable event stored in an append-only event store. The event store is the single source of truth. Materialized read models (projections) are built from the event stream to serve queries — purchase order status, invoice dashboards, scorecard reports — but these views are derived, not canonical.

This pattern is used in financial systems (double-entry ledgers are inherently event-sourced), compliance-heavy platforms, and supply chain visibility systems where the sequence of events matters as much as the current state. GS1 EPCIS is itself an event-sourced standard — it records "what happened to this product" as a chain of events, not a mutable row.

In a supplier collaboration portal, event sourcing provides a complete, tamper-evident audit trail of every PO change, every invoice status transition, every acknowledgement and shipment event. This is critical for dispute resolution ("who changed the quantity and when?"), regulatory compliance (GDPR right to explanation, SOX audit requirements), and AI-powered analytics that learn from the full history of supplier interactions rather than just current snapshots.

**Best for:** Organisations requiring complete audit trails, temporal queries ("what was the PO status on March 15th?"), dispute resolution evidence, and AI analytics over change patterns.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every change is preserved forever
- Pro: Temporal queries are native — reconstruct state at any point in time
- Pro: Natural fit for AI analytics over event sequences and patterns
- Pro: Event replay enables safe schema evolution and new projections
- Pro: Aligns naturally with GS1 EPCIS event model and EDI transaction logging
- Con: Higher storage requirements (events accumulate indefinitely)
- Con: Eventual consistency between event store and read models
- Con: More complex to implement — requires event handlers, projectors, and snapshot strategies
- Con: Simple CRUD queries require read model infrastructure
- Con: Debugging requires understanding event replay, not just "look at the row"

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 | The event store design is directly compatible with EPCIS event semantics — each shipment event is stored as-is in the event stream with EPCIS fields |
| ANSI X12 850/855/856/810 | Each inbound/outbound EDI document is stored as a raw event; projections materialize the current PO/invoice state |
| cXML 1.2 | Inbound cXML documents stored as events with the raw payload preserved alongside extracted structured fields |
| ISO 31000 | Risk events (score changes, alert triggers, mitigation actions) stored in the event stream, enabling temporal risk analysis |
| ISO 20400 | ESG assessment events enable tracking sustainability score evolution over time |
| OCSF (Open Cybersecurity Schema Framework) | Audit events follow OCSF-like categorization for structured security event logging |

---

## Event Store (Source of Truth)

```sql
-- The central event store: append-only, immutable
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,         -- aggregate root ID (e.g., PO ID, invoice ID)
    stream_type     VARCHAR(50) NOT NULL,  -- 'PurchaseOrder', 'Invoice', 'ShipNotice', 'Supplier', 'Scorecard'
    event_type      VARCHAR(80) NOT NULL,  -- e.g., 'PurchaseOrderIssued', 'InvoiceSubmitted', 'POLineQuantityChanged'
    event_version   INTEGER NOT NULL,      -- monotonically increasing per stream
    data            JSONB NOT NULL,        -- event payload
    metadata        JSONB NOT NULL DEFAULT '{}', -- actor, IP, correlation_id, causation_id
    tenant_id       UUID NOT NULL,         -- buyer org for multi-tenancy
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)      -- optimistic concurrency control
);

-- Primary query path: replay events for a specific aggregate
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
-- Cross-stream queries: all events of a type within a tenant
CREATE INDEX idx_events_tenant_type ON events(tenant_id, event_type, created_at);
-- Global ordering for projections
CREATE INDEX idx_events_created ON events(created_at);
-- Catch-up subscriptions
CREATE INDEX idx_events_type_created ON events(event_type, created_at);

-- Example event payloads:
--
-- PurchaseOrderIssued:
-- {
--   "po_number": "PO-2026-00142",
--   "buyer_org_id": "...",
--   "supplier_org_id": "...",
--   "currency": "USD",
--   "lines": [
--     {"line_number": 1, "item": "WIDGET-A", "qty": 1000, "unit_price": 12.50, "uom": "EA"},
--     {"line_number": 2, "item": "GASKET-B", "qty": 500, "unit_price": 3.25, "uom": "EA"}
--   ],
--   "ship_to": {"line_1": "123 Factory Rd", "city": "Detroit", "country": "US"},
--   "requested_delivery": "2026-06-15",
--   "source_format": "portal"
-- }
--
-- POLineQuantityChanged:
-- {
--   "line_number": 1,
--   "old_quantity": 1000,
--   "new_quantity": 800,
--   "reason": "demand_reduction",
--   "change_request_ref": "CR-2026-0031"
-- }
--
-- InvoiceThreeWayMatched:
-- {
--   "invoice_id": "...",
--   "po_id": "...",
--   "match_result": "matched",
--   "price_variance": 0.00,
--   "quantity_variance": 0,
--   "matched_by": "ai"
-- }

-- Snapshots for performance (avoid replaying thousands of events)
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,     -- event_version at snapshot time
    state           JSONB NOT NULL,        -- full aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

## Event Type Registry

```sql
-- Documents all known event types for validation and schema evolution
CREATE TABLE event_type_registry (
    event_type      VARCHAR(80) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB,                 -- JSON Schema for the event data payload
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed data examples:
-- ('PurchaseOrderIssued', 'PurchaseOrder', 'A new PO was created and sent to the supplier', 1, ...)
-- ('PurchaseOrderAcknowledged', 'PurchaseOrder', 'Supplier acknowledged the PO', 1, ...)
-- ('POLineQuantityChanged', 'PurchaseOrder', 'Buyer changed the quantity on a PO line', 1, ...)
-- ('ShipNoticeCreated', 'ShipNotice', 'Supplier created an ASN', 1, ...)
-- ('InvoiceSubmitted', 'Invoice', 'Supplier submitted an invoice', 1, ...)
-- ('InvoiceThreeWayMatched', 'Invoice', 'Invoice matched against PO and GRN', 1, ...)
-- ('SupplierRiskScoreChanged', 'Supplier', 'AI risk monitor updated a supplier risk score', 1, ...)
-- ('ScorecardPublished', 'Scorecard', 'Buyer published a scorecard to supplier', 1, ...)
```

## Identity Tables (Mutable — not event-sourced)

```sql
-- These are mutable reference/identity tables, not event-sourced.
-- User and org data changes infrequently and needs direct lookup.

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    org_type        VARCHAR(20) NOT NULL CHECK (org_type IN ('buyer', 'supplier', 'both')),
    tax_id          VARCHAR(50),
    lei             VARCHAR(20),
    duns_number     VARCHAR(9),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
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
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_terms   VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (buyer_org_id, supplier_org_id)
);
```

## Read Models (Materialized Projections)

These tables are rebuilt from the event stream. They can be dropped and recreated at any time.

```sql
-- ============================================================
-- PROJECTION: Current Purchase Order State
-- Built from: PurchaseOrderIssued, POAcknowledged, POLineQuantityChanged,
--             POCancelled, POClosed, etc.
-- ============================================================

CREATE TABLE rm_purchase_orders (
    id              UUID PRIMARY KEY,      -- same as stream_id
    buyer_org_id    UUID NOT NULL,
    supplier_org_id UUID NOT NULL,
    po_number       VARCHAR(50) NOT NULL,
    order_date      DATE NOT NULL,
    status          VARCHAR(20) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    grand_total     NUMERIC(15,2) NOT NULL,
    line_count      INTEGER NOT NULL DEFAULT 0,
    requested_delivery_date DATE,
    acknowledged_at TIMESTAMPTZ,
    last_change_at  TIMESTAMPTZ,
    source_format   VARCHAR(10),
    event_version   INTEGER NOT NULL,      -- tracks which event this projection is current to
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_po_buyer ON rm_purchase_orders(buyer_org_id);
CREATE INDEX idx_rm_po_supplier ON rm_purchase_orders(supplier_org_id);
CREATE INDEX idx_rm_po_status ON rm_purchase_orders(status);

CREATE TABLE rm_po_lines (
    id              UUID PRIMARY KEY,
    po_id           UUID NOT NULL REFERENCES rm_purchase_orders(id),
    line_number     INTEGER NOT NULL,
    item_number     VARCHAR(50),
    description     VARCHAR(500),
    quantity        NUMERIC(12,4) NOT NULL,
    unit_price      NUMERIC(15,4) NOT NULL,
    line_total      NUMERIC(15,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    shipped_quantity NUMERIC(12,4) DEFAULT 0,
    invoiced_quantity NUMERIC(12,4) DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Current Invoice State
-- Built from: InvoiceSubmitted, InvoiceApproved, InvoiceDisputed,
--             InvoicePaid, InvoiceThreeWayMatched, etc.
-- ============================================================

CREATE TABLE rm_invoices (
    id              UUID PRIMARY KEY,
    supplier_org_id UUID NOT NULL,
    buyer_org_id    UUID NOT NULL,
    po_id           UUID,
    invoice_number  VARCHAR(50) NOT NULL,
    invoice_date    DATE NOT NULL,
    due_date        DATE,
    grand_total     NUMERIC(15,2) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    match_result    VARCHAR(20),
    paid_date       DATE,
    paid_amount     NUMERIC(15,2),
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_inv_buyer ON rm_invoices(buyer_org_id);
CREATE INDEX idx_rm_inv_supplier ON rm_invoices(supplier_org_id);
CREATE INDEX idx_rm_inv_status ON rm_invoices(status);

-- ============================================================
-- PROJECTION: Current Ship Notice State
-- Built from: ShipNoticeCreated, ShipmentInTransit, ShipmentDelivered, etc.
-- ============================================================

CREATE TABLE rm_ship_notices (
    id              UUID PRIMARY KEY,
    po_id           UUID NOT NULL,
    supplier_org_id UUID NOT NULL,
    asn_number      VARCHAR(50) NOT NULL,
    ship_date       DATE NOT NULL,
    estimated_delivery DATE,
    actual_delivery DATE,
    carrier_name    VARCHAR(100),
    tracking_number VARCHAR(100),
    status          VARCHAR(20) NOT NULL,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Supplier Risk Dashboard
-- Built from: SupplierRiskScoreChanged, RiskAlertTriggered, RiskAlertAcknowledged
-- ============================================================

CREATE TABLE rm_supplier_risk (
    supplier_org_id UUID PRIMARY KEY,
    overall_risk_score NUMERIC(5,2),
    overall_risk_level VARCHAR(10),
    financial_score NUMERIC(5,2),
    geopolitical_score NUMERIC(5,2),
    operational_score NUMERIC(5,2),
    esg_score       NUMERIC(5,2),
    compliance_score NUMERIC(5,2),
    open_alerts     INTEGER DEFAULT 0,
    last_assessed_at TIMESTAMPTZ,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Supplier Scorecard History
-- Built from: ScorecardCreated, ScorecardScored, ScorecardPublished
-- ============================================================

CREATE TABLE rm_supplier_scorecards (
    id              UUID PRIMARY KEY,
    buyer_org_id    UUID NOT NULL,
    supplier_org_id UUID NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    overall_score   NUMERIC(5,2),
    overall_rating  VARCHAR(10),
    on_time_delivery NUMERIC(5,2),
    quality_reject_rate NUMERIC(5,2),
    responsiveness_score NUMERIC(5,2),
    cost_score      NUMERIC(5,2),
    esg_score       NUMERIC(5,2),
    published       BOOLEAN DEFAULT false,
    event_version   INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_sc_supplier ON rm_supplier_scorecards(supplier_org_id);
CREATE INDEX idx_rm_sc_period ON rm_supplier_scorecards(period_start);
```

## Outbox Pattern (Reliable Event Publishing)

```sql
-- Transactional outbox for reliable event publishing to message brokers
CREATE TABLE event_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(event_id),
    destination     VARCHAR(50) NOT NULL,  -- 'kafka', 'webhook', 'email_notification'
    payload         JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'published', 'failed')),
    retry_count     INTEGER NOT NULL DEFAULT 0,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at    TIMESTAMPTZ
);

CREATE INDEX idx_outbox_pending ON event_outbox(status, created_at) WHERE status = 'pending';
```

## Raw Document Archive

```sql
-- Preserves the raw EDI/cXML/PDF documents that generated events
CREATE TABLE raw_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID REFERENCES events(event_id),
    format          VARCHAR(20) NOT NULL CHECK (format IN ('cxml', 'edi_x12', 'edi_edifact', 'pdf', 'json', 'csv')),
    content_type    VARCHAR(100) NOT NULL,
    raw_content     BYTEA,                 -- for small documents
    storage_url     VARCHAR(500),          -- for large documents (S3, etc.)
    file_hash       VARCHAR(64) NOT NULL,  -- SHA-256 for integrity verification
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_raw_docs_event ON raw_documents(event_id);
```

## Example Queries

### Reconstruct PO state at a specific point in time

```sql
-- "What was PO abc123's state on March 15th 2026?"
SELECT event_type, data, created_at
FROM events
WHERE stream_id = 'abc123'
  AND stream_type = 'PurchaseOrder'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;
-- Application replays these events to reconstruct the state
```

### Find all quantity changes for a supplier in the last 90 days

```sql
SELECT e.stream_id AS po_id,
       e.data->>'line_number' AS line,
       e.data->>'old_quantity' AS old_qty,
       e.data->>'new_quantity' AS new_qty,
       e.data->>'reason' AS reason,
       e.created_at
FROM events e
WHERE e.tenant_id = :buyer_org_id
  AND e.event_type = 'POLineQuantityChanged'
  AND e.created_at >= now() - INTERVAL '90 days'
ORDER BY e.created_at DESC;
```

### AI analytics: supplier response time distribution

```sql
-- Time between PO issuance and acknowledgement per supplier
WITH issued AS (
    SELECT stream_id, created_at AS issued_at,
           data->>'supplier_org_id' AS supplier_id
    FROM events
    WHERE event_type = 'PurchaseOrderIssued'
      AND tenant_id = :buyer_org_id
),
acked AS (
    SELECT stream_id, created_at AS acked_at
    FROM events
    WHERE event_type = 'PurchaseOrderAcknowledged'
      AND tenant_id = :buyer_org_id
)
SELECT i.supplier_id,
       AVG(EXTRACT(EPOCH FROM (a.acked_at - i.issued_at)) / 3600) AS avg_hours_to_ack,
       PERCENTILE_CONT(0.95) WITHIN GROUP (
           ORDER BY EXTRACT(EPOCH FROM (a.acked_at - i.issued_at)) / 3600
       ) AS p95_hours_to_ack,
       COUNT(*) AS po_count
FROM issued i
JOIN acked a ON a.stream_id = i.stream_id
GROUP BY i.supplier_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | events, snapshots, event_type_registry |
| Identity (Mutable) | 4 | organisations, users, org_memberships, trading_relationships |
| Read Model: Purchase Orders | 2 | rm_purchase_orders, rm_po_lines |
| Read Model: Invoices | 1 | rm_invoices |
| Read Model: Ship Notices | 1 | rm_ship_notices |
| Read Model: Risk | 1 | rm_supplier_risk |
| Read Model: Scorecards | 1 | rm_supplier_scorecards |
| Infrastructure | 2 | event_outbox, raw_documents |
| **Total** | **15** | Plus additional read models as needed |

---

## Key Design Decisions

1. **Single `events` table rather than one table per stream type** — simplifies the event store, enables cross-stream queries, and avoids table proliferation. The `stream_type` column partitions logically. For very high volumes, PostgreSQL table partitioning on `created_at` or `stream_type` can be applied later.

2. **Optimistic concurrency via `(stream_id, event_version)` unique constraint** — prevents lost updates when two users try to modify the same PO simultaneously. The application must retry on conflict.

3. **JSONB event payloads with a JSON Schema registry** — balances flexibility (events can evolve without DDL) with validation (the event_type_registry holds JSON Schema definitions that the application layer validates against before writing).

4. **Identity tables are mutable, not event-sourced** — user and org data changes too infrequently to justify the complexity of event sourcing. These tables are queried by FK from read models.

5. **Read models are disposable and rebuildable** — the `rm_` prefix signals that these tables are projections. If a projection becomes corrupt or a new reporting dimension is needed, the projection can be dropped and rebuilt from the event stream.

6. **Raw document archive** — preserves the original cXML/EDI/PDF that generated events. This is critical for EDI dispute resolution ("show me the original 856 that the supplier sent") and for AI training on document extraction.

7. **Transactional outbox pattern** — events are written to both the `events` table and the `event_outbox` in the same database transaction. A separate publisher process reads the outbox and publishes to Kafka/webhooks, ensuring at-least-once delivery without distributed transactions.

8. **Snapshots for performance** — for aggregates with hundreds of events (e.g., a PO with many change orders), the application takes periodic snapshots to avoid replaying the full event history on every read. Snapshots are versioned so stale snapshots are detected.

9. **Temporal queries are first-class** — the event store naturally answers "what was true at time T?" by replaying events up to that timestamp. This is critical for dispute resolution, regulatory reporting, and AI model training on historical patterns.

10. **Event-native AI analytics** — the event stream is the ideal training data source for ML models that predict supplier behavior (e.g., "suppliers who acknowledge within 4 hours have 15% fewer delivery issues"). No ETL pipeline is needed — the events are already structured, timestamped, and complete.
