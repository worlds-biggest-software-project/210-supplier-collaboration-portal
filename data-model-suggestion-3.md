# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Supplier Collaboration Portal · Created: 2026-05-20

## Philosophy

This model uses strongly typed relational columns for the stable, universally applicable fields (PO number, dates, amounts, statuses) and PostgreSQL JSONB columns for variable, jurisdiction-specific, category-specific, or integration-specific data that differs between buyers, supplier types, or regions.

This pattern is widely used in modern SaaS platforms that serve diverse customer segments. Shopify's order model, Stripe's metadata fields, and Salesforce's custom fields all follow this principle: a fixed schema for the core business logic, plus a flexible extension mechanism for everything else. In a supplier collaboration portal, this is particularly relevant because different buyers have different document requirements, different jurisdictions have different tax and compliance fields, and different supplier categories (manufacturing vs. services vs. raw materials) need different profile attributes.

The JSONB columns are not unstructured blobs — they follow documented JSON Schema contracts enforced at the application layer. PostgreSQL GIN indexes on JSONB columns enable efficient querying. This approach dramatically reduces schema migration frequency during rapid development while preserving relational integrity for the fields that matter most for queries, reporting, and data integrity.

**Best for:** Rapid MVP development with multi-region/multi-buyer flexibility, teams that need to iterate on features without frequent schema migrations, platforms serving diverse buyer configurations.

**Trade-offs:**
- Pro: Far fewer tables than fully normalized (approximately 20 vs. 35+)
- Pro: New buyer-specific or region-specific fields added without DDL migrations
- Pro: Fast iteration — new features can start in JSONB and graduate to columns when patterns stabilize
- Pro: GIN-indexed JSONB queries are fast for containment and existence checks
- Pro: PostgreSQL JSONB is battle-tested at scale
- Con: JSONB fields are not enforced by the database — validation is application-level
- Con: Complex queries on deeply nested JSONB are slower than column queries
- Con: Reporting tools may struggle with JSONB — may need to materialize views for BI
- Con: Risk of "schema drift" if JSONB structures are not disciplined
- Con: Foreign key constraints cannot reference inside JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ANSI X12 850/855/856/810 | Core header/line fields are relational columns; EDI-specific segments and qualifiers stored in `edi_data JSONB` |
| cXML 1.2 | cXML-specific fields (Extrinsics, custom elements) stored in `cxml_data JSONB`; core fields mapped to relational columns |
| ISO 3166-1/2 | Country codes as `CHAR(2)` columns; jurisdiction-specific compliance rules in `compliance_config JSONB` |
| ISO 4217 | Currency codes as typed columns on all monetary tables |
| ISO 20400 / ESG | ESG questionnaire responses stored in flexible JSONB — different buyers ask different ESG questions |
| ISO 31000 | Risk scores as relational columns; risk signal details in `risk_details JSONB` |
| Peppol / EN 16931 | Peppol-specific invoice fields in `peppol_data JSONB` on the invoices table |
| GS1 EPCIS 2.0 | EPCIS event payloads stored as JSONB — naturally fits the variable event structure |

---

## Core Identity & Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    org_type        VARCHAR(20) NOT NULL CHECK (org_type IN ('buyer', 'supplier', 'both')),
    tax_id          VARCHAR(50),
    lei             VARCHAR(20),           -- ISO 17442
    duns_number     VARCHAR(9),
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Flexible organisation-level configuration
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings JSONB:
    -- {
    --   "branding": {"logo_url": "...", "primary_color": "#003366"},
    --   "procurement": {"default_payment_terms": "NET30", "auto_acknowledge": false},
    --   "compliance": {"require_w9": true, "require_insurance": true, "min_insurance_amount": 1000000},
    --   "notifications": {"email_on_po": true, "email_on_invoice_status": true}
    -- }
    addresses       JSONB NOT NULL DEFAULT '[]',
    -- Example addresses JSONB:
    -- [
    --   {"type": "billing", "primary": true, "line_1": "123 Main St", "city": "Chicago", "state": "IL", "postal": "60601", "country": "US"},
    --   {"type": "shipping", "primary": false, "line_1": "456 Warehouse Dr", "city": "Gary", "state": "IN", "postal": "46401", "country": "US"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_settings ON organisations USING GIN (settings);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    auth_provider   VARCHAR(20) NOT NULL DEFAULT 'local',
    auth_subject    VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example preferences:
    -- {"timezone": "America/Chicago", "locale": "en-US", "dashboard_layout": "compact"}
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    role            VARCHAR(30) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions:
    -- ["po:read", "po:acknowledge", "invoice:create", "invoice:submit", "asn:create"]
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, org_id, role)
);

