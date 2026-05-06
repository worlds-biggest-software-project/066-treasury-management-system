# Standards & API Reference

> Project: Treasury Management System · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 20022 — Universal Financial Industry Message Scheme**
- URL: https://www.iso20022.org/iso-20022-message-definitions
- The foundational global standard for financial messaging. All treasury platforms must handle ISO 20022 message formats. The SWIFT network completed its ISO 20022 coexistence period in November 2025; from November 2026, fully structured address data is mandatory across all ISO 20022 messages. Any TMS built today should be ISO 20022 native rather than translating from legacy MT formats.

**ISO 20022 CAMT (Cash Management) Messages**
- URL: https://www.iso20022.org/iso-20022-message-definitions
- The CAMT family is the primary input for automated cash positioning in any TMS. Key message types:
  - `camt.052` — Intraday account report (replaces MT942); provides real-time balance updates during the banking day
  - `camt.053` — End-of-day bank account statement (replaces MT940); the primary input for daily cash positioning
  - `camt.054` — Bank-to-customer debit/credit notification (replaces MT900/MT910); individual transaction alerts
  - `camt.056` — FI-to-FI payment cancellation request
  - `camt.029` — Resolution of investigation; used for payment status tracking workflows
- The migration from MT940/MT942 bank statements to camt.052/camt.053 is the single most important integration migration in corporate treasury technology in 2025–2026.

**ISO 20022 PAIN (Payments Initiation) Messages**
- URL: https://www.iso20022.org/iso-20022-message-definitions / https://www.cashbook.com/understanding-iso-20022-and-pain-messages-a-complete-guide/
- The PAIN family covers payment instruction and status reporting. Key message types:
  - `pain.001` — Customer credit transfer initiation (replaces MT101); used by corporates to initiate payments to their bank. Migration deadline: November 2026.
  - `pain.002` — Payment status report; bank acknowledgement and rejection notification
  - `pain.007` — Customer payment reversal request
  - `pain.008` — Customer direct debit initiation
  - `pain.013` — Creditor payment activation request (Request to Pay)
- Full replacement of MT101 with pain.001 by November 2026 is mandatory on the SWIFT network.

**ISO 20022 PACS (Payments Clearing and Settlement)**
- URL: https://www.iso20022.org/iso-20022-message-definitions
- Used between financial institutions for interbank clearing; relevant to TMS platforms that need to parse payment confirmation messages or integrate with bank-side clearing infrastructure:
  - `pacs.008` — FI credit transfer (replaces MT103); the equivalent of a wire transfer between banks
  - `pacs.002` — FI payment status report; confirmation of payment settlement

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The base authorization standard for all Open Banking APIs (FDX, PSD2/Berlin Group NextGenPSD2, CDR). A TMS using API-first bank connectivity must implement OAuth 2.0 to obtain delegated authorization tokens to read account data and initiate payments. All major Open Banking frameworks mandate OAuth 2.0 as the authentication foundation.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the token format used in OAuth 2.0 and OpenID Connect flows across all Open Banking frameworks. TMS platforms handling multi-bank API connectivity will issue and validate JWTs for every bank connection session.

**RFC 8705 — OAuth 2.0 Mutual-TLS Client Authentication (mTLS)**
- URL: https://datatracker.ietf.org/doc/html/rfc8705
- Mutual TLS is required by FAPI 1.0 Advanced and FAPI 2.0 for high-security financial API connections. Required by UK Open Banking, Australian CDR, and Brazilian Open Finance. Any TMS connecting to regulated bank APIs must support mTLS certificate-bound access tokens.

**RFC 9126 — OAuth 2.0 Pushed Authorization Requests (PAR)**
- URL: https://datatracker.ietf.org/doc/html/rfc9126
- Required by FAPI 2.0; PAR requires the authorization request to be pre-registered at the authorization server before redirecting the user, improving security by preventing parameter injection. Required for FAPI 2.0-compliant Open Banking connections.

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The de facto REST API description standard across all major TMS vendors (Modern Treasury, Kyriba, Coupa, Trovata). An OSS TMS should publish an OAS 3.1 specification to enable SDK generation, Postman collections, and third-party integration tooling. OAS 3.1 aligns schema definitions with JSON Schema Draft 2020-12 for full interoperability.

---

### Data Model & API Specifications

