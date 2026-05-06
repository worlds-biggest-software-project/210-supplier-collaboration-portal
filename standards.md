# Standards & API Reference

> Project: Supplier Collaboration Portal · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Supplier portals tracking quality reject rates, corrective actions, and audit outcomes are directly aligned with ISO 9001 clauses on supplier control and measurement. Buyers often require ISO 9001 certification as a pre-qualification criterion managed through the portal.

**ISO 20400:2017 — Sustainable Procurement**
- URL: https://www.iso.org/standard/63026.html
- Provides guidance for embedding sustainability criteria (environmental, social, governance) into procurement and supplier management processes. Supplier portals collecting ESG questionnaires, certifications, and sustainability scores implement the due diligence framework prescribed by this standard.

**ISO 28000:2022 — Security and Resilience — Security Management Systems**
- URL: https://www.iso.org/standard/79612.html
- Specifies requirements for security management systems covering aspects critical to supply chain security assurance: financing, manufacturing, information management, and transportation. Relevant to supplier portals storing sensitive financial, banking, and logistics data for millions of suppliers.

**ISO 31000:2018 — Risk Management — Guidelines**
- URL: https://www.iso.org/standard/65694.html
- Provides a universal framework for risk identification, assessment, treatment, and monitoring. Supplier risk scoring and monitoring modules in collaboration portals implement ISO 31000 principles when evaluating supplier financial health, geopolitical exposure, and compliance risk.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Specifies requirements for establishing an information security management system. Essential compliance baseline for any cloud-hosted supplier portal processing sensitive commercial and financial data across multiple tenants.

---

### W3C & IETF Standards

**RFC 9110 — HTTP Semantics (2022)**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- The foundational HTTP specification governing request/response semantics, methods (GET, POST, PUT, PATCH, DELETE), and status codes used by all REST APIs in supplier portal platforms.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The authorisation framework used by SAP Ariba, Coupa (OIDC/OAuth 2.0), Jaggaer, and Ivalua for API authentication. Specifies authorisation code flow (for user-delegated access), client credentials flow (for service-to-service), and token lifecycle management.

**RFC 8414 — OAuth 2.0 Authorization Server Metadata**
- URL: https://www.rfc-editor.org/rfc/rfc8414
- Defines the well-known endpoint for OAuth 2.0 server discovery, used in modern supplier portal API integrations to enable dynamic client registration and token endpoint discovery.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. Used by Coupa and Ivalua for API authentication and SSO integration. Standardises user identity claims (sub, email, name) enabling supplier portal users to authenticate via enterprise identity providers.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://www.rfc-editor.org/rfc/rfc7807
- Defines a standard JSON format for machine-readable error responses from HTTP APIs. Adopted by modern REST APIs in procurement platforms to convey error codes, titles, and diagnostic detail in a consistent, parseable structure.

**W3C Verifiable Credentials Data Model 2.0**
- URL: https://www.w3.org/TR/vc-data-model-2.0/
- Emerging standard for digitally signed, tamper-evident credentials. Increasingly relevant to supplier portal document management (ISO certifications, auditor reports, compliance attestations) and GS1 EPCIS traceability scenarios requiring cryptographic proof of origin.

---

### Data Model & API Specifications

**cXML 1.2 (Commerce XML)**
- URL: http://cxml.org/
- XML-based standard for B2B procurement document exchange, originally created by SAP Ariba (1999) and now widely adopted across the industry. Covers PunchOut catalogues, purchase orders, order confirmations, advance ship notices, and invoices. Current version 1.2.069 (February 2026). Used by SAP Ariba, Coupa, Jaggaer, GEP SMART, and Ivalua as their primary structured document exchange format.

**ANSI X12 EDI Standards (850, 855, 856, 810, 860)**
- URL: https://www.x12.org/
- The dominant US EDI transaction set standards governing the most common supplier collaboration documents: 850 (Purchase Order), 855 (PO Acknowledgement), 856 (Advance Ship Notice), 810 (Invoice), 860 (PO Change Request). SPS Commerce and SAP Ariba rely heavily on X12 for retail and manufacturing supply chains.