CREATE INDEX idx_memberships_user ON org_memberships(user_id);
CREATE INDEX idx_memberships_org ON org_memberships(org_id);
```

## Trading Relationships & Supplier Profiles

```sql
CREATE TABLE trading_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_terms   VARCHAR(20),
    default_currency CHAR(3),
    onboarded_at    TIMESTAMPTZ,
    -- Buyer-specific supplier configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "category": "direct_materials",
    --   "commodity_codes": ["31162800", "31162900"],
    --   "scorecard_template": "manufacturing",
    --   "required_documents": ["iso_cert", "insurance", "w9"],
    --   "auto_match_invoices": true,
    --   "payment_method": "ach"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (buyer_org_id, supplier_org_id)
);

CREATE INDEX idx_tr_buyer ON trading_relationships(buyer_org_id);
CREATE INDEX idx_tr_supplier ON trading_relationships(supplier_org_id);
CREATE INDEX idx_tr_config ON trading_relationships USING GIN (config);

CREATE TABLE supplier_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) UNIQUE,
    business_type   VARCHAR(50),
    year_established INTEGER,
    -- Flexible profile attributes that vary by supplier type and region
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example attributes for a US manufacturer:
    -- {
    --   "employee_count": 250,
    --   "annual_revenue_usd": 45000000,
    --   "certifications": ["ISO 9001:2015", "AS9100D", "ITAR"],
    --   "capabilities": ["CNC machining", "sheet metal", "surface treatment"],
    --   "diversity": {"minority_owned": false, "women_owned": true, "veteran_owned": false, "small_business": true},
    --   "naics_codes": ["332710", "332999"],
    --   "insurance": {"general_liability": 2000000, "workers_comp": true, "expiry": "2027-03-15"},
    --   "banking": {"bank_name": "Chase", "routing": "021000021", "account_last4": "4567", "verified": true}
    -- }
    -- Example attributes for an EU services supplier:
    -- {
    --   "employee_count": 50,
    --   "vat_number": "DE123456789",
    --   "esg_rating": "B+",
    --   "csrd_reporting": true,
    --   "data_processing_agreement": true,
    --   "services": ["IT consulting", "software development"],
    --   "rates": {"junior_hourly_eur": 85, "senior_hourly_eur": 145}
    -- }
    documents       JSONB NOT NULL DEFAULT '[]',
    -- Example documents:
    -- [
    --   {"type": "iso_cert", "title": "ISO 9001:2015", "url": "s3://...", "expires": "2027-06-30", "verified": true, "verified_at": "2026-01-15"},
    --   {"type": "insurance", "title": "General Liability", "url": "s3://...", "expires": "2027-03-15", "verified": true},
    --   {"type": "w9", "title": "W-9 2026", "url": "s3://...", "verified": true}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sp_attributes ON supplier_profiles USING GIN (attributes);
CREATE INDEX idx_sp_documents ON supplier_profiles USING GIN (documents);
```

## Purchase Orders

```sql
CREATE TABLE purchase_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    po_number       VARCHAR(50) NOT NULL,
    order_date      DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'issued',
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(15,2) NOT NULL DEFAULT 0,
    tax_total       NUMERIC(15,2) NOT NULL DEFAULT 0,
    grand_total     NUMERIC(15,2) NOT NULL DEFAULT 0,
    payment_terms   VARCHAR(20),
    requested_delivery_date DATE,
    source_format   VARCHAR(10),
    -- Flexible fields for buyer-specific PO attributes, cXML/EDI metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "ship_to": {"line_1": "123 Factory Rd", "city": "Detroit", "state": "MI", "country": "US"},
    --   "bill_to": {"line_1": "456 HQ Blvd", "city": "Chicago", "state": "IL", "country": "US"},
    --   "buyer_contact": {"name": "Jane Smith", "email": "jane@buyer.com", "phone": "555-0100"},
    --   "cxml": {"payload_id": "2026.05.20.123456@buyer.com", "deployment_mode": "production"},
    --   "edi": {"isa_control": "000000042", "gs_control": "42"},
    --   "custom_fields": {"project_code": "PROJ-2026-A", "cost_center": "CC-4500", "gl_account": "50100"}
    -- }
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (buyer_org_id, po_number)
);

