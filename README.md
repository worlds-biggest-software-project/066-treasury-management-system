# Treasury Management System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native treasury platform that brings real-time cash positioning, ML-driven forecasting, and Open Banking connectivity to the mid-market companies that legacy TMS vendors ignore.

Treasury Management System is a candidate open-source TMS targeting CFOs, treasurers, and treasury analysts at companies in the $50M–$5B revenue range. It addresses the largest single open-source gap in the financial software landscape: there is no viable OSS TMS today offering cash positioning, FX risk management, or AI forecasting. The mid-market currently runs treasury operations on spreadsheets and bank portals while Kyriba and GTreasury — holding 80%+ combined market share — remain priced for the enterprise tier only.

---

## Why Treasury Management System?

- **No viable open-source alternative exists.** ERPNext provides only basic bank reconciliation and cash flow reporting; it has no real-time cash positioning, no FX risk management, no multi-bank aggregation, and no ML forecasting. This is the most complete OSS gap among the financial software domains surveyed.
- **The mid-market is unserved.** Kyriba is enterprise-only ($500M+ revenue in practice) with implementation measured in 6–12 months and custom modular pricing. GTreasury starts around $300M revenue. The $50M–$500M segment is left running treasury on spreadsheets.
- **Legacy moats are eroding.** Kyriba's 69.87% share rests on its 9,900+ bank SWIFT connectivity network. Open Banking APIs (FDX in the US, PSD2 in Europe, CDR in Australia) are publicly documented standards that bypass SWIFT — the primary switching cost locking enterprises to legacy vendors.
- **AI adoption is early but ROI is proven.** 82% of corporate treasury teams are still in the AI exploration phase (Capgemini 2025), yet peer-reviewed research (MDPI 2025) shows ML forecasting reduces cash flow prediction error by 37.2%, and BCG estimates $1.2M in capital efficiency improvement per $1B revenue from sophisticated predictive models.
- **Commercial AI forecasting is opaque.** All commercial cash forecasting models are black boxes. An OSS approach with inspectable models directly addresses audit and regulatory concerns in treasury functions.

---

## Key Features

### Multi-Bank Cash Positioning

- Real-time consolidated view of all bank accounts, entities, and currencies
- Open Banking API connectivity (FDX, PSD2, CDR) for live balance feeds without SWIFT file processing
- CSV import fallback for banks not yet supporting modern APIs
- Multi-currency position tracking with real-time FX rate integration
- Bank reconciliation with automated matching of bank statement items to GL transactions

### AI/ML Cash Flow Forecasting

- Rolling 13-week and 12-month projections with actuals overlay
- ML models (random forest, LSTM) trained on historical cash flows, with documented 37% lower error rates than traditional methods
- Inspectable, transparent forecasting models — addressing the black-box concern of commercial vendors
- Scenario planning and stress testing against FX movements and interest rate changes
- Liquidity anomaly detection flagging unusual cash movements before they compound

### Payments & Bank Connectivity

- Centralised payment execution with dual-approval workflow
- Bank submission via Open Banking payment initiation
- Optional SWIFT MT940/MT942 statement import and MT101 payment initiation for legacy bank coverage
- ISO 20022 CAMT.053/054 message processing for modern bank connectivity
- ERP integration for GL-source cash flow data (ERPNext, NetSuite, QuickBooks)

### FX Risk & Treasury Intelligence

- FX exposure monitoring: net currency exposure by entity with configurable alert thresholds
- Autonomous monitoring of cross-currency exposures with hedging scenario simulation against real-time FX rates
- Debt covenant compliance monitoring tracking financial ratios against covenant terms
- LLM-powered natural language treasury queries over the live cash data model ("How much cash do we have in Europe today?")

### Hedge Accounting & Advanced Instruments (Backlog)

- FX hedging workflow: exposure-to-hedge matching with trade confirmation generation
- Hedge accounting support under ASC 815 / IFRS 9 with effectiveness test automation
- Working capital optimisation: supply chain finance and dynamic discounting

---

## AI-Native Advantage

AI is architecturally integrated, not bolted on. Inspectable LSTM and random forest forecasting models deliver the 37% error reduction documented in peer-reviewed research while remaining auditable — a capability commercial black-box vendors cannot offer. An LLM layer over the structured cash data model answers recurring treasurer questions in natural language without custom report building. Continuous AI monitoring of FX exposures and debt covenants replaces the weekly spreadsheet analysis that dominates mid-market treasury work today, and ML-driven liquidity anomaly detection flags fraud, timing errors, and operational disruptions in real time.

---

## Tech Stack & Deployment

The platform is designed API-first around Open Banking standards (FDX, PSD2, CDR) — bypassing SWIFT's proprietary network entirely for banks that support modern connectivity, while retaining SWIFT MT/MX and ISO 20022 CAMT message support as a fallback for legacy bank coverage. Expected deployment modes are self-hosted (on-premises or private cloud) and managed cloud, with REST APIs for all treasury transactions and connectors to ERPNext, NetSuite, and QuickBooks for GL-source data. ISDA-compliant trade confirmation generation is planned for the hedging module.

---

## Market Context

The global TMS market is sized between $5.81B and $12.5B in 2024 depending on definitional scope, projected to reach $15.1B–$25.5B by 2032–2033 at 11.95%–12.84% CAGR (Business Research Insights; Coherent Market Insights). Kyriba and GTreasury hold a combined ~80% share of the enterprise tier (6sense), with custom enterprise pricing and bespoke implementations costing $400K–$1M; Trovata starts at ~$2,400/year as the most accessible commercial option. Primary buyers are treasurers and CFOs at mid-enterprise ($500M–$5B) and mid-market ($50M–$500M) companies, with treasury analysts as daily users.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
