# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Supplier Collaboration Portal · Created: 2026-05-20

## Philosophy

This model follows classical third-normal-form (3NF) relational design, with a dedicated table for every business concept and explicit foreign key relationships between them. Every entity — organisations, purchase orders, invoices, shipments, scorecards — has its own table with strongly typed columns and referential integrity enforced at the database level.

This approach mirrors how enterprise ERP systems (SAP S/4HANA, Oracle Fusion) model procurement data internally. It provides the highest degree of data integrity, supports complex cross-entity reporting queries without denormalization, and maps cleanly onto the ANSI X12 EDI document model (850/855/856/810) and cXML schemas where each document type has a well-defined structure.

The trade-off is rigidity: adding new fields requires schema migrations, and jurisdiction-specific or supplier-category-specific attributes need either nullable columns or extension tables. This model is best suited for teams with strong database engineering skills building a product with a well-defined, stable domain model.

**Best for:** Organisations that need the highest data integrity, complex cross-entity SQL reporting, and alignment with established EDI/cXML document structures.

**Trade-offs:**
- Pro: Maximum data integrity via foreign keys and constraints
- Pro: Excellent for complex analytical queries and reporting
- Pro: Clear mapping to EDI/cXML document types
- Pro: Well-understood by database engineers; mature tooling
- Con: Schema migrations required for new fields or entity types
- Con: High table count increases join complexity
- Con: Jurisdiction-specific or category-specific fields are awkward (nullable columns or extension tables)
- Con: Less agile for rapid MVP iteration

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ANSI X12 850/855/856/810 | Each EDI document type maps directly to a table pair (header + lines): `purchase_orders`/`po_lines`, `po_acknowledgements`/`po_ack_lines`, `ship_notices`/`sn_lines`, `invoices`/`invoice_lines` |
| cXML 1.2 | OrderRequest, ConfirmationRequest, ShipNoticeRequest, InvoiceDetailRequest map to the same table structures with cXML-specific fields (e.g., `cxml_payload_id`) |
| ISO 3166-1/2 | `countries` and `country_subdivisions` reference tables for jurisdiction modeling |
| ISO 4217 | `currencies` reference table; all monetary columns paired with a `currency_code` FK |
| ISO 20400 | Sustainability scorecard criteria stored in `scorecard_templates` / `scorecard_criteria` tables |
| ISO 31000 | Risk categories and assessment framework modeled in `risk_categories` / `risk_assessments` tables |
| GS1 EPCIS 2.0 | Shipment events stored in `shipment_events` table with EPCIS-compatible event fields (what/when/where/why) |
| UN/CEFACT CII | Invoice fields aligned with EN 16931 semantic model for Peppol compatibility |

---

## Core Identity & Multi-Tenancy

```sql
-- Row-level tenancy via org_id on all transactional tables

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    org_type        VARCHAR(20) NOT NULL CHECK (org_type IN ('buyer', 'supplier', 'both')),
    tax_id          VARCHAR(50),           -- EIN, VAT number, etc.
    lei             VARCHAR(20),           -- ISO 17442 Legal Entity Identifier
    duns_number     VARCHAR(9),            -- D-U-N-S Number
    website         VARCHAR(500),
    logo_url        VARCHAR(500),
    default_currency_code CHAR(3) REFERENCES currencies(code),
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'deactivated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_addresses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    address_type    VARCHAR(20) NOT NULL CHECK (address_type IN ('billing', 'shipping', 'registered', 'remittance')),
    line_1          VARCHAR(255) NOT NULL,
    line_2          VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,      -- ISO 3166-1 alpha-2
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_addresses_org ON org_addresses(org_id);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    auth_provider   VARCHAR(20) NOT NULL DEFAULT 'local' CHECK (auth_provider IN ('local', 'saml', 'oidc')),
    auth_subject    VARCHAR(255),          -- external IdP subject identifier
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    role            VARCHAR(30) NOT NULL CHECK (role IN ('admin', 'buyer', 'approver', 'supplier_admin', 'supplier_user', 'quality_engineer', 'viewer')),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, org_id, role)
);

CREATE INDEX idx_org_memberships_user ON org_memberships(user_id);
CREATE INDEX idx_org_memberships_org ON org_memberships(org_id);
```

