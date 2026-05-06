# Supplier Collaboration Portal — Feature & Functionality Survey

> Candidate #210 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| SAP Ariba (Business Network) | Source-to-pay suite + supplier network | Commercial SaaS — custom quote | https://www.sap.com/products/spend-management/ariba-network.html |
| Coupa Supplier Portal | Procure-to-pay suite + supplier portal | Commercial SaaS — custom quote | https://supplier.coupa.com/ |
| Jaggaer ONE | Source-to-pay suite — strong direct materials | Commercial SaaS — custom quote | https://www.jaggaer.com/ |
| Ivalua | Source-to-pay suite | Commercial SaaS — custom quote | https://www.ivalua.com/ |
| GEP SMART | AI-native source-to-pay suite | Commercial SaaS — custom quote | https://www.gep.com/software/gep-smart |
| Kodiak Hub | Supplier relationship management (SRM) | Commercial SaaS — custom quote | https://www.kodiakhub.com/ |
| Zycus (ZSN) | Source-to-pay + agentic AI | Commercial SaaS — custom quote | https://www.zycus.com/solution/supplier-network |
| Graphite Connect | Supplier onboarding & risk | Commercial SaaS — subscription | https://www.graphiteconnect.com/ |
| SPS Commerce | EDI / supply chain collaboration network | Commercial SaaS — subscription | https://www.spscommerce.com/ |

---

## Feature Analysis by Solution

### SAP Ariba / SAP Business Network

**Core features**
- Purchase order receipt, acknowledgement, and order-change management for suppliers
- Advance Ship Notice (ASN) creation including multi-line item shipment batching via Items to Ship tab
- PO-based and non-PO invoice submission; automated PO-to-invoice matching
- Invoice status tracking and payment remittance visibility
- Supplier profile management and network identity on the Ariba Network (5M+ suppliers)
- Centralised messaging and real-time notifications between buyer and supplier
- Catalogue management for suppliers publishing punchout or hosted catalogues
- Support for both light-touch (standard account) and deep integration (full EDI/cXML)
- Contract compliance checks on invoices against negotiated prices and terms

**Differentiating features**
- Largest procurement network in the world — many buyers mandate Ariba as their only portal
- Deep SAP S/4HANA integration means near-zero latency for ERP-side PO and GR status
- Multi-buyer support so a single supplier account connects to multiple buying organisations

**UX patterns**
- Tile-based dashboard with buyer-specific workspaces
- Email notification-driven workflow where suppliers act from notification links (low-friction for low-volume suppliers)
- Guided document wizard for ASN and invoice creation
- Mobile-responsive but not mobile-first

**Integration points**
- cXML and EDI (ANSI X12, EDIFACT) for transactional document exchange
- SAP Integration Suite for ERP-to-Ariba data flows
- REST APIs for supplier data, sourcing events, and contract data (SAP API Hub)
- SFTP batch file upload for high-volume document exchange

**Known gaps**
- UI widely criticised as complex and dated — steep learning curve for supplier staff
- Slow platform performance under large data loads (per G2 reviews)
- Integration with non-SAP ERPs requires expensive custom API development
- Supplier fees on certain Ariba Network tiers cause supplier resistance to adoption
- Limited strategic collaboration beyond transactional PO/invoice workflow

**Licence / IP notes**
- Proprietary SaaS; buyer licences SAP Ariba and suppliers may pay a transaction fee above a free tier. No open-source components. cXML protocol is openly published (cxml.org) but the platform itself is closed.

---

### Coupa Supplier Portal (CSP)

**Core features**
- Free supplier portal — no supplier fee for standard CSP access
- Purchase order management: view, acknowledge, flip to invoice
- Invoice creation from PO, contract, or time sheet; non-PO invoices supported
- Service sheets for milestone or time-and-materials services
- Supplier Actionable Notifications (SAN) — act on POs directly from email without portal login
- Catalogue upload (hosted and punchout cXML) for item suppliers
- Payment status and remittance visibility
- Coupa Advanced (premium): AI-powered InvoiceSmash PDF extraction, SSO, QuickBooks/NetSuite sync, cross-buyer invoice dashboard

**Differentiating features**
- SAN (Supplier Actionable Notifications) enables zero-barrier PO acknowledgement and invoicing direct from email — highest supplier adoption rates in category
- InvoiceSmash AI extracts non-PO invoices from PDF upload — reduces manual keying
- No supplier fees on the standard portal level, reducing onboarding friction