**UN/EDIFACT**
- URL: https://unece.org/trade/uncefact/introducing-unedifact
- International EDI standard equivalent to ANSI X12, used in European and global supply chains. ORDERS, ORDRSP, DESADV, and INVOIC message types correspond to the X12 850/855/856/810 equivalents. Required for global supplier portals operating across regions.

**UN/CEFACT Cross-Industry Invoice (CII)**
- URL: https://tfig.unece.org/contents/cross-industry-invoice-cii.htm
- XML-based e-invoicing standard from UN/CEFACT; mandatory syntax option in European EN 16931 (the EU e-invoicing semantic standard). Alongside UBL, CII is required for Peppol network invoicing and for compliance with EU e-invoicing mandates affecting suppliers in EU member states.

**Peppol BIS Billing 3.0**
- URL: https://docs.peppol.eu/poacc/billing/3.0/bis/
- The Peppol Business Interoperability Specification for electronic billing, supporting both UBL and UN/CEFACT CII syntax. Governs cross-border e-invoicing across public and private sector procurement in 40+ countries. Supplier portals serving EU buyers must support Peppol Access Point connectivity.

**OAGIS 10.x (Open Applications Group Integration Specification)**
- URL: https://www.openapplications.org/
- Global XML business document standard covering enterprise-wide processes including purchase orders, invoices, ship notices, and supplier profile data. Used in manufacturing ERP integrations (SAP, Oracle) as the canonical message format. OAGIS 10.1 is used as the global standard XML format in several enterprise supply chain collaboration platforms.

**GS1 EPCIS 2.0 & CBV**
- URL: https://www.gs1.org/standards/epcis
- GS1's flagship data sharing standard for supply chain event visibility — capturing "what, when, where, why and how" for products and assets across the supply chain. EPCIS 2.0 added REST/JSON-LD and linked data support alongside the legacy SOAP/XML interface. Critical for Digital Product Passports, EU ESPR compliance, pharmaceutical DSCSA, and food traceability regulations.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for describing RESTful APIs in machine-readable format. Modern supplier portal APIs (Coupa, Ivalua, Jaggaer) publish OpenAPI 3.x specifications enabling code generation, automated testing, and API gateway governance.

**GraphQL Specification**
- URL: https://spec.graphql.org/
- Query language and runtime for APIs enabling clients to request exactly the data they need. Coupa has introduced GraphQL alongside its REST API. Particularly valuable for supplier portal integrations querying complex supplier performance and spend data with variable field sets.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) + PKCE (RFC 7636)**
- URL: https://www.rfc-editor.org/rfc/rfc7636
- PKCE (Proof Key for Code Exchange) is the mandatory extension to OAuth 2.0 authorisation code flow for public clients (mobile apps, SPAs) accessing supplier portal APIs. Required for secure supplier-side mobile integrations.

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf
- XML-based SSO federation standard widely used in enterprise procurement for connecting supplier portal identity to buyer-side identity providers (Active Directory, Okta, Ping). Supported by Ivalua, GEP SMART, and SAP Ariba for enterprise buyer SSO.

**OWASP Top 10 (2021)**
- URL: https://owasp.org/Top10/
- The industry reference for web application security risk. Supplier portals handling financial transactions, banking data, and commercial contracts must address injection, broken authentication, insecure direct object references, and supply chain attack vectors documented in the OWASP Top 10.

**NIST SP 800-63B — Digital Identity Guidelines: Authentication**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- NIST's authoritative guidance on authenticator assurance levels (AAL1/2/3), password policy, multi-factor authentication requirements. Relevant to supplier portal identity assurance for financial transactions and regulatory compliance in US government supply chains.

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- URL: https://gdpr-info.eu/
- EU regulation governing the processing of personal data. Supplier portals storing supplier company officer details, banking beneficiary information, and contact data across EU operations must implement data subject rights, lawful basis documentation, and cross-border data transfer mechanisms (Standard Contractual Clauses or adequacy decisions).