## Buyer-Supplier Relationships

```sql
CREATE TABLE trading_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'suspended', 'terminated')),
    onboarded_at    TIMESTAMPTZ,
    payment_terms   VARCHAR(20),           -- e.g., 'NET30', 'NET60'
    default_currency_code CHAR(3) REFERENCES currencies(code),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (buyer_org_id, supplier_org_id)
);

CREATE INDEX idx_trading_rel_buyer ON trading_relationships(buyer_org_id);
CREATE INDEX idx_trading_rel_supplier ON trading_relationships(supplier_org_id);
```

## Supplier Profile & Documents

```sql
CREATE TABLE supplier_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) UNIQUE,
    business_type   VARCHAR(50),           -- e.g., 'manufacturer', 'distributor', 'services'
    employee_count_range VARCHAR(20),
    annual_revenue_range VARCHAR(20),
    year_established INTEGER,
    minority_owned  BOOLEAN,
    women_owned     BOOLEAN,
    veteran_owned   BOOLEAN,
    small_business  BOOLEAN,
    naics_codes     VARCHAR(50)[],         -- North American Industry Classification
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    document_type   VARCHAR(30) NOT NULL CHECK (document_type IN ('iso_cert', 'insurance', 'w9', 'w8ben', 'bank_details', 'nda', 'cod_of_conduct', 'esg_report', 'audit_report', 'other')),
    title           VARCHAR(255) NOT NULL,
    file_url        VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    expires_at      DATE,
    verified        BOOLEAN NOT NULL DEFAULT false,
    verified_by     UUID REFERENCES users(id),
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_docs_org ON supplier_documents(org_id);
CREATE INDEX idx_supplier_docs_expiry ON supplier_documents(expires_at) WHERE expires_at IS NOT NULL;
```

## Purchase Orders (EDI 850 / cXML OrderRequest)

```sql
CREATE TABLE purchase_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    po_number       VARCHAR(50) NOT NULL,
    order_date      DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'issued' CHECK (status IN ('draft', 'issued', 'acknowledged', 'partially_shipped', 'shipped', 'invoiced', 'closed', 'cancelled')),
    currency_code   CHAR(3) NOT NULL,      -- ISO 4217
    subtotal        NUMERIC(15,2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(15,2) NOT NULL DEFAULT 0,
    shipping_total  NUMERIC(15,2) NOT NULL DEFAULT 0,
    grand_total     NUMERIC(15,2) NOT NULL DEFAULT 0,
    payment_terms   VARCHAR(20),
    ship_to_address_id UUID REFERENCES org_addresses(id),
    bill_to_address_id UUID REFERENCES org_addresses(id),
    requested_delivery_date DATE,
    notes           TEXT,
    cxml_payload_id VARCHAR(255),          -- cXML PayloadID for traceability
    edi_control_number VARCHAR(20),        -- X12 ISA control number
    source_format   VARCHAR(10) CHECK (source_format IN ('portal', 'cxml', 'edi_x12', 'edi_edifact', 'api')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (buyer_org_id, po_number)
);

CREATE INDEX idx_po_buyer ON purchase_orders(buyer_org_id);
CREATE INDEX idx_po_supplier ON purchase_orders(supplier_org_id);
CREATE INDEX idx_po_status ON purchase_orders(status);
CREATE INDEX idx_po_order_date ON purchase_orders(order_date);

CREATE TABLE po_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_id           UUID NOT NULL REFERENCES purchase_orders(id),
    line_number     INTEGER NOT NULL,
    item_number     VARCHAR(50),           -- buyer's part number
    supplier_part_number VARCHAR(50),
    description     VARCHAR(500) NOT NULL,
    uom             VARCHAR(10) NOT NULL,  -- unit of measure (EA, KG, etc.)
    quantity        NUMERIC(12,4) NOT NULL,
    unit_price      NUMERIC(15,4) NOT NULL,
    line_total      NUMERIC(15,2) NOT NULL,
    requested_delivery_date DATE,
    commodity_code  VARCHAR(20),           -- UNSPSC or custom category
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (po_id, line_number)
);

CREATE INDEX idx_po_lines_po ON po_lines(po_id);
```