**Financial-grade API (FAPI) 1.0 Advanced**
- URL: https://openid.net/specs/openid-financial-api-part-2-1_0.html / https://fapi.openid.net/
- Published by the OpenID Foundation's FAPI Working Group, this is the security profile mandated by UK Open Banking, Australian CDR, Brazil Open Finance, and the majority of regulated open banking standards globally. FAPI 1.0 Advanced requires: mTLS or private_key_jwt client authentication, Pushed Authorization Requests (PAR), and restricted OAuth 2.0 flows. Any TMS connecting to regulated European, Australian, or UK bank APIs must implement FAPI 1.0 Advanced.

**Financial-grade API (FAPI) 2.0 Security Profile**
- URL: https://openid.net/specs/fapi-security-profile-2_0-final.html
- The next generation of FAPI, FAPI 2.0 adds stronger non-repudiation requirements, transactional authorization (Rich Authorization Requests), and replay attack prevention. Colombia's SFC mandated FAPI 2.0 in 2024. Adoption is expanding; a forward-looking TMS architecture should target FAPI 2.0 compliance for new bank integrations.

**FDX API v6.5 — Financial Data Exchange**
- URL: https://financialdataexchange.org/ / https://developer.financialdataexchange.org/ (members)
- The US open finance standard body recognized by the CFPB as a Standard-Setting Body under Section 1033. FDX API v6.5 (released Q4 2025) defines RESTful APIs covering deposit accounts, loan accounts, investment accounts, tax data, and reward programmes. Over 130 million US consumer and business accounts connected as of Q1 2026. FDX compliance provides "indicia of compliance" for CFPB Section 1033's API documentation requirements. The primary standard for US bank API connectivity in any API-first TMS.

**Berlin Group NextGenPSD2 — XS2A Framework**
- URL: https://www.berlin-group.org/nextgenpsd2-downloads
- The EU's dominant open banking API framework, implemented by more than 75% of European banks. Defines OpenAPI-described REST APIs for Account Information Services (AIS) — accessing account balances, transaction history, and account owner data — and Payment Initiation Services (PIS) — initiating credit transfers and direct debits. Specifications published under Creative Commons (CC-BY-ND) and available at the Berlin Group download portal. Essential for any TMS targeting EU bank connectivity.

**Australian Consumer Data Right (CDR) — Banking API Standards**
- URL: https://consumerdatastandardsaustralia.github.io/standards/ / https://github.com/ConsumerDataStandardsAustralia/standards
- The Australian open banking standard maintained by the Data Standards Body (DSB), part of the Treasury. Defines OpenAPI-described REST APIs for account information access (balances, transactions, payees) with a FAPI 1.0 Advanced security profile. The CDR is extending to non-bank lenders from 2026. Required for any TMS targeting Australian bank connectivity.

**FpML (Financial products Markup Language) v5.x**
- URL: https://www.fpml.org/ / https://www.isda.org/isda-solutions-infohub/fpml/
- Published by ISDA, FpML is the open-source XML standard for electronically representing OTC derivatives: interest rate swaps, FX forwards, cross-currency swaps, options, and structured products. FpML defines business processes and data models for the full derivatives lifecycle: request for quote, confirmation, affirmation, novation, termination, and amendment. Required for any TMS module implementing hedge accounting or FX hedging workflows. FpML is an open standard; ISDA offers training courses and publishes all schemas openly.

---

### Security & Authentication Standards

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Applies to any TMS that handles or transmits payment card credentials or banking tokens. PCI DSS v4.0 (current) mandates: network segmentation for cardholder data, API authentication and authorisation controls, encryption of payment data in transit and at rest, and detailed audit logging of all payment access events. TMS platforms routing payments through banking APIs must be PCI DSS compliant.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework / https://nvlpubs.nist.gov/nistpubs/CSWP/NIST.CSWP.29.pdf
- The authoritative risk management framework for financial services technology. CSF 2.0 (published February 2024) adds a new "Govern" function and explicitly addresses supply chain security — critical for a TMS integrating with dozens of banking APIs, ERP connectors, and market data providers. NIST CSF 2.0 compliance and PCI DSS compliance are complementary (CSF defines what must be governed; PCI DSS defines specific controls for payment data).