CREATE INDEX idx_po_buyer ON purchase_orders(buyer_org_id);
CREATE INDEX idx_po_supplier ON purchase_orders(supplier_org_id);
CREATE INDEX idx_po_status ON purchase_orders(status);
CREATE INDEX idx_po_date ON purchase_orders(order_date);
CREATE INDEX idx_po_metadata ON purchase_orders USING GIN (metadata);

CREATE TABLE po_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_id           UUID NOT NULL REFERENCES purchase_orders(id) ON DELETE CASCADE,
    line_number     INTEGER NOT NULL,
    item_number     VARCHAR(50),
    supplier_part_number VARCHAR(50),
    description     VARCHAR(500) NOT NULL,
    uom             VARCHAR(10) NOT NULL,
    quantity        NUMERIC(12,4) NOT NULL,
    unit_price      NUMERIC(15,4) NOT NULL,
    line_total      NUMERIC(15,2) NOT NULL,
    requested_delivery_date DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    -- Line-level flexible data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "commodity_code": "31162800",
    --   "specifications": {"material": "6061-T6 Aluminum", "tolerance": "+/- 0.005in"},
    --   "packaging": {"type": "pallet", "quantity_per_package": 100}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (po_id, line_number)
);

CREATE INDEX idx_po_lines_po ON po_lines(po_id);
```

## PO Acknowledgements, Ship Notices, Invoices (Compact Design)

```sql
-- Acknowledgement stored as a JSONB document on the PO,
-- plus a dedicated table for line-level detail
CREATE TABLE po_acknowledgements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_id           UUID NOT NULL REFERENCES purchase_orders(id),
    overall_status  VARCHAR(20) NOT NULL,
    supplier_reference VARCHAR(50),
    ack_date        TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Full line-level acknowledgement data
    line_responses  JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"po_line_id": "...", "line_number": 1, "status": "accepted", "confirmed_qty": 1000, "promised_date": "2026-06-15"},
    --   {"po_line_id": "...", "line_number": 2, "status": "backordered", "confirmed_qty": 300, "promised_date": "2026-07-01", "notes": "Partial stock available"}
    -- ]
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ack_po ON po_acknowledgements(po_id);

CREATE TABLE ship_notices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_id           UUID NOT NULL REFERENCES purchase_orders(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    asn_number      VARCHAR(50) NOT NULL,
    ship_date       DATE NOT NULL,
    estimated_delivery DATE,
    actual_delivery DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'shipped',
    carrier_name    VARCHAR(100),
    tracking_number VARCHAR(100),
    -- Shipment detail including line items and EPCIS events
    shipment_data   JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "lines": [
    --     {"po_line_id": "...", "shipped_qty": 1000, "lot": "LOT-2026-042", "serials": []},
    --     {"po_line_id": "...", "shipped_qty": 300, "lot": "LOT-2026-043"}
    --   ],
    --   "packages": [{"number": 1, "weight_kg": 45.2, "dimensions": "60x40x30cm"}],
    --   "epcis_events": [
    --     {"type": "ObjectEvent", "time": "2026-05-20T08:30:00Z", "biz_step": "shipping", "read_point": "urn:epc:id:sgln:0614141.12345.0"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sn_po ON ship_notices(po_id);
CREATE INDEX idx_sn_supplier ON ship_notices(supplier_org_id);
CREATE INDEX idx_sn_status ON ship_notices(status);
CREATE INDEX idx_sn_data ON ship_notices USING GIN (shipment_data);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    po_id           UUID REFERENCES purchase_orders(id),
    invoice_number  VARCHAR(50) NOT NULL,
    invoice_date    DATE NOT NULL,
    due_date        DATE,
    currency_code   CHAR(3) NOT NULL,
    subtotal        NUMERIC(15,2) NOT NULL,
    tax_total       NUMERIC(15,2) NOT NULL DEFAULT 0,
    grand_total     NUMERIC(15,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'submitted',
    source_format   VARCHAR(10),
    -- Line items stored in JSONB for flexibility
    lines           JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"line": 1, "po_line_id": "...", "description": "Widget A", "qty": 1000, "unit_price": 12.50, "total": 12500.00, "tax_rate": 0.08, "tax": 1000.00},
    --   {"line": 2, "description": "Shipping", "qty": 1, "unit_price": 250.00, "total": 250.00, "tax_rate": 0, "tax": 0}
    -- ]
    -- Invoice matching and payment metadata
    match_data      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "match_type": "three_way",
    --   "po_matched": true,
    --   "grn_matched": true,
    --   "price_variance": 0.00,
    --   "matched_by": "ai",
    --   "matched_at": "2026-05-21T10:30:00Z"
    -- }
    payment_data    JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {"paid_date": "2026-06-20", "paid_amount": 13750.00, "payment_ref": "ACH-20260620-0042"}
    -- Peppol/regulatory metadata
    regulatory_data JSONB NOT NULL DEFAULT '{}',
    -- Example for EU invoice:
    -- {"peppol_id": "...", "scheme_id": "0088", "tax_scheme": "VAT", "reverse_charge": false}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_org_id, invoice_number)
);