**UX patterns**
- Clean, modern interface; rated higher than SAP Ariba for ease of use on G2
- Progressive feature disclosure — basic users see a simple PO/invoice flow; power users access analytics and cross-buyer views
- Email-first interaction model (SAN) before portal engagement

**Integration points**
- REST API and GraphQL (Coupa Core API) for all platform objects including suppliers, POs, invoices
- OAuth 2.0 / OpenID Connect for API authentication; API key header required
- cXML for catalogue, PO, and invoice document exchange
- CSV/SFTP for batch data import/export
- Pre-built connectors for SAP, Oracle, NetSuite, Workday

**Known gaps**
- Supplier portal navigation confusing to new supplier users, causing invoice entry errors (per G2 reviews)
- Integration experience described as inflexible — Coupa's integration approach is opinionated and can be frustrating
- Mobile app has known performance and attachment issues
- Contract management less feature-rich than SAP Ariba (scored 6.8 vs 8.4 on G2)
- Limited visibility for suppliers into buyer-side approval status after invoice submission

**Licence / IP notes**
- Proprietary SaaS. CSP free to suppliers at standard tier; Coupa Advanced is a paid supplier subscription. Closed source.

---

### Jaggaer ONE

**Core features**
- Full source-to-contract and procure-to-pay lifecycle in one platform
- Direct materials focus: ERP-integrated demand/supply planning, scheduling agreements, delivery orders
- Quality management: APQP, PPAP, first article inspection (FAI), initial sample inspection reports (ISIR)
- 8D corrective action and root cause analysis workflow managed through supplier portal
- Supplier audit planning, execution, and documentation via portal
- Supplier scorecard with integrated quality, delivery, and sustainability metrics
- Sustainability and ESG ratings embedded in supplier profiles
- Supply chain collaboration portal used by more than 4 million suppliers

**Differentiating features**
- Deepest direct-materials quality management in the category (APQP/PPAP/8D workflows natively)
- Single ERP integration hub aggregating data from multiple ERPs to one collaboration layer
- Integrated audit management end-to-end through the supplier portal

**UX patterns**
- Role-based portal views — quality engineers see a different dashboard from category managers
- Audit and CAPA tasks surfaced as workflow items with deadlines and escalation
- ERP-driven transaction alerts push to supplier portal in real time

**Integration points**
- REST APIs (JSON messaging) with request/response and event-driven (push) modes
- JAGGAER iDoc for legacy ERP integration
- JAGGAER Integration-as-a-Service for managed connectivity
- SAP S/4HANA, Oracle, and other major ERP connectors available
- EDI and cXML for transactional document exchange

**Known gaps**
- Less brand recognition than SAP Ariba or Coupa — smaller network effect
- Quality management depth creates implementation complexity for indirect-spend-only organisations
- Advanced API-based workflows require dedicated developer resource (OAuth 2.0 and middleware proficiency)
- UI density can be overwhelming for occasional supplier users

**Licence / IP notes**
- Proprietary SaaS. PE-backed (Cinven). No open-source components. Quality management processes (APQP, 8D) follow industry standards (AIAG, VDA) which are public frameworks, not proprietary.

---

### Ivalua

**Core features**
- Unified supplier portal: onboarding, profile management, certification uploads, and renewals
- Two-way collaboration: suppliers can propose engineering changes, flag capacity issues, and submit corrective action plans
- Real-time PO, invoice, and payment status visible to suppliers
- Shared performance dashboards between buyer and supplier
- Co-innovation tools for capturing and evaluating supplier-initiated ideas
- Risk monitoring integration (financial, ESG, geopolitical) surfaced in supplier scorecard
- No supplier fees; 90%+ supplier enablement claimed

**Differentiating features**
- True two-way collaboration model beyond transactional PO/invoice — suppliers are active participants
- Co-innovation module for structured new-product-introduction (NPI) idea capture from supply base
- High configurability — one platform covering all spend categories and supplier tiers

**UX patterns**
- Branded portal per buyer organisation — suppliers see the buyer's identity, not Ivalua's
- Task-based inbox model directing supplier users to pending actions
- Guided onboarding wizard with conditional logic for different supplier categories

**Integration points**
- REST APIs for all platform objects
- Pre-built ERP connectors (SAP, Oracle, Microsoft Dynamics)
- SSO via SAML 2.0 and OpenID Connect
- cXML and EDI for document exchange

