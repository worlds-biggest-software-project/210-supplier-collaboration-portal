# Supplier Collaboration Portal — Phased Development Plan

> Project: 210-supplier-collaboration-portal · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (full-stack) | Dashboard-heavy portal with dual user bases (buyer + supplier); shared types for PO/invoice/ASN schemas between API and frontend; EDI parsing libraries available in JS |
| Framework | Next.js 15 (App Router) for frontend; Fastify for API | Fastify for high-throughput EDI/cXML document processing; Next.js for buyer and supplier portal UIs |
| Database | PostgreSQL 16 | Normalised schema from data-model-suggestion-1 (~50 tables); JSONB for EDI extension fields and supplier profile custom attributes |
| ORM | Drizzle ORM | Type-safe; handles complex procurement entity relationships (PO→lines→acknowledgement→ASN→invoice); generates migrations |
| Task queue | BullMQ + Redis | Async EDI/cXML processing, invoice matching, risk monitoring, scorecard computation, AI document extraction |
| Document storage | S3-compatible (MinIO) | Supplier certificates, invoices, ASN documents, quality reports |
| LLM integration | Anthropic TypeScript SDK (Claude) | Invoice PDF extraction, risk summary generation, natural-language PO/invoice queries |
| OCR / Document AI | OpenAI Vision API (GPT-4o) | Non-PO invoice extraction from PDF; supplier certificate validation |
| EDI processing | Custom X12 parser (TypeScript) | Parse/generate ANSI X12 850/855/856/810; no mature TS library — build thin parser following segment specs |
| cXML processing | xml2js + custom mapper | Parse/generate cXML OrderRequest, ShipNoticeRequest, InvoiceDetailRequest |
| Auth | Better Auth | Multi-org auth; supplier SSO via SAML 2.0; buyer-side OAuth 2.0/OIDC; role-based access per org |
| Email | React Email + Resend | PO notifications, invoice reminders, risk alerts, payment remittance |
| Containerisation | Docker + docker-compose | PostgreSQL, Redis, MinIO, Fastify API, BullMQ workers, Next.js |
| Testing | Vitest + Playwright | Unit/integration (Vitest); E2E (Playwright) |
| Linting | Biome | Fast linter + formatter |
| Type checking | TypeScript strict mode | Enforced in CI |
| Package manager | pnpm | Workspace support |

### Project Structure