**EU Regulation 2024/1781 (ESPR) — Ecodesign for Sustainable Products**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1781
- Mandates Digital Product Passports for products sold in the EU, requiring product lifecycle and sustainability data from suppliers. Supplier portals will need to collect, validate, and expose DPP-compliant data through a GS1 EPCIS-compatible event chain by the 2030 product category rollout deadlines; 2026 is the implementation planning year for most sectors.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- Open protocol for connecting AI models to external tools, data sources, and APIs. Relevant to AI-native supplier portal features: an MCP server exposing supplier performance data, PO status, and invoice history would enable LLM-powered procurement assistants to query live portal data using natural language without custom integration code.

---

## Similar Products — Developer Documentation & APIs

### SAP Ariba / SAP Business Network
- **Description:** The largest procurement network globally (5M+ suppliers). Covers PO collaboration, ASN, invoicing, sourcing events, contracts, and supplier lifecycle management.
- **API Documentation:** https://developer.ariba.com/ and https://api.sap.com/package/SAPAribaOpenAPIs/rest
- **SDKs/Libraries:** SAP Integration Suite connectors; SAP-samples on GitHub: https://github.com/SAP-samples/ariba-extensibility-samples
- **Developer Guide:** https://help.sap.com/docs/ariba-apis
- **Standards:** REST/JSON, cXML, ANSI X12 EDI, OAGIS; OpenAPI-described endpoints on SAP API Hub
- **Authentication:** OAuth 2.0 (SAP BTP-managed tokens); API key for some legacy endpoints

---

### Coupa
- **Description:** Business spend management platform with supplier portal, PO/invoice management, catalogue, and analytics. Widely deployed in mid-market and enterprise.
- **API Documentation:** https://compass.coupa.com/en-us/products/product-documentation/integration-technical-documentation/the-coupa-core-api
- **SDKs/Libraries:** No official SDK; REST and GraphQL client libraries available via community. Pre-built ERP adaptors documented at https://compass.coupa.com
- **Developer Guide:** https://compass.coupa.com/en-us/products/product-documentation/integration-technical-documentation/the-coupa-core-api/get-started-with-the-api
- **Standards:** REST/JSON, GraphQL, cXML, CSV/SFTP; OpenAPI descriptions available
- **Authentication:** OAuth 2.0 / OpenID Connect (OIDC); X-COUPA-API-KEY header required on all requests

---

### Jaggaer ONE
- **Description:** AI-powered source-to-pay platform with particularly deep direct-materials quality management (APQP, PPAP, 8D) and supply chain collaboration.
- **API Documentation:** https://asodocs.jaggaer.com/ (Advanced Sourcing Optimizer APIs); https://www.jaggaer.com/wp-content/uploads/2024/06/JAGGAER-Integration-via-JAGGAER-Public-APIs.pdf
- **SDKs/Libraries:** Python client available on GitHub (UCLALibrary/jaggaer-api); JAGGAER Integration-as-a-Service for managed middleware
- **Developer Guide:** JAGGAER Supplier Integration Specification: https://www.jaggaer.com/app/uploads/2020/12/SupplierIntegrationSpecification-1.pdf
- **Standards:** REST/JSON (request/response and event-driven push), iDoc for legacy ERP, EDI, cXML
- **Authentication:** OAuth 2.0

---

### Ivalua
- **Description:** Highly configurable cloud source-to-pay platform with true two-way supplier collaboration, co-innovation tools, and no supplier fees.
- **API Documentation:** https://www.ivalua.com/solutions/business/supplier-collaboration-innovation/
- **SDKs/Libraries:** REST API with pre-built ERP connectors; no public SDK repository found
- **Developer Guide:** Available via Ivalua customer portal (not publicly indexed)
- **Standards:** REST/JSON, cXML, EDI, OpenAPI; SAML 2.0 and OIDC for SSO
- **Authentication:** OAuth 2.0 / OpenID Connect; SAML 2.0 for enterprise SSO

---