CREATE INDEX idx_inv_buyer ON invoices(buyer_org_id);
CREATE INDEX idx_inv_supplier ON invoices(supplier_org_id);
CREATE INDEX idx_inv_po ON invoices(po_id);
CREATE INDEX idx_inv_status ON invoices(status);
CREATE INDEX idx_inv_due ON invoices(due_date);
CREATE INDEX idx_inv_lines ON invoices USING GIN (lines);
```

## Scorecards & Risk (Flexible Criteria via JSONB)

```sql
CREATE TABLE supplier_scorecards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    trading_rel_id  UUID NOT NULL REFERENCES trading_relationships(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    overall_score   NUMERIC(5,2),
    overall_rating  VARCHAR(10),
    published       BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    -- All criteria, weights, and scores in a single JSONB structure
    scores          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "template": "manufacturing_v2",
    --   "criteria": [
    --     {"name": "On-Time Delivery", "category": "delivery", "weight": 30, "target": 95, "actual": 92.3, "score": 77, "rating": "yellow"},
    --     {"name": "Quality Reject Rate", "category": "quality", "weight": 25, "target": 500, "actual": 320, "unit": "PPM", "score": 88, "rating": "green"},
    --     {"name": "Invoice Accuracy", "category": "cost", "weight": 15, "target": 98, "actual": 99.1, "score": 95, "rating": "green"},
    --     {"name": "Responsiveness", "category": "responsiveness", "weight": 15, "target": 24, "actual": 8.5, "unit": "hours", "score": 95, "rating": "green"},
    --     {"name": "ESG Score", "category": "sustainability", "weight": 15, "target": 70, "actual": 65, "score": 65, "rating": "yellow"}
    --   ]
    -- }
    generated_by    VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sc_buyer ON supplier_scorecards(buyer_org_id);
CREATE INDEX idx_sc_supplier ON supplier_scorecards(supplier_org_id);
CREATE INDEX idx_sc_period ON supplier_scorecards(period_start);
CREATE INDEX idx_sc_scores ON supplier_scorecards USING GIN (scores);

CREATE TABLE risk_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    overall_score   NUMERIC(5,2) NOT NULL,
    overall_level   VARCHAR(10) NOT NULL,
    source          VARCHAR(30) NOT NULL,
    -- Detailed risk breakdown by category
    risk_details    JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "financial": {"score": 72, "level": "medium", "signals": ["revenue_decline_10pct", "credit_rating_downgrade"]},
    --   "geopolitical": {"score": 85, "level": "low", "country": "DE", "signals": []},
    --   "operational": {"score": 60, "level": "medium", "signals": ["single_site_manufacturing"]},
    --   "esg": {"score": 78, "level": "low", "signals": ["carbon_reduction_target_set"]},
    --   "compliance": {"score": 90, "level": "low", "signals": []},
    --   "cyber": {"score": 65, "level": "medium", "signals": ["no_soc2_cert"]}
    -- }
    alerts          JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"severity": "warning", "title": "Credit rating downgraded", "description": "...", "source_url": "...", "created_at": "2026-05-19", "acknowledged": false}
    -- ]
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    next_review     DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_supplier ON risk_assessments(supplier_org_id);
CREATE INDEX idx_risk_level ON risk_assessments(overall_level);
CREATE INDEX idx_risk_details ON risk_assessments USING GIN (risk_details);
```

## Corrective Actions, Messaging, Notifications

```sql
CREATE TABLE corrective_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trading_rel_id  UUID NOT NULL REFERENCES trading_relationships(id),
    po_id           UUID REFERENCES purchase_orders(id),
    capa_number     VARCHAR(30) NOT NULL,
    issue_type      VARCHAR(30) NOT NULL,
    severity        VARCHAR(10) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    description     TEXT NOT NULL,
    -- Structured CAPA workflow data
    workflow_data   JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "root_cause": "Incoming raw material out of spec from sub-tier supplier",
    --   "corrective_action": "Implemented incoming inspection for critical dimension",
    --   "preventive_action": "Added sub-tier supplier audit requirement",
    --   "timeline": [
    --     {"status": "open", "at": "2026-05-01", "by": "user-uuid-1"},
    --     {"status": "investigating", "at": "2026-05-03", "by": "user-uuid-2"},
    --     {"status": "action_planned", "at": "2026-05-10", "by": "user-uuid-2"}
    --   ],
    --   "attachments": [{"title": "8D Report", "url": "s3://...", "uploaded_at": "2026-05-10"}]
    -- }
    due_date        DATE,
    assigned_to     UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_capa_rel ON corrective_actions(trading_rel_id);