## PO Acknowledgements (EDI 855 / cXML ConfirmationRequest)

```sql
CREATE TABLE po_acknowledgements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_id           UUID NOT NULL REFERENCES purchase_orders(id),
    ack_date        TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_status  VARCHAR(20) NOT NULL CHECK (overall_status IN ('accepted', 'accepted_with_changes', 'rejected', 'pending')),
    supplier_reference VARCHAR(50),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE po_ack_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ack_id          UUID NOT NULL REFERENCES po_acknowledgements(id),
    po_line_id      UUID NOT NULL REFERENCES po_lines(id),
    line_status     VARCHAR(20) NOT NULL CHECK (line_status IN ('accepted', 'backordered', 'rejected', 'substituted', 'quantity_changed', 'date_changed')),
    confirmed_quantity NUMERIC(12,4),
    confirmed_unit_price NUMERIC(15,4),
    promised_delivery_date DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_po_ack_lines_ack ON po_ack_lines(ack_id);
```

## Advance Ship Notices (EDI 856 / cXML ShipNoticeRequest)

```sql
CREATE TABLE ship_notices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_id           UUID NOT NULL REFERENCES purchase_orders(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    asn_number      VARCHAR(50) NOT NULL,
    ship_date       DATE NOT NULL,
    estimated_delivery_date DATE,
    actual_delivery_date DATE,
    carrier_name    VARCHAR(100),
    tracking_number VARCHAR(100),
    shipping_method VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'shipped' CHECK (status IN ('shipped', 'in_transit', 'delivered', 'exception')),
    total_weight    NUMERIC(12,4),
    weight_uom      VARCHAR(5),            -- KG, LB
    total_packages  INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sn_po ON ship_notices(po_id);
CREATE INDEX idx_sn_supplier ON ship_notices(supplier_org_id);
CREATE INDEX idx_sn_status ON ship_notices(status);

CREATE TABLE sn_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ship_notice_id  UUID NOT NULL REFERENCES ship_notices(id),
    po_line_id      UUID NOT NULL REFERENCES po_lines(id),
    shipped_quantity NUMERIC(12,4) NOT NULL,
    lot_number      VARCHAR(50),
    serial_numbers  VARCHAR(255)[],
    package_number  INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sn_lines_sn ON sn_lines(ship_notice_id);

-- GS1 EPCIS-aligned shipment events
CREATE TABLE shipment_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ship_notice_id  UUID NOT NULL REFERENCES ship_notices(id),
    event_type      VARCHAR(30) NOT NULL,  -- ObjectEvent, AggregationEvent, TransactionEvent
    event_time      TIMESTAMPTZ NOT NULL,
    event_timezone  VARCHAR(10) NOT NULL,
    biz_step        VARCHAR(100),          -- urn:epcglobal:cbv:bizstep:shipping, :receiving, etc.
    disposition     VARCHAR(100),          -- urn:epcglobal:cbv:disp:in_transit, etc.
    read_point      VARCHAR(255),          -- location GLN or URI
    biz_location    VARCHAR(255),          -- location GLN or URI
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shipment_events_sn ON shipment_events(ship_notice_id);
CREATE INDEX idx_shipment_events_time ON shipment_events(event_time);
```

## Invoices (EDI 810 / cXML InvoiceDetailRequest)

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    po_id           UUID REFERENCES purchase_orders(id),  -- NULL for non-PO invoices
    invoice_number  VARCHAR(50) NOT NULL,
    invoice_date    DATE NOT NULL,
    due_date        DATE,
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(15,2) NOT NULL,
    tax_total       NUMERIC(15,2) NOT NULL DEFAULT 0,
    discount_total  NUMERIC(15,2) NOT NULL DEFAULT 0,
    grand_total     NUMERIC(15,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'submitted' CHECK (status IN ('draft', 'submitted', 'under_review', 'approved', 'disputed', 'paid', 'cancelled')),
    payment_reference VARCHAR(100),
    paid_date       DATE,
    paid_amount     NUMERIC(15,2),
    peppol_id       VARCHAR(100),          -- Peppol document identifier
    source_format   VARCHAR(10) CHECK (source_format IN ('portal', 'cxml', 'edi_x12', 'edi_edifact', 'pdf_extract', 'api')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_org_id, invoice_number)
);