### GEP SMART
- **Description:** AI-first unified source-to-pay platform with agentic AI (GEP Quantum Intelligence), demand forecast sharing, and strong mid-market fit.
- **API Documentation:** https://www.gep.com/software/gep-smart/procurement-software
- **SDKs/Libraries:** GEP Integration Platform (managed); pre-built connectors for SAP, Oracle, Workday, NetSuite
- **Developer Guide:** Available via GEP customer portal; integration architecture documented at https://procurementaiagents.com/agents/gep-smart.html
- **Standards:** REST/JSON, cXML, EDI; OpenAPI-described endpoints
- **Authentication:** OAuth 2.0; API key for legacy integrations

---

### Kodiak Hub
- **Description:** Modular SRM platform for supplier onboarding, performance scorecards, risk monitoring, and ESG compliance — not a full S2P suite.
- **API Documentation:** https://www.kodiakhub.com/platform
- **SDKs/Libraries:** REST API for ERP and BI integration; third-party risk data feed connectors
- **Developer Guide:** Available via Kodiak Hub customer portal
- **Standards:** REST/JSON; OpenAPI-described endpoints
- **Authentication:** OAuth 2.0

---

### Zycus Supplier Network (ZSN)
- **Description:** Agentic AI-powered supplier network covering full source-to-pay with autonomous agent-driven supplier interaction.
- **API Documentation:** https://www.zycus.com/solution/supplier-network
- **SDKs/Libraries:** REST APIs; ERP connectors for SAP, Oracle, Microsoft Dynamics; analytics export connectors
- **Developer Guide:** Available via Zycus customer portal; overview at https://www.zycus.com/glossary/supplier-portal
- **Standards:** REST/JSON, cXML, EDI; OpenAPI-described endpoints
- **Authentication:** OAuth 2.0

---

### Graphite Connect
- **Description:** Supplier onboarding, identity verification, and risk platform with AI/OCR document scanning and cross-buyer shared supplier profiles.
- **API Documentation:** https://www.graphiteconnect.com/product/supplier-onboarding
- **SDKs/Libraries:** REST API for ERP synchronisation; integration with third-party identity and banking verification services
- **Developer Guide:** Available via Graphite customer portal
- **Standards:** REST/JSON; OpenAPI-described endpoints
- **Authentication:** OAuth 2.0; enterprise SSO support

---

### SPS Commerce
- **Description:** Full-service EDI network and supply chain collaboration platform for retail and CPG supply chains; MAX AI layer applies network intelligence to transaction data.
- **API Documentation:** https://www.spscommerce.com/products/fulfillment/edi/
- **SDKs/Libraries:** MAX Connect REST API for AI agent and ERP integration; pre-built connectors for major ERPs, WMS, and retail platforms
- **Developer Guide:** https://www.spscommerce.com/edi-guide/edi-solution/
- **Standards:** ANSI X12 EDI, REST/JSON (MAX Connect API); pre-built retail platform integrations (Amazon, Walmart, Target)
- **Authentication:** API key / OAuth 2.0 for REST API; EDI transport-layer security via AS2, SFTP

---

## Notes

**Emerging regulatory drivers (2026):** The EU Digital Product Passport (ESPR) and CSRD are creating new data collection requirements that will need to be embedded in supplier portals over the next 2–4 years. GS1 EPCIS 2.0's REST/JSON-LD interface makes it a strong candidate as the traceability backbone for DPP-compliant supplier data flows.

**API standardisation gap:** There is no cross-industry standard API for supplier portal data (e.g., no "OpenSupplier API" equivalent to OpenBanking). Each platform exposes a proprietary REST API, and integration is achieved through standards at the document level (cXML, EDI) rather than API schema level. An open-source AI-native portal could define and publish an OpenAPI 3.1 specification that becomes a community standard, increasing ecosystem interoperability.

**EDI modernisation:** The 2026 market is characterised by a hybrid EDI + REST/API model. Legacy EDI remains dominant in retail and automotive supply chains; REST APIs serve newer integrations and AI agent connectivity. New platforms should support both transports from day one.

**MCP opportunity:** No major supplier portal vendor has published an MCP server specification as of May 2026. Publishing an MCP-compatible API layer on an AI-native supplier portal would enable immediate integration with any LLM-powered procurement assistant, differentiating from incumbent closed-integration models.