**Known gaps**
- Complex and lengthy implementation (common complaint in G2 reviews)
- Smaller supplier network than Ariba — less value from network discovery features
- AI capabilities still maturing compared to pure-play AI-native competitors

**Licence / IP notes**
- Proprietary SaaS. Significant growth equity raised. Closed source.

---

### GEP SMART

**Core features**
- Unified source-to-pay platform with AI-powered procurement portal
- Supplier portal: profile management, RFx participation, PO acknowledgement, invoice submission
- Demand forecast sharing with suppliers; supplier commitment to delivery schedules
- Two-way communication on quality issues and engineering changes
- Spend analytics and predictive analytics for demand forecasting and price budgeting
- Generative AI for sourcing event drafting, RFx creation, and natural-language spend queries
- Agentic AI (GEP Quantum Intelligence) coordinating decisions across source-to-pay lifecycle
- Mobile app with AI chatbot for PO approval, order status queries

**Differentiating features**
- GEP Quantum Intelligence — coordinated AI agents operating across the full procurement lifecycle
- Demand forecast sharing with suppliers through the portal (bridging procurement and supply planning)
- Strong mid-market fit with faster implementation than Ariba/Coupa

**UX patterns**
- AI-assisted natural language interface for buyers — plain-English queries for spend and supplier data
- Self-service portal customisation by procurement administrators
- Mobile-first design for field approvals and status queries

**Integration points**
- REST APIs and pre-built ERP connectors
- EDI, cXML for transactional document exchange
- GEP Integration Platform for managed connectivity
- SAP, Oracle, Workday, NetSuite connectors

**Known gaps**
- Smaller supplier network and reference base than Ariba/Coupa
- AI agentic features still maturing in production deployments
- Less deep direct-materials quality management than Jaggaer

**Licence / IP notes**
- Proprietary SaaS. Private equity backed. No open-source components.

---

### Kodiak Hub

**Core features**
- Modular SRM platform: intake, onboarding, risk monitoring, performance management, and collaboration
- 360° supplier scorecards aggregating quality (PPM, OTIF), ESG benchmarks, and compliance data
- Dynamic scorecard updates with automatic comparison against industry benchmarks
- Supply chain risk and resilience monitoring via continuous AI scan of live data feeds
- ESG and sustainability data collection from suppliers with scoring
- Risk monitoring for financial failures, reputational events, and compliance violations
- Supplier development workflows with assigned improvement actions and accountability tracking

**Differentiating features**
- Modular architecture — buyers select only the modules they need, enabling fast rollout
- Industry benchmark comparison built into supplier scorecards automatically
- Continuous AI-driven macro risk and ESG scanning without manual data entry

**UX patterns**
- Clean, analytics-forward dashboard with visual scorecard presentation
- Module-based progressive deployment — start with onboarding, expand to risk, then performance
- Supplier-facing collaboration workspace for data updates and document submission

**Integration points**
- REST APIs for data exchange with ERP and BI tools
- Pre-built connectors for major ERPs
- Third-party risk data feeds (financial health, ESG data providers)

**Known gaps**
- Not a full source-to-pay suite — no native PO/ASN/invoice transactional workflow
- Relies on integrations with separate ERP or procurement platforms for transactional data
- Smaller customer base and ecosystem than Ariba/Coupa/Jaggaer

**Licence / IP notes**
- Proprietary SaaS. Series A funded. Modular subscription model. No open-source components.

---

### Zycus Supplier Network (ZSN)

**Core features**
- AI-powered supplier network portal covering RFP/RFQ participation, PO management, contract, and invoice
- Agentic AI for automated data extraction, validation, and supplier profile enrichment
- Real-time visibility into POs, contracts, invoices across buyer organisations
- Intelligent supplier matching: matching suppliers to opportunities based on capability profiles
- AI compliance monitoring: continuous due diligence and risk flagging
- Integrated workflow optimisation connecting procurement workflows end-to-end
- Recognised in 2026 Gartner Magic Quadrant for Source-to-Pay Suites

**Differentiating features**
- Agentic AI at core of supplier interaction — AI agents negotiate with suppliers, run sourcing events, enforce compliance
- Capability-based intelligent matching connecting the right suppliers to the right opportunities

**UX patterns**
- AI-first interface — many routine interactions surfaced as AI recommendations or automated actions
- Real-time notification system for RFP/RFQ opportunities so suppliers never miss an event
- Unified inbox for all buyer interactions across the network