CREATE INDEX idx_invoices_buyer ON invoices(buyer_org_id);
CREATE INDEX idx_invoices_supplier ON invoices(supplier_org_id);
CREATE INDEX idx_invoices_po ON invoices(po_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due_date ON invoices(due_date);

CREATE TABLE invoice_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    po_line_id      UUID REFERENCES po_lines(id),
    line_number     INTEGER NOT NULL,
    description     VARCHAR(500) NOT NULL,
    quantity        NUMERIC(12,4) NOT NULL,
    unit_price      NUMERIC(15,4) NOT NULL,
    line_total      NUMERIC(15,2) NOT NULL,
    tax_rate        NUMERIC(5,4),
    tax_amount      NUMERIC(15,2) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (invoice_id, line_number)
);

CREATE INDEX idx_invoice_lines_invoice ON invoice_lines(invoice_id);

CREATE TABLE invoice_match_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    match_type      VARCHAR(20) NOT NULL CHECK (match_type IN ('two_way', 'three_way')),
    po_match        BOOLEAN NOT NULL DEFAULT false,
    grn_match       BOOLEAN NOT NULL DEFAULT false,   -- goods receipt note
    price_variance  NUMERIC(15,2),
    quantity_variance NUMERIC(12,4),
    overall_result  VARCHAR(20) NOT NULL CHECK (overall_result IN ('matched', 'variance', 'mismatch')),
    matched_by      VARCHAR(20) CHECK (matched_by IN ('auto', 'manual', 'ai')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_match_results_invoice ON invoice_match_results(invoice_id);
```

## Supplier Performance & Scorecards

```sql
CREATE TABLE scorecard_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scorecard_criteria (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES scorecard_templates(id),
    name            VARCHAR(100) NOT NULL,  -- e.g., 'On-Time Delivery', 'Quality Reject Rate'
    category        VARCHAR(30) NOT NULL CHECK (category IN ('delivery', 'quality', 'cost', 'responsiveness', 'sustainability', 'compliance', 'innovation')),
    weight          NUMERIC(5,2) NOT NULL,  -- percentage weight, must sum to 100 per template
    target_value    NUMERIC(10,4),
    unit            VARCHAR(20),           -- '%', 'PPM', 'days', 'score'
    scoring_method  VARCHAR(20) NOT NULL CHECK (scoring_method IN ('higher_is_better', 'lower_is_better', 'target_match')),
    red_threshold   NUMERIC(10,4),
    yellow_threshold NUMERIC(10,4),
    green_threshold NUMERIC(10,4),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scorecard_criteria_template ON scorecard_criteria(template_id);

CREATE TABLE supplier_scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trading_relationship_id UUID NOT NULL REFERENCES trading_relationships(id),
    template_id     UUID NOT NULL REFERENCES scorecard_templates(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    overall_score   NUMERIC(5,2),
    overall_rating  VARCHAR(10) CHECK (overall_rating IN ('green', 'yellow', 'red')),
    generated_by    VARCHAR(20) CHECK (generated_by IN ('manual', 'auto', 'ai')),
    published       BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scorecards_rel ON supplier_scorecards(trading_relationship_id);
CREATE INDEX idx_scorecards_period ON supplier_scorecards(period_start, period_end);

CREATE TABLE scorecard_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scorecard_id    UUID NOT NULL REFERENCES supplier_scorecards(id),
    criteria_id     UUID NOT NULL REFERENCES scorecard_criteria(id),
    actual_value    NUMERIC(10,4) NOT NULL,
    weighted_score  NUMERIC(5,2) NOT NULL,
    rating          VARCHAR(10) CHECK (rating IN ('green', 'yellow', 'red')),
    data_source     VARCHAR(20),           -- 'erp', 'manual', 'calculated'
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scorecard_id, criteria_id)
);
```

## Risk Management (ISO 31000 aligned)

```sql
CREATE TABLE risk_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(50) NOT NULL UNIQUE,  -- 'financial', 'geopolitical', 'operational', 'esg', 'compliance', 'cyber'
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE risk_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    category_id     UUID NOT NULL REFERENCES risk_categories(id),
    risk_score      NUMERIC(5,2) NOT NULL,  -- 0-100
    risk_level      VARCHAR(10) NOT NULL CHECK (risk_level IN ('low', 'medium', 'high', 'critical')),
    likelihood      NUMERIC(5,2),
    impact          NUMERIC(5,2),
    source          VARCHAR(30) NOT NULL CHECK (source IN ('manual', 'ai_scan', 'third_party_feed', 'financial_data')),
    summary         TEXT,
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    next_review_at  DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_assessments_supplier ON risk_assessments(supplier_org_id);
CREATE INDEX idx_risk_assessments_level ON risk_assessments(risk_level);

CREATE TABLE risk_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    risk_assessment_id UUID NOT NULL REFERENCES risk_assessments(id),
    alert_type      VARCHAR(30) NOT NULL,
    severity        VARCHAR(10) NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    source_url      VARCHAR(500),
    acknowledged    BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_alerts_assessment ON risk_alerts(risk_assessment_id);
```

## Corrective Actions & CAPA

```sql
CREATE TABLE corrective_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trading_relationship_id UUID NOT NULL REFERENCES trading_relationships(id),
    po_id           UUID REFERENCES purchase_orders(id),
    invoice_id      UUID REFERENCES invoices(id),
    capa_number     VARCHAR(30) NOT NULL,
    issue_type      VARCHAR(30) NOT NULL CHECK (issue_type IN ('quality_defect', 'delivery_delay', 'quantity_discrepancy', 'documentation', 'compliance', 'other')),
    severity        VARCHAR(10) NOT NULL CHECK (severity IN ('minor', 'major', 'critical')),
    status          VARCHAR(20) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'investigating', 'action_planned', 'in_progress', 'verification', 'closed')),
    description     TEXT NOT NULL,
    root_cause      TEXT,
    corrective_action TEXT,
    preventive_action TEXT,
    due_date        DATE,
    closed_at       TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    assigned_to     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_capa_rel ON corrective_actions(trading_relationship_id);