```
supplier-portal/
├── package.json
├── pnpm-workspace.yaml
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.frontend
├── packages/
│   ├── shared/                      # Shared types, EDI constants
│   │   └── src/types/
│   │       ├── po.ts
│   │       ├── invoice.ts
│   │       ├── asn.ts
│   │       ├── scorecard.ts
│   │       ├── risk.ts
│   │       └── edi.ts
│   ├── api/                         # Fastify API
│   │   ├── drizzle.config.ts
│   │   ├── drizzle/migrations/
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── config.ts
│   │   │   ├── db/schema.ts
│   │   │   ├── routes/
│   │   │   │   ├── purchase-orders.ts
│   │   │   │   ├── invoices.ts
│   │   │   │   ├── ship-notices.ts
│   │   │   │   ├── suppliers.ts
│   │   │   │   ├── scorecards.ts
│   │   │   │   ├── quality.ts
│   │   │   │   ├── payments.ts
│   │   │   │   ├── edi.ts
│   │   │   │   ├── risk.ts
│   │   │   │   └── ai.ts
│   │   │   ├── services/
│   │   │   │   ├── po-manager.ts
│   │   │   │   ├── invoice-matcher.ts
│   │   │   │   ├── asn-manager.ts
│   │   │   │   ├── scorecard-engine.ts
│   │   │   │   ├── risk-monitor.ts
│   │   │   │   ├── capa-manager.ts
│   │   │   │   ├── invoice-extractor.ts    # AI PDF extraction
│   │   │   │   ├── nl-assistant.ts         # NL query interface
│   │   │   │   └── esg-scorer.ts
│   │   │   ├── integrations/
│   │   │   │   ├── edi-parser.ts           # X12 850/855/856/810
│   │   │   │   ├── edi-generator.ts
│   │   │   │   └── cxml-mapper.ts
│   │   │   ├── workers/
│   │   │   │   ├── invoice-match.worker.ts
│   │   │   │   ├── risk.worker.ts
│   │   │   │   ├── scorecard.worker.ts
│   │   │   │   ├── edi.worker.ts
│   │   │   │   └── extraction.worker.ts
│   │   │   └── lib/
│   │   │       ├── auth.ts
│   │   │       ├── s3.ts
│   │   │       └── pdf.ts
│   │   └── tests/
│   └── frontend/                    # Next.js (buyer + supplier portals)
│       ├── package.json
│       └── src/app/
│           ├── buyer/               # Buyer-facing pages
│           │   ├── purchase-orders/
│           │   ├── invoices/
│           │   ├── suppliers/
│           │   ├── scorecards/
│           │   ├── quality/
│           │   ├── risk/
│           │   └── settings/
│           ├── supplier/            # Supplier-facing pages
│           │   ├── orders/
│           │   ├── invoices/
│           │   ├── shipments/
│           │   ├── payments/
│           │   ├── profile/
│           │   ├── performance/
│           │   └── quality/
│           └── api/                 # Next.js API routes (if needed)
├── tests/e2e/
└── docs/
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database schema from data-model-suggestion-1 (organisations, users, supplier profiles, buyer-supplier relationships), dual-portal authentication, and Docker environment.

### Tasks

#### 1.1 — Project Scaffold and Configuration

**What**: Create pnpm workspace, Fastify API, Next.js frontend with buyer/supplier portal layouts, Docker.

**Design**:

```typescript
const envSchema = z.object({
  DATABASE_URL: z.string().default("postgresql://portal:portal@localhost:5432/portal"),
  REDIS_URL: z.string().default("redis://localhost:6379"),
  S3_ENDPOINT: z.string().default("http://localhost:9000"),
  S3_BUCKET: z.string().default("documents"),
  S3_ACCESS_KEY: z.string().default("minioadmin"),
  S3_SECRET_KEY: z.string().default("minioadmin"),
  ANTHROPIC_API_KEY: z.string().optional(),
  OPENAI_API_KEY: z.string().optional(),
  RESEND_API_KEY: z.string().optional(),
  SITE_URL: z.string().default("http://localhost:3000"),
  API_PORT: z.coerce.number().default(3001),
});
```

**Testing**:
- Unit: config parses env
- Integration: `docker-compose up -d` → all services healthy

#### 1.2 — Database Schema — Core Tables

**What**: Implement organisations (buyer/supplier), users, roles, buyer_supplier_relationships, supplier_profiles from data-model-suggestion-1.

**Design**:

Key entities from data-model-suggestion-1: `organisations` (with org_type: buyer/supplier/both), `users` (with role per organisation), `buyer_supplier_relationships` (linking buyer org to supplier org with status: pending/active/suspended/terminated), `supplier_profiles` (certifications, banking, tax docs, ESG fields).

Multi-buyer support: one supplier organisation can have relationships with many buyer organisations. Supplier profile fields are per-relationship (different payment terms per buyer) plus global fields (legal name, tax ID, certifications).

```typescript
export const buyerSupplierRelationships = pgTable("buyer_supplier_relationships", {
  id: uuid("id").defaultRandom().primaryKey(),
  buyerOrgId: uuid("buyer_org_id").notNull().references(() => organisations.id),
  supplierOrgId: uuid("supplier_org_id").notNull().references(() => organisations.id),
  status: varchar("status", { length: 30, enum: ["pending", "active", "suspended", "terminated"] }).notNull().default("pending"),
  paymentTermsDays: integer("payment_terms_days").default(30),
  currencyCode: char("currency_code", { length: 3 }).default("USD"),
  supplierCode: varchar("supplier_code", { length: 50 }),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow(),
}, (t) => [uniqueIndex().on(t.buyerOrgId, t.supplierOrgId)]);
```

**Testing**:
- Integration: migrations apply
- Integration: create buyer org → supplier org → relationship → FK holds
- Unit: one supplier, 3 buyer relationships → all visible to supplier

#### 1.3 — Dual-Portal Authentication

**What**: Buyer portal and supplier portal with org-scoped auth; SSO support; role-based access.

**Design**:

Buyer roles: `procurement_admin`, `buyer`, `ap_clerk`, `quality_engineer`, `viewer`. Supplier roles: `supplier_admin`, `sales_rep`, `shipping_clerk`, `invoice_clerk`, `viewer`.

Routes: `/buyer/*` requires buyer org membership; `/supplier/*` requires supplier org membership. A user can belong to both a buyer and supplier org (for distributors who are both).

**Testing**:
- Unit: buyer user accessing `/supplier/*` → 403
- Unit: supplier user with `invoice_clerk` role can submit invoices → allowed
- Unit: supplier user cannot access buyer scorecards → 403

---

## Phase 2: Purchase Order Management

### Purpose
PO receipt, acknowledgement, and change management — the transactional backbone of buyer-supplier interaction.

### Tasks

#### 2.1 — PO Creation and Supplier Receipt

**What**: Buyer creates POs (via API or UI); supplier receives, views, and acknowledges.

**Design**:

```typescript
// Buyer: POST /api/v1/purchase-orders
interface POCreateRequest {
  supplierOrgId: string;
  poNumber: string;
  orderDate: string;          // ISO 8601
  deliveryDate: string;
  shippingAddress: AddressInput;
  lines: {
    lineNumber: number;
    productCode: string;
    description: string;
    quantity: number;
    unitPriceCents: number;
    uom: string;              // EA, CS, KG, etc.
    deliveryDate?: string;    // line-level delivery date
  }[];
  currencyCode: string;
  paymentTermsDays: number;
  notes?: string;
}

// PO state: draft → sent → acknowledged → partially_shipped → fully_shipped → closed (or cancelled)
// Supplier: POST /api/v1/purchase-orders/{id}/acknowledge
interface POAcknowledgeRequest {
  acknowledgedLines: {
    lineNumber: number;
    confirmedQuantity: number;
    confirmedDeliveryDate: string;
    notes?: string;
  }[];
}
```

PO numbers auto-generated per buyer org. Email notification to supplier on PO send.

**Testing**:
- Unit: create PO with 5 lines → PO record + 5 line records
- Unit: supplier acknowledges all lines → status = `acknowledged`
- Unit: supplier partially acknowledges → exception flagged for buyer review
- E2E: buyer creates PO → supplier receives notification → views PO in portal → acknowledges

#### 2.2 — PO Change Management

**What**: Buyer issues PO changes; supplier reviews and accepts/rejects changes.

**Design**:

```typescript
// POST /api/v1/purchase-orders/{id}/changes
interface POChangeRequest {
  changeType: "quantity_change" | "date_change" | "price_change" | "line_add" | "line_cancel";
  affectedLines: {
    lineNumber: number;
    newQuantity?: number;
    newDeliveryDate?: string;
    newUnitPriceCents?: number;
  }[];
  reason: string;
}

// Change state: proposed → accepted | rejected → applied
// All changes create audit trail records
```

**Testing**:
- Unit: quantity increase → change record created, supplier notified
- Unit: supplier accepts change → PO line updated
- Unit: supplier rejects change → buyer notified, original values retained
- Integration: full change cycle → audit trail with before/after values

---

## Phase 3: ASN and Invoice Management

### Purpose
Supplier creates ASNs for shipments and submits invoices; buyer receives, matches, and processes for payment.

### Tasks

#### 3.1 — Advance Ship Notice (ASN)

**What**: Supplier creates ASN against PO lines; buyer tracks inbound shipments.

**Design**:

```typescript
// Supplier: POST /api/v1/ship-notices
interface ASNCreateRequest {
  poId: string;
  shipDate: string;
  estimatedArrivalDate: string;
  carrier: string;
  trackingNumber?: string;
  lines: {
    poLineNumber: number;
    shippedQuantity: number;
    lotNumber?: string;
    serialNumbers?: string[];
  }[];
  packingListUrl?: string;     // uploaded document
}

// ASN state: created → in_transit → received → discrepancy (if qty mismatch)
```

**Testing**:
- Unit: ASN for 3 PO lines → 3 ASN line records
- Unit: shipped qty = PO qty → clean receipt expected
- Unit: shipped qty < PO qty → partial shipment flagged
- E2E: supplier creates ASN → buyer sees inbound shipment in dashboard

#### 3.2 — Invoice Submission and PO Matching

**What**: Supplier submits invoices; system auto-matches to PO/ASN; discrepancies flagged.

**Design**:

```typescript
// Supplier: POST /api/v1/invoices
interface InvoiceCreateRequest {
  invoiceType: "po_based" | "non_po" | "credit_note";
  poId?: string;              // for PO-based invoices
  invoiceNumber: string;
  invoiceDate: string;
  dueDate: string;
  lines: {
    poLineNumber?: number;
    description: string;
    quantity: number;
    unitPriceCents: number;
    taxCents: number;
  }[];
  totalCents: number;
  taxTotalCents: number;
  attachmentUrl?: string;     // PDF invoice
}

// Invoice state: submitted → matching → matched | discrepancy → approved → scheduled_for_payment → paid
```

Auto-matching logic (3-way match: PO → ASN → Invoice):
1. Match invoice lines to PO lines by poLineNumber
2. Verify: invoice qty ≤ received qty (from ASN)
3. Verify: invoice unit price = PO unit price (tolerance ±2%)
4. If all lines match → status = `matched`, auto-approve if below threshold
5. If discrepancy → flag for buyer review with specific mismatch details

**Testing**:
- Unit: invoice matches PO + ASN within tolerance → auto-matched
- Unit: invoice price 5% above PO → discrepancy flagged with "Price variance +5%"
- Unit: invoice qty > received qty → discrepancy "Invoiced exceeds received"
- Unit: non-PO invoice → manual review required (no auto-match)
- Integration: submit PO-based invoice → 3-way match → status = matched

---

## Phase 4: Payment Visibility and Supplier Dashboard

### Purpose
Give suppliers visibility into payment status and provide a unified supplier dashboard with all transactional data.

### Tasks

#### 4.1 — Payment Status and Remittance

**What**: Buyer updates payment status; supplier sees payment schedule, remittance advice, and payment history.

**Design**:

```typescript
// Buyer: POST /api/v1/payments
interface PaymentCreateRequest {
  invoiceIds: string[];
  paymentMethod: "ach" | "wire" | "check" | "virtual_card";
  paymentDate: string;
  totalCents: number;
  remittanceReference: string;
}

// Payment state: scheduled → processed → remittance_sent
// Supplier view: GET /api/v1/supplier/payments → list with invoice references, amounts, dates
```

**Testing**:
- Unit: payment for 3 invoices → invoices status = `paid`
- E2E: supplier views payment history → all paid invoices with dates and amounts

#### 4.2 — Supplier Portal Dashboard

**What**: Unified supplier dashboard showing open POs, pending ASNs, invoice status, payment schedule, and performance summary.

**Design**:

`/supplier/dashboard` page: summary cards (Open POs: 12, Pending Invoices: 5, Upcoming Payments: $45K, Performance Score: 87/100). Recent activity feed. Quick actions: Create ASN, Submit Invoice, View Payment Schedule.

**Testing**:
- E2E: supplier with 10 POs, 5 invoices → dashboard shows correct counts
- E2E: click "Create ASN" → ASN form pre-populated with open PO lines

---

## Phase 5: Supplier Performance Scorecards

### Purpose
Track and score supplier performance on delivery, quality, and responsiveness. Enables data-driven supplier management decisions.

### Tasks

#### 5.1 — Scorecard Engine

**What**: Compute supplier scorecards from transactional data; make scores visible to both buyer and supplier.

**Design**:

```typescript
interface SupplierScorecard {
  supplierId: string;
  period: string;                    // "2026-Q2"
  onTimeDeliveryPct: number;         // ASN arrival vs. PO delivery date
  qualityRejectRatePct: number;      // rejected items / total received
  invoiceAccuracyPct: number;        // auto-matched invoices / total invoices
  responsivenessScore: number;       // avg hours to acknowledge PO
  overallScore: number;              // weighted composite 0-100
  trend: "improving" | "stable" | "declining";
}

// Weights: on-time delivery (35%), quality (25%), invoice accuracy (20%), responsiveness (20%)
// Computed monthly via BullMQ scheduled task
```

Supplier self-service: suppliers see their own scorecard with trend charts. Buyer comparison: buyer sees all suppliers ranked by score.

**Testing**:
- Unit: supplier with 95% on-time, 2% reject rate → high overall score
- Unit: supplier with 60% on-time, 10% reject → low score, "declining" trend
- E2E: supplier views `/supplier/performance` → scorecard with trend chart
- E2E: buyer views `/buyer/scorecards` → ranked supplier list

---

## Phase 6: Quality Notifications and CAPA

### Purpose
Track quality issues and corrective actions between buyer and supplier. Lightweight quality management without full APQP suite.

### Tasks

#### 6.1 — Quality Notification and CAPA Workflow

**What**: Buyer creates quality notifications; supplier responds with corrective action plans; buyer verifies closure.

**Design**:

```typescript
// Buyer: POST /api/v1/quality/notifications
interface QualityNotificationCreate {
  supplierOrgId: string;
  poId?: string;
  type: "ncr" | "complaint" | "inspection_failure" | "field_failure";
  severity: "minor" | "major" | "critical";
  description: string;
  affectedQuantity: number;
  attachmentIds?: string[];
}

// Notification state: created → acknowledged → corrective_action_submitted → under_review → closed | reopened

// Supplier: POST /api/v1/quality/notifications/{id}/corrective-action
interface CorrectiveActionSubmit {
  rootCause: string;
  containmentAction: string;
  correctiveAction: string;
  preventiveAction: string;
  targetCompletionDate: string;
  attachmentIds?: string[];
}
```

CAPA follows 8D methodology structure (lightweight): containment, root cause, corrective action, preventive action, verification.

**Testing**:
- Unit: create NCR → notification created, supplier notified
- Unit: supplier submits CAPA → status = `corrective_action_submitted`
- Unit: buyer verifies and closes → status = `closed`, scorecard quality metric updated
- E2E: full 8D cycle in UI

---

## Phase 7: EDI and cXML Integration

### Purpose
Exchange PO, ASN, and invoice documents via standard EDI (X12 850/855/856/810) and cXML for ERP integration.

### Tasks

#### 7.1 — EDI X12 Document Processing

**What**: Parse inbound and generate outbound X12 850 (PO), 855 (PO Ack), 856 (ASN), 810 (Invoice).

**Design**:

```typescript
class X12Parser {
  parse850(ediContent: string): POCreateRequest { /* BEG, PO1, N1, CTT segments */ }
  parse855(ediContent: string): POAcknowledgeRequest { /* BAK, PO1, ACK segments */ }
  parse856(ediContent: string): ASNCreateRequest { /* BSN, HL, TD5, REF, MAN segments */ }
  parse810(ediContent: string): InvoiceCreateRequest { /* BIG, IT1, TDS, CTT segments */ }
}