**Integration points**
- REST APIs for all platform objects
- EDI and cXML for transactional document exchange
- ERP connectors for SAP, Oracle, Microsoft Dynamics
- Analytics export and data warehouse connectors

**Known gaps**
- Smaller reference base than Ariba/Coupa despite Gartner recognition
- Agentic AI features are leading-edge — some organisations not ready for fully autonomous procurement agents
- Network effect weaker than SAP Ariba's 5M-supplier ecosystem

**Licence / IP notes**
- Proprietary SaaS. No open-source components. Agentic AI approach is proprietary.

---

### Graphite Connect

**Core features**
- Supplier onboarding platform: self-service supplier profile creation, document upload, and validation
- AI and OCR-based document scanning to verify TIN, EIN, company name, and banking information against uploaded documents
- Security waterfall: identity verification, fraud detection, and sanctions screening before supplier enters buyer systems
- Workflow routing: suppliers assign onboarding sections to the right internal stakeholder within their own organisation
- Internal team alerting and document review/approval workflow for buyer-side procurement and AP
- Multi-lingual portal (19+ languages)
- Full ERP integration for syncing verified supplier data
- Shared network profile: one supplier profile shared across multiple buyers on the Graphite network

**Differentiating features**
- AI-powered document fraud detection and identity verification — differentiates from form-only onboarding tools
- Network-shared supplier profile — suppliers maintain one record across all Graphite-connected buyers
- Claimed 85% reduction in onboarding time vs. manual processes

**UX patterns**
- Wizard-based self-service onboarding with section delegation
- Status-tracking dashboard for buyers to monitor where each supplier is in the onboarding funnel
- Alert-driven internal review workflow

**Integration points**
- ERP connectors for major platforms
- REST API for supplier data synchronisation
- Third-party identity and banking verification services

**Known gaps**
- Limited PO, ASN, and invoice transactional workflow — primarily an onboarding and risk tool
- Does not replace a full procurement or collaboration portal once supplier is onboarded
- Network discovery still developing compared to Ariba

**Licence / IP notes**
- Proprietary SaaS. Subscription model. No open-source components. Intel and UCSF are reference customers.

---

### SPS Commerce

**Core features**
- Largest retail-oriented EDI network (trading partner connectivity for PO, ASN, invoice)
- Fulfillment product: unified order management dashboard for POs, documents, and required workflow steps
- Pre-built EDI integrations for standardised data transfer across retail supply chains
- Full-service EDI management — SPS handles setup, mapping, and ongoing maintenance
- MAX Chat: AI network intelligence assistant for guided workflow queries
- MAX Monitor: continuous transaction monitoring with automated routine-work handling
- MAX Connect: network intelligence API layer for AI agents, ERPs, and data platforms
- 100% PO delivery guarantee (via SourceDay partnership)

**Differentiating features**
- Largest retail/CPG EDI trading partner network — critical mass in retail supply chain
- Full-service model — no developer resource required on buyer side for EDI maintenance
- MAX AI layer applied to network-scale transaction data — unique intelligence from volume

**UX patterns**
- Operations-focused dashboard highlighting what changed, what is required, and what to do next
- Notification-driven action model for buyers and suppliers
- Minimal supplier self-configuration — full-service setup by SPS team

**Integration points**
- EDI (ANSI X12, EDIFACT) as primary transport
- REST API (MAX Connect) for ERP and AI agent integration
- Pre-built connectors for major ERPs and WMS platforms
- Marketplace integrations for retail platforms (Amazon, Walmart, etc.)

**Known gaps**
- Primarily transactional EDI — limited strategic supplier performance management
- Retail/CPG-centric network — limited fit for manufacturing direct materials or services procurement
- Less suitable for strategic collaboration (scorecards, development plans, ESG) vs. SRM-focused tools

**Licence / IP notes**
- Proprietary SaaS. Public company (SPSC). Subscription model. EDI standards (ANSI X12) are open standards published by ASC X12.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Purchase order receipt, acknowledgement, and status tracking through the supplier portal
- Invoice creation and submission (PO-based, non-PO, contract-based), with matching validation
- Advance Ship Notice (ASN) creation and submission
- Supplier profile and document management (certifications, banking details, tax documents)
- Payment status and remittance visibility
- Notification and alerting system for pending actions (PO changes, invoice rejections, new opportunities)
- Basic supplier performance scorecard (on-time delivery, quality metrics)
- EDI / cXML / REST API integration with buyer ERP systems
- User role and permission management within the portal