CREATE INDEX idx_capa_status ON corrective_actions(status);
```

## Messaging & Notifications

```sql
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trading_relationship_id UUID NOT NULL REFERENCES trading_relationships(id),
    sender_user_id  UUID NOT NULL REFERENCES users(id),
    subject         VARCHAR(255),
    body            TEXT NOT NULL,
    related_entity_type VARCHAR(30),       -- 'purchase_order', 'invoice', 'ship_notice', 'corrective_action'
    related_entity_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_rel ON messages(trading_relationship_id);
CREATE INDEX idx_messages_related ON messages(related_entity_type, related_entity_id);

CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    type            VARCHAR(30) NOT NULL,  -- 'po_received', 'invoice_approved', 'asn_required', etc.
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    related_entity_type VARCHAR(30),
    related_entity_id UUID,
    read            BOOLEAN NOT NULL DEFAULT false,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user_unread ON notifications(user_id) WHERE NOT read;
```

## ESG & Sustainability (ISO 20400 aligned)

```sql
CREATE TABLE esg_questionnaires (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(100) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE esg_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    questionnaire_id UUID NOT NULL REFERENCES esg_questionnaires(id),
    category        VARCHAR(30) NOT NULL CHECK (category IN ('environmental', 'social', 'governance')),
    question_text   TEXT NOT NULL,
    response_type   VARCHAR(20) NOT NULL CHECK (response_type IN ('yes_no', 'numeric', 'text', 'file_upload', 'select')),
    weight          NUMERIC(5,2) DEFAULT 1,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE esg_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id     UUID NOT NULL REFERENCES esg_questions(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    response_text   TEXT,
    response_numeric NUMERIC(15,4),
    response_bool   BOOLEAN,
    file_url        VARCHAR(500),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (question_id, supplier_org_id)
);

CREATE TABLE esg_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    questionnaire_id UUID NOT NULL REFERENCES esg_questionnaires(id),
    environmental_score NUMERIC(5,2),
    social_score    NUMERIC(5,2),
    governance_score NUMERIC(5,2),
    overall_score   NUMERIC(5,2),
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_org_id, questionnaire_id)
);