class X12Generator {
  generate850(po: PurchaseOrder): string { /* Generate valid X12 850 */ }
  generate856(asn: ShipNotice): string { /* Generate valid X12 856 */ }
  generate810(invoice: Invoice): string { /* Generate valid X12 810 */ }
}
```

EDI exchange: file-based (poll inbound directory) + REST API for direct submission.

**Testing**:
- Unit: parse sample 850 → correct PO fields extracted
- Unit: generate 856 → valid X12 with correct segments
- Fixture: `tests/fixtures/edi_850_sample.txt` etc.

#### 7.2 — cXML Document Support

**What**: Parse/generate cXML OrderRequest, ShipNoticeRequest, InvoiceDetailRequest for Ariba/Coupa interop.

**Design**:

cXML → internal model mapping via xml2js. Support for PunchOut setup (future) but MVP focuses on document exchange.

**Testing**:
- Unit: parse cXML OrderRequest → correct PO fields
- Unit: generate cXML InvoiceDetailRequest → valid cXML schema

---

## Phase 8: AI-Native Features (v1.1)

### Purpose
Add AI differentiators: invoice PDF extraction, supplier risk monitoring, and natural-language query interface.

### Tasks

#### 8.1 — AI Invoice PDF Extraction

**What**: Extract structured invoice data from uploaded PDF invoices using GPT-4o Vision.

**Design**:

```typescript
class InvoiceExtractor {
  SYSTEM_PROMPT = `Extract structured invoice data from this PDF image.
    Return JSON: { invoiceNumber, invoiceDate, vendorName, poNumber (if referenced),
    lines: [{ description, quantity, unitPrice, totalPrice }],
    subtotal, taxAmount, totalAmount, currency }`;