CREATE INDEX idx_capa_status ON corrective_actions(status);

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trading_rel_id  UUID NOT NULL REFERENCES trading_relationships(id),
    sender_user_id  UUID NOT NULL REFERENCES users(id),
    subject         VARCHAR(255),
    body            TEXT NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example context:
    -- {"entity_type": "purchase_order", "entity_id": "...", "po_number": "PO-2026-00142"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_msg_rel ON messages(trading_rel_id);

CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    type            VARCHAR(30) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    context         JSONB NOT NULL DEFAULT '{}',
    read            BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notif_user_unread ON notifications(user_id) WHERE NOT read;
```

## ESG (Fully JSONB-Driven Questionnaires)

```sql
CREATE TABLE esg_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organisations(id),
    supplier_org_id UUID NOT NULL REFERENCES organisations(id),
    -- Questionnaire definition and responses in a single flexible structure
    questionnaire   JSONB NOT NULL,
    -- Example:
    -- {
    --   "name": "ESG Assessment 2026",
    --   "version": 2,
    --   "sections": [
    --     {
    --       "category": "environmental",
    --       "questions": [
    --         {"id": "E1", "text": "What are your Scope 1+2 emissions (tCO2e)?", "type": "numeric", "response": 12500, "weight": 3},
    --         {"id": "E2", "text": "Do you have a science-based emissions target?", "type": "yes_no", "response": true, "weight": 2},
    --         {"id": "E3", "text": "Upload your CDP disclosure", "type": "file", "response": "s3://...", "weight": 1}
    --       ]
    --     },
    --     {
    --       "category": "social",
    --       "questions": [
    --         {"id": "S1", "text": "What is your Lost Time Injury Rate?", "type": "numeric", "response": 0.8, "weight": 2}
    --       ]
    --     }
    --   ]
    -- }
    environmental_score NUMERIC(5,2),
    social_score    NUMERIC(5,2),
    governance_score NUMERIC(5,2),
    overall_score   NUMERIC(5,2),
    submitted_at    TIMESTAMPTZ,
    scored_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_esg_buyer ON esg_assessments(buyer_org_id);
CREATE INDEX idx_esg_supplier ON esg_assessments(supplier_org_id);
CREATE INDEX idx_esg_questionnaire ON esg_assessments USING GIN (questionnaire);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    org_id          UUID,
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {"field": "status", "old": "submitted", "new": "approved"}
    -- or for JSONB field changes:
    -- {"path": "metadata.custom_fields.project_code", "old": "PROJ-A", "new": "PROJ-B"}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);
```

## Example Queries

### Find all suppliers with a specific certification

```sql
-- Find suppliers with ISO 9001 certification using JSONB containment
SELECT o.name, sp.attributes
FROM supplier_profiles sp
JOIN organisations o ON o.id = sp.org_id
WHERE sp.attributes @> '{"certifications": ["ISO 9001:2015"]}';
```

### Query invoice line items across JSONB

```sql
-- Find all invoice lines for a specific PO line, even though lines are in JSONB
SELECT i.invoice_number, i.invoice_date, line_item.*
FROM invoices i,
     jsonb_to_recordset(i.lines) AS line_item(
         line int, po_line_id text, description text,
         qty numeric, unit_price numeric, total numeric
     )