CREATE INDEX idx_esg_scores_supplier ON esg_scores(supplier_org_id);
```

## Reference Data

```sql
CREATE TABLE currencies (
    code            CHAR(3) PRIMARY KEY,   -- ISO 4217
    name            VARCHAR(50) NOT NULL,
    symbol          VARCHAR(5)
);

CREATE TABLE countries (
    code            CHAR(2) PRIMARY KEY,   -- ISO 3166-1 alpha-2
    name            VARCHAR(100) NOT NULL,
    alpha3          CHAR(3),               -- ISO 3166-1 alpha-3
    numeric_code    CHAR(3)                -- ISO 3166-1 numeric
);

CREATE TABLE commodity_codes (
    code            VARCHAR(20) PRIMARY KEY, -- UNSPSC code
    description     VARCHAR(255) NOT NULL,
    segment         VARCHAR(100),
    family          VARCHAR(100),
    class           VARCHAR(100)
);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    org_id          UUID REFERENCES organisations(id),
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL CHECK (action IN ('create', 'update', 'delete', 'status_change', 'login', 'export')),
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_time ON audit_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organisations, org_addresses, users, org_memberships |
| Trading Relationships | 1 | buyer-supplier links |
| Supplier Profile | 2 | profiles, documents |
| Purchase Orders | 2 | headers + lines |
| PO Acknowledgements | 2 | headers + lines |
| Ship Notices & Events | 3 | headers, lines, EPCIS events |
| Invoices & Matching | 3 | headers, lines, match results |
| Scorecards | 4 | templates, criteria, scorecards, scores |
| Risk Management | 3 | categories, assessments, alerts |
| Corrective Actions | 1 | CAPA workflow |
| Messaging & Notifications | 2 | messages, notifications |
| ESG & Sustainability | 4 | questionnaires, questions, responses, scores |
| Reference Data | 3 | currencies, countries, commodity codes |
| Audit | 1 | append-only audit log |
| **Total** | **35** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe for multi-region deployments, and eliminates sequential ID enumeration attacks on supplier-facing APIs.

2. **Separate header + lines tables for all transaction documents** — mirrors the EDI 850/855/856/810 and cXML document structures exactly, making import/export mapping straightforward.

3. **Row-level tenancy via `buyer_org_id` / `supplier_org_id`** — chose shared-table multi-tenancy over schema-per-tenant for simplicity at the MVP stage; can add PostgreSQL Row-Level Security (RLS) policies later.

4. **Explicit `trading_relationships` junction table** — models the buyer-supplier relationship as a first-class entity carrying its own attributes (payment terms, status, onboarding date), rather than inferring it from transactional data.

5. **Scorecard template/instance separation** — buyers define templates with weighted criteria; individual scorecards are instances scored against a template for a specific period and supplier. Supports different scorecard templates for different supplier categories.

6. **EPCIS-aligned shipment events** — rather than inventing a custom event schema, the `shipment_events` table uses GS1 EPCIS field names (biz_step, disposition, read_point, biz_location) so events can be serialized to EPCIS 2.0 JSON-LD without transformation.

7. **Invoice match results as a separate table** — decouples the matching engine (which may be rules-based, ML-based, or manual) from the invoice itself, allowing multiple match attempts and preserving match history.

8. **JSONB used only in audit log** — the `old_values` / `new_values` columns use JSONB because audit payloads vary by entity type, but all operational data is fully relational.

9. **ISO-aligned reference data** — currencies (ISO 4217), countries (ISO 3166), and commodity codes (UNSPSC) as dedicated reference tables rather than free-text fields, enabling consistent reporting and standards-compliant document generation.

10. **No soft deletes on transactional data** — status fields (`cancelled`, `closed`) replace deletion for business objects; the audit log captures all state transitions for compliance.