### Differentiating Features
- AI-powered document extraction and validation (InvoiceSmash, Graphite OCR)
- Agentic AI for autonomous supplier interaction and compliance enforcement (Zycus)
- Network-shared supplier identity across multiple buyers (Ariba Network, Graphite, SPS)
- Two-way innovation and engineering change collaboration with suppliers (Ivalua, GEP)
- Deep direct-materials quality management: APQP, PPAP, 8D, ISIR (Jaggaer)
- Continuous AI-driven supply chain risk and ESG monitoring (Kodiak Hub, Zycus)
- Supplier capability matching and intelligent sourcing opportunity routing (Zycus, GEP)
- Digital Product Passport and CSRD-linked sustainability data collection (emerging)

### Underserved Areas / Opportunities
- Plain-language natural language interface for suppliers to query PO status, invoice history, and payment ETA without navigating complex portal UIs
- Unified cross-buyer supplier identity that is truly portable (not locked to one network like Ariba)
- Proactive supplier risk early-warning with explainable AI reasoning, not just risk scores
- Automated on-time delivery variance explanation (ML root-cause tagging of delays)
- ESG and Digital Product Passport data collection embedded into transactional PO workflows — not as a separate compliance module
- Supplier self-service analytics — performance trends, benchmark comparison — accessible to the supplier, not only the buyer
- Lightweight, fast-to-deploy collaboration for SME suppliers who resist large-network registration fees or complex onboarding
- Corrective action and CAPA workflow that is accessible without full quality management suite deployment

### AI-Augmentation Candidates
- Invoice matching and discrepancy resolution — today largely rules-based; ML can handle edge cases and learn from dispute history
- Supplier risk scoring — today driven by periodic manual data pulls; AI can run continuously against live news, financial, and ESG feeds
- Supplier onboarding document validation — today manual review; AI OCR and entity verification can automate most checks
- Scorecard data aggregation — today requires manual export from ERP and spreadsheet assembly; AI agents can compile and narrate
- Demand forecast sharing and commitment reconciliation — today largely manual email exchange; AI can automate variance detection and escalation
- Natural language procurement assistant — today absent from most portals; LLM can serve buyers and suppliers alike

---

## Legal & IP Summary

All tools analysed are proprietary commercial SaaS platforms with closed-source codebases. No open-source alternatives with comparable depth were identified in this category. The underlying transactional standards (ANSI X12 EDI, cXML, UN/CEFACT CII, GS1 EPCIS, OAGIS) are publicly published open standards and carry no IP risk for implementation. Quality management process frameworks (APQP, PPAP, 8D) are published by AIAG and VDA and are freely implementable. The Peppol network is governed by OpenPeppol, a non-profit body, with openly published BIS specifications. No patented features were identified in the public literature, though AI-specific methods (e.g. specific invoice extraction models, risk-scoring algorithms) are likely protected as trade secrets within each vendor's platform. An AI-native open-source implementation would have no IP barriers from the standards side.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Supplier portal with PO receipt, acknowledgement, and status tracking
- Invoice creation and submission (PO-based and non-PO) with automated matching validation
- ASN creation and shipment visibility
- Supplier profile and document management (certifications, banking, tax)
- Payment status and remittance visibility
- Basic supplier performance scorecard (on-time delivery, quality reject rate, responsiveness)
- REST API for ERP integration and cXML / EDI document exchange
- Role-based access control and multi-buyer support (one supplier account, many buyers)

**Should-have (v1.1)**
- AI-powered invoice PDF extraction for non-PO invoices (replacing manual keying)
- Continuous supplier risk monitoring with plain-English risk summaries
- Natural language query interface for PO/invoice/payment status
- Corrective action and CAPA workflow (lightweight, not full APQP suite)
- ESG/sustainability data collection embedded in supplier profile with scoring
- Supplier self-service performance analytics and benchmark comparison

**Nice-to-have (backlog)**
- Full direct-materials quality management (APQP, PPAP, ISIR, 8D) with ERP integration
- Two-way engineering change and new product introduction (NPI) collaboration workflow
- Digital Product Passport data collection linked to supplier-submitted shipment records
- Agentic AI for autonomous PO acknowledgement, ASN scheduling, and capacity commitment
- Network-scale supplier discovery and capability matching across buyer organisations