WHERE i.buyer_org_id = :buyer_id
  AND line_item.po_line_id = :po_line_id;
```

### Scorecard trend analysis across JSONB criteria

```sql
-- On-time delivery trend for a supplier over time
SELECT sc.period_start,
       sc.overall_score,
       criterion->>'actual' AS otd_actual,
       criterion->>'rating' AS otd_rating
FROM supplier_scorecards sc,
     jsonb_array_elements(sc.scores->'criteria') AS criterion
WHERE sc.supplier_org_id = :supplier_id
  AND criterion->>'name' = 'On-Time Delivery'
ORDER BY sc.period_start;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Tenancy | 3 | organisations (with addresses in JSONB), users, org_memberships |
| Trading Relationships | 1 | with config in JSONB |
| Supplier Profiles | 1 | attributes, documents, capabilities all in JSONB |
| Purchase Orders | 2 | headers + lines (metadata in JSONB) |
| PO Acknowledgements | 1 | line responses in JSONB |
| Ship Notices | 1 | shipment detail + EPCIS events in JSONB |
| Invoices | 1 | lines, match data, payment data, regulatory data all in JSONB |
| Scorecards | 1 | criteria and scores in JSONB |
| Risk | 1 | risk breakdown and alerts in JSONB |
| Corrective Actions | 1 | workflow timeline in JSONB |
| Messaging | 2 | messages, notifications |
| ESG | 1 | questionnaire + responses in single JSONB |
| Audit | 1 | changes in JSONB |
| **Total** | **17** | Approximately half the table count of Model 1 |

---

## Key Design Decisions

1. **JSONB for variable data, columns for queryable/filterable data** — the rule is: if you WHERE-clause on it, GROUP BY it, or JOIN on it, it is a column. If it varies by buyer, region, or category, it goes in JSONB. This keeps the core schema stable while allowing infinite extensibility.

2. **Addresses embedded in organisations JSONB** — avoids a separate table for a simple nested list. Since addresses are always fetched with the org and rarely queried independently, JSONB is a better fit than a junction table.

3. **Invoice lines in JSONB rather than a separate table** — a deliberate trade-off: reduces table count, simplifies the API (the invoice is a single document), and matches how invoices are processed (as complete documents). The trade-off is that cross-invoice line queries require `jsonb_to_recordset` expansion, which is slower than a relational join.

4. **Scorecard criteria defined inline rather than via templates** — in Model 1, scorecards reference template/criteria tables. Here, the criteria definition is embedded in the scorecard JSONB. This means different periods can have different criteria without schema changes, but it also means updating a template does not retroactively update old scorecards (which is actually the correct behavior — historical scorecards should reflect the criteria used at the time).

5. **GIN indexes on all JSONB columns** — PostgreSQL GIN indexes support `@>` (containment), `?` (key existence), and `?|` / `?&` (key array operations). These cover 90% of JSONB query patterns efficiently. For deeply nested path queries, consider expression indexes (e.g., `CREATE INDEX ON invoices ((regulatory_data->>'peppol_id'))`).

6. **Permission arrays in org_memberships** — instead of a separate permissions table or bit flags, permissions are stored as a JSONB array of permission strings. This makes permission checks a simple `permissions @> '["po:acknowledge"]'` containment query and allows buyer-specific permission vocabularies without schema changes.

7. **CAPA workflow timeline in JSONB** — the corrective action status history is stored as an array of timeline entries in JSONB. This provides a lightweight audit trail for the CAPA lifecycle without requiring a separate status history table.

8. **ESG questionnaires as self-contained JSONB documents** — different buyers ask different ESG questions, and questions evolve over time. Storing the complete questionnaire (questions + responses) in JSONB means each assessment is a self-contained snapshot, and adding new ESG questions requires zero schema changes.

9. **Graduated column promotion** — the design philosophy is "start in JSONB, promote to column when patterns stabilize." For example, if every buyer starts using a `project_code` custom field on POs, it can be promoted to a real column in a later migration while keeping the JSONB field for backward compatibility.

10. **17 tables vs. 35 in Model 1** — the hybrid approach roughly halves the table count, which reduces migration complexity, simplifies the ORM layer, and makes the API surface smaller. The cost is paid in application-level validation and more complex JSONB queries for cross-entity reporting.