**SWIFT gpi (Global Payments Innovation)**
- URL: https://www.swift.com/products/swift-gpi / https://developer-test.swift.com/apis/gpi-apis
- SWIFT gpi is the mandatory payment tracking standard for all SWIFT cross-border payments. Over 4,000 financial institutions send $300B+ per day via SWIFT gpi. The Unique End-to-End Transaction Reference (UETR) acts as a tracking ID for every cross-border payment. The SWIFT gpi API enables corporates to query real-time payment status across all clearing and settlement stages. A TMS integrating SWIFT for cross-border payments must implement the gpi tracking API to provide treasurers with real-time payment visibility.

**FX Global Code (GFXC, 2024 edition)**
- URL: https://www.globalfxc.org/fx-global-code/ / https://www.bis.org/about/factmktc/fx_global_code.htm
- Published by the Global Foreign Exchange Committee (GFXC) and maintained by the BIS, the FX Global Code establishes 55 principles for good practice in the OTC FX market. The December 2024 update introduced a "risk waterfall" approach for FX settlement risk mitigation (January 2025 GFXC update). While voluntary, the Code defines expected standards of conduct for TMS platforms executing FX transactions and hedges on behalf of corporate clients. Relevant to FX risk monitoring, hedging workflow, and audit trail design.

**ASC 815 (US GAAP) / IFRS 9 — Hedge Accounting Standards**
- URL: https://www.iasb.org/ifrs/ifrs-9 (IFRS 9) / https://fasb.org/page/document?pdf=ASC815.pdf (ASC 815)
- The accounting standards that govern how companies can record hedging transactions in their financial statements. ASC 815 (US GAAP) and IFRS 9 (International) both define three hedge types — fair value, cash flow, and net investment — and require documentation, designation, and effectiveness testing at inception and on an ongoing basis. IFRS 9 uses a principles-based "economic relationship" effectiveness test (replacing the old 80–125% bright-line); ASC 815 aligns closely with IFRS 9 in practice. Any TMS implementing hedge accounting must generate and store the required designation documentation and effectiveness test results in an audit-accepted format.

**ISDA Master Agreement / ISDA Credit Support Annex**
- URL: https://www.isda.org/
- The standard legal documentation framework for OTC derivatives trades. When a corporate treasury executes an FX forward or interest rate swap, it does so under an ISDA Master Agreement with its bank counterparty. A TMS must store ISDA-compliant trade confirmation data, track amendment and termination events, and manage the credit support annex (CSA) collateral calls. This is a data modelling and document management requirement as much as a technical integration requirement.

### MCP Server Specifications

**Anthropic Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- MCP defines a standard client–server interface through which AI assistants can access external tools, data sources, and actions. A treasury management system exposing an MCP server would allow AI agents (including LLM-powered treasury copilots) to query cash positions, retrieve bank statement data, initiate payment approvals, and run forecast scenarios via natural language — directly integrating with Claude, GPT-4, and other LLM runtimes. Given the strong AI-native opportunity identified for this project (LLM-powered treasury queries, autonomous FX monitoring), building an MCP server is a high-value differentiator for the OSS TMS.

---

## Similar Products — Developer Documentation & APIs

### Modern Treasury
- **Description:** US-headquartered payments operations platform providing a unified API for payment initiation (ACH, Wire, RTP), ledger management, counterparty verification, and reconciliation. Positioned for fintech teams embedding money movement in software products; also used by corporate treasury teams for payment automation.
- **API Documentation:** https://docs.moderntreasury.com/platform/reference/getting-started
- **SDKs/Libraries:** Node.js, Python, Go, Java/Kotlin, Ruby — https://docs.moderntreasury.com/platform/docs/sdks-libraries
- **OpenAPI Spec:** https://github.com/Modern-Treasury/modern-treasury-openapi (publicly available on GitHub)
- **Developer Guide:** https://docs.moderntreasury.com/
- **Standards:** REST/JSON, OpenAPI 3.1, full OpenAPI spec published on GitHub
- **Authentication:** API key (HTTP Basic Auth with organisation ID + API key); OAuth 2.0 for user-facing flows
- **Key Resources:** Payment Orders, Ledgers, Ledger Accounts, Ledger Transactions, Counterparties, External Accounts, Internal Accounts, Expected Payments (for automated reconciliation), Webhooks for event-driven integration