  async extract(pdfUrl: string): Promise<ExtractedInvoice> {
    // 1. Render PDF to images (pdf.js)
    // 2. Send to GPT-4o Vision for extraction
    // 3. Validate extracted fields against supplier profile
    // 4. Return structured data for review before submission
  }
}
```

Human-in-the-loop: extracted data presented for supplier review before formal submission.

**Testing**:
- Unit (mocked): sample invoice PDF → correct fields extracted
- Unit (mocked): ambiguous invoice → low-confidence fields flagged for review
- E2E: upload PDF → extracted data in review form → confirm → invoice submitted

#### 8.2 — Supplier Risk Monitoring

**What**: Continuous risk scoring from news, financial, and ESG signals with plain-English summaries.

**Design**:

```typescript
class RiskMonitor {
  async assessRisk(supplierOrgId: string): Promise<RiskAssessment> {
    // 1. Fetch recent news about supplier (GDELT or news API)
    // 2. Check financial signals (payment delays to other buyers if available)
    // 3. Claude summarises risk factors in plain English
    // 4. Compute risk score (0-100, higher = more risk)
    // 5. Store assessment, notify buyer if score crosses threshold
  }
}

interface RiskAssessment {
  supplierId: string;
  riskScore: number;
  riskTier: "low" | "medium" | "high" | "critical";
  summary: string;               // "Supplier X reported Q1 revenue decline of 15%..."
  riskFactors: { factor: string; severity: string; source: string }[];
  assessedAt: string;
}
```

BullMQ job: weekly risk scan for all active suppliers.

**Testing**:
- Unit (mocked): supplier with negative news → risk score increased, summary generated
- Unit (mocked): supplier with no news → risk score stable
- Integration: weekly scan → risk assessments stored, alerts sent for high-risk

#### 8.3 — Natural-Language Query Interface

**What**: Buyers and suppliers query PO/invoice/payment status in plain language.

**Design**:

```typescript
// POST /api/v1/ai/query
// Request: { "question": "When will invoice INV-2026-0042 be paid?" }
// Response: { "answer": "Invoice INV-2026-0042 for $12,500 was approved on May 25 and is scheduled for payment on June 5 via ACH.", "data": {...} }
```

Claude receives: user's org context (buyer or supplier), relevant PO/invoice/payment data, scorecard data.

**Testing**:
- Unit (mocked): "status of PO-1234" → returns PO status with line details
- Unit (mocked): "unpaid invoices over $10,000" → returns filtered invoice list
- Unit (mocked): supplier asks "my performance score" → returns current scorecard

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Purchase Order Management     ─── requires Phase 1
    │
Phase 3: ASN & Invoice Management      ─── requires Phase 2
    │
Phase 4: Payment & Supplier Dashboard   ─── requires Phase 3
    │
Phase 5: Supplier Scorecards           ─── requires Phase 3
    │
Phase 6: Quality & CAPA                ─── requires Phase 2 (parallel with Phases 3-5)
    │
Phase 7: EDI & cXML Integration       ─── requires Phase 2 (parallel with Phases 3-6)
    │
Phase 8: AI-Native Features           ─── requires Phases 3 + 5
```

Parallelism opportunities:
- Phases 5 and 6 can be developed concurrently with Phases 3-4
- Phase 7 can be developed concurrently with Phases 3-6 (EDI is an alternative input method)
- Phase 8 tasks can be developed independently after their data dependencies

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specification.
2. All unit and integration tests pass (`vitest run`).
3. Biome linting passes with zero warnings.
4. TypeScript compiles in strict mode with zero errors.
5. Docker build succeeds for API and frontend.
6. `docker-compose up` brings all services to healthy state.
7. Feature works end-to-end from both buyer and supplier portal perspectives.
8. 3-way invoice matching (PO→ASN→Invoice) produces correct results.
9. New API endpoints appear in auto-generated OpenAPI spec.
10. Database migrations created and tested.
11. Role-based permissions enforced for both buyer and supplier roles.
12. EDI/cXML documents validate against standard segment structures (where applicable).
13. New environment variables documented in config.ts.