### Kyriba
- **Description:** Enterprise-grade TMS holding 69.87% market share in treasury management. Covers cash positioning, AI forecasting, FX risk, payments hub, working capital, and bank connectivity (9,900+ banks). The reference architecture for enterprise TMS feature completeness.
- **API Documentation:** https://developer.kyriba.com/ (registration required; sandbox access available)
- **SDKs/Libraries:** No public SDKs identified; REST API with JSON; IFS Cloud documented REST endpoints for payment file upload (ISO 20022 XML)
- **Developer Guide:** https://developer.kyriba.com/ — full API catalog, samples, and sandbox environment
- **Standards:** REST/JSON, ISO 20022 pain.001/camt.053/camt.054, SWIFT MT/MX, OpenAPI (internal)
- **Authentication:** OAuth 2.0 client credentials for system-to-system integrations
- **Key Resources:** Payments API (payment initiation, routing, approval workflow), Liquidity API (cash positions, balances, forecasts), Reference Data API (bank accounts, counterparties, entities), SWIFT file upload endpoint

### Goldman Sachs Transaction Banking (TxB)
- **Description:** Goldman Sachs's corporate cash management and payments platform, launched 2020. API-first from inception; designed for corporates and fintechs automating treasury operations. Covers virtual accounts, payment initiation, intraday/end-of-day reporting, and SWIFT gpi tracking. A reference architecture for bank-native treasury APIs.
- **API Documentation:** https://developer.gs.com/docs/services/transaction-banking/
- **SDKs/Libraries:** Documentation available via GS Developer Portal; no public third-party SDKs identified
- **Developer Guide:** https://developer.gs.com/discover/txb
- **Standards:** REST/JSON, ISO 20022 CAMT/PAIN, SWIFT gpi (UETR tracking), SFTP for file-based integrations
- **Authentication:** OAuth 2.0 with mTLS (documented at https://developer.gs.com/docs/services/transaction-banking/auth-connection/)
- **Key Resources:** Virtual Accounts, Payments (ACH, Wire, SWIFT), Transaction Reports, Monthly Statements, Intraday Balance Reporting, SWIFT Reporting (MT940/CAMT.053)

### Trovata
- **Description:** API-first cash reporting and forecasting platform. Maintains the largest catalogue of corporate banking connections globally across modern bank APIs, host-to-host (H2H) connectivity, and SWIFT. The reference architecture for multi-bank API connectivity aggregation in a TMS.
- **API Documentation:** https://trovata.io/api-platform/ (partner/enterprise access)
- **SDKs/Libraries:** MultiBank Connector SDK — described as "low-code with initialization as simple as adding a few lines of code"; intended for end-customer self-service connectivity
- **Developer Guide:** https://trovata.io/api-platform/ / https://trovata.io/multibank-connector
- **Standards:** FDX (US), PSD2/Berlin Group (EU), Open Banking (UK), SWIFT; REST/JSON
- **Authentication:** OAuth 2.0; FAPI 1.0 Advanced for regulated Open Banking connections
- **Key Resources:** Multi-bank account aggregation, Balance and Transaction Reporting, Cash Flow Tagging (ML-driven), Payment Processing (via ATOM acquisition), SWIFT network access for non-API banks

### Coupa Treasury (formerly Coupa Cash Management)
- **Description:** Treasury module within Coupa's broader spend management suite. Covers cash positioning, bank reconciliation, and payment execution. Strong integration with Coupa's AP/procurement data for working capital forecasting context.
- **API Documentation:** https://docs.coupa.com/en/developer-documentation/treasury-integrations/treasury-apis/view-treasury-api-documentation
- **SDKs/Libraries:** Coupa Core API framework; REST/JSON only; accessed from within the Treasury Management application UI
- **Developer Guide:** https://docs.coupa.com/en/developer-documentation
- **Standards:** REST/JSON; ISO 20022 CAMT/PAIN; SWIFT; the Treasury REST API follows the same structure as the Coupa Core API
- **Authentication:** OAuth 2.0
- **Key Resources:** Cash Positions, Bank Accounts, Bank Statements, Payments, Reconciliation

### Plaid
- **Description:** Financial data aggregation platform connecting applications to 12,000+ US, Canadian, and European banks. Not a TMS itself, but the primary infrastructure layer for US bank connectivity in fintech applications. Used by Trovata and similar platforms as the underlying data connectivity layer. Highly relevant as a bank connectivity component for an OSS TMS targeting the US market.
- **API Documentation:** https://plaid.com/docs/api/
- **SDKs/Libraries:** Official SDKs for Node, Python, Ruby, Go, Java — https://plaid.com/docs/libraries/
- **Developer Guide:** https://plaid.com/docs/
- **Standards:** REST/JSON, FDX v6.5, OAuth 2.0, TLS v1.2; FAPI-aligned for regulated connections
- **Authentication:** Client ID + Secret (HTTP headers or request body); OAuth 2.0 for user consent flows (Plaid Link)
- **Key Resources:** Transactions, Auth (bank account verification), Balance, Identity, Investments, Liabilities, Income; Sandbox environment (free, synthetic data), Development (100 free live Items), Production

### HSBC Global Banking API (including SWIFT gpi)
- **Description:** HSBC's developer API platform exposes corporate treasury services including SWIFT gpi payment tracking, account information, and payment initiation. Representative of major bank API portals that an OSS TMS would integrate with directly.
- **API Documentation:** https://develop.hsbc.com/api-overview/treasury-swift-gpi
- **SDKs/Libraries:** HSBC Developer Portal provides API specifications; no public SDKs
- **Developer Guide:** https://develop.hsbc.com/
- **Standards:** REST/JSON; ISO 20022; SWIFT gpi; OAuth 2.0
- **Authentication:** OAuth 2.0; FAPI 1.0 Advanced for regulated Open Banking connections
- **Key Resources:** SWIFT gpi payment tracking (UETR), Account Reporting, Payment Initiation

### GTreasury (ClearConnect API)
- **Description:** Enterprise TMS and Kyriba's closest competitor. ClearConnect is GTreasury's connectivity layer providing 80+ API call categories for bank connectivity. The ClearConnect Gateway is described as the fastest and most comprehensive banking API integration platform for corporate treasury.
- **API Documentation:** https://gtreasury.com/solutions/connectivity/ (enterprise access)
- **SDKs/Libraries:** Not publicly available; enterprise integrations via professional services
- **Developer Guide:** https://resources.gtreasury.com/rs/128-UQV-616/images/GTreasuryAPIOverview.pdf
- **Standards:** REST/JSON; ISO 20022 CAMT/PAIN; SWIFT MT/MX; FDX; PSD2; Open Banking UK
- **Authentication:** OAuth 2.0; enterprise SSO
- **Key Resources:** ClearConnect Gateway (real-time balance and transaction reporting), Payment Technologies (RTP, FedNow, ACH, SWIFT, SEPA), Bank Fee Data, Market Data, ERP Integration (SAP, Oracle, NetSuite, Dynamics 365)

---

## Notes

**SWIFT MT → ISO 20022 migration urgency (2026):** The November 2026 deadline for mandatory structured address data in ISO 20022 messages, and the migration from MT101 to pain.001 for payment initiation by the same date, means any TMS built today should be ISO 20022 native. Supporting SWIFT MT formats should be a fallback for legacy bank connections only, not the primary data model.

**Open Banking API coverage gaps:** FDX (US), Berlin Group NextGenPSD2 (EU), and CDR (Australia) cover the three major economic zones but coverage within each zone is incomplete. Some regional and community banks in the US do not yet support FDX APIs; a practical TMS must support CSV bank statement import as a fallback alongside API connectivity.

**FAPI 2.0 adoption trajectory:** FAPI 2.0 is mandated in Colombia (2024) and is increasingly being adopted in new Open Banking regulatory frameworks. A TMS targeting long-term international bank connectivity should plan for FAPI 2.0 compliance; FAPI 1.0 Advanced is sufficient for all current major markets (UK, EU, Australia, Brazil).

**FpML for MVP scope:** FpML is only relevant for TMS implementations that include FX hedging and derivatives trade confirmation. For the MVP scope (cash positioning, AI forecasting, payment execution), FpML is a post-MVP dependency. The features.md recommends FX hedging as a "nice-to-have (backlog)" feature, consistent with deferring FpML integration.

**MCP server as differentiator:** No existing commercial TMS exposes an MCP server. An OSS TMS that builds MCP server capabilities alongside its REST API would enable AI agent integration (treasury copilot, autonomous FX monitoring, natural language queries) that commercial platforms cannot match without significant replatforming.
