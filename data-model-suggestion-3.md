# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Treasury Management System · Created: 2026-05-12

## Philosophy

The hybrid relational + JSONB model uses traditional normalised tables for the core domain concepts that are stable and well-understood (bank accounts, payment orders, cash positions), but stores variable, jurisdiction-specific, and rapidly evolving data in PostgreSQL JSONB columns. This approach acknowledges that treasury management spans multiple jurisdictions, bank APIs, ERP systems, and regulatory frameworks — each with different data requirements — and that forcing all this variance into rigid relational columns leads to either hundreds of nullable columns or an unmanageable number of junction tables.

The core principle is: **if a field is queried, constrained, or joined on frequently, it is a relational column. If a field varies by jurisdiction, bank, or configuration, it goes in JSONB.** For example, every bank account has an `account_name`, `currency`, and `legal_entity_id` — these are relational. But the specific fields required for a PSD2 bank connection differ from an FDX connection differ from a SWIFT connection — these go in a `connection_config JSONB` column with GIN indexing for queries.

This pattern is widely used in production SaaS systems that must support multi-region deployments without per-region schema migrations. Stripe, Plaid, and Modern Treasury all use variations of this approach. The trade-off is that JSONB columns are less self-documenting than relational columns — the schema file does not fully describe the data — so disciplined JSON Schema validation and thorough example documentation are required to maintain clarity.

**Best for:** Rapid MVP development targeting multiple jurisdictions (US/FDX, EU/PSD2, AU/CDR) where bank connectivity, regulatory reporting, and ERP integration requirements vary by region, and the team needs to ship fast without schema migrations for every new bank or jurisdiction.

**Trade-offs:**
- Pro: Dramatically fewer tables (~25 vs. ~37+ in normalised) — faster to build and iterate
- Pro: Multi-jurisdiction support without schema migrations — new bank types, regulatory fields, and ERP formats are configuration, not DDL
- Pro: JSONB columns with GIN indexes support full query capability including containment queries
- Pro: Easier to add new features — new fields are added to JSONB payloads without ALTER TABLE
- Pro: Natural fit for API responses — JSONB columns can be returned directly to API consumers
- Con: JSONB columns lack schema-level type checking — validation must be enforced in application code or check constraints
- Con: Less self-documenting — developers must read documentation, not just the DDL, to understand JSONB field structures
- Con: GIN index performance degrades for very large JSONB documents (>10KB); careful field segregation is needed
- Con: Foreign key constraints cannot reference fields inside JSONB — referential integrity for JSONB-embedded IDs must be application-enforced
- Con: Reporting queries on JSONB fields require JSON path operators (`->>`, `@>`, `jsonb_path_query`) which are less familiar to SQL-only teams

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 20022 CAMT/PAIN | Core message fields (amount, currency, dates) are relational columns; message-version-specific fields and SupplementaryData go in JSONB |
| ISO 4217 | `currency` is always a relational CHAR(3) column, never JSONB |
| ISO 3166 | `country_code` is always a relational column |
| ISO 17442 (LEI) | `lei` is a relational column on `legal_entity` |
| FDX API v6.x | FDX-specific account fields stored in `bank_account.provider_data` JSONB |
| Berlin Group NextGenPSD2 | PSD2 consent fields stored in `bank_connection.connection_config` JSONB |
| Australian CDR | CDR-specific consent and scope fields stored in `bank_connection.connection_config` JSONB |
| SWIFT gpi | UETR stored as relational UUID column; gpi-specific tracking metadata in `payment_order.provider_data` JSONB |
| FpML | Derivative-specific fields stored in `fx_trade.instrument_data` JSONB |
| ASC 815 / IFRS 9 | Core hedge designation fields are relational; jurisdiction-specific accounting treatment details in JSONB |

---

## Organisation & Entities

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    -- Tenant-wide configuration in JSONB
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "supported_currencies": ["USD", "EUR", "GBP", "JPY"],
    --   "payment_approval_levels": 2,
    --   "forecast_default_horizon_weeks": 13,
    --   "fx_alert_threshold_pct": 5,
    --   "integrations": {
    --     "erp": {"type": "netsuite", "tenant_id": "NS-12345"},
    --     "slack_webhook": "https://hooks.slack.com/..."
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE legal_entity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES legal_entity(id),
    name            TEXT NOT NULL,
    short_name      TEXT,
    lei             VARCHAR(20),
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,
    functional_currency CHAR(3) NOT NULL,
    entity_type     TEXT NOT NULL CHECK (entity_type IN (
        'holding', 'operating', 'spv', 'branch', 'joint_venture'
    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Jurisdiction-specific regulatory fields
    regulatory_data JSONB NOT NULL DEFAULT '{}',
    -- Example regulatory_data (US entity):
    -- {
    --   "ein": "12-3456789",
    --   "state_of_incorporation": "DE",
    --   "sec_cik": "0001234567",
    --   "naics_code": "523110"
    -- }
    -- Example regulatory_data (EU entity):
    -- {
    --   "vat_number": "DE123456789",
    --   "trade_register": "HRB 12345",
    --   "competent_authority": "BaFin"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_entity_tenant ON legal_entity(tenant_id);
CREATE INDEX idx_entity_parent ON legal_entity(parent_id);
CREATE INDEX idx_entity_regulatory ON legal_entity USING GIN (regulatory_data);
```

## Bank Accounts & Connectivity

```sql
CREATE TABLE bank_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_name       TEXT NOT NULL,
    swift_bic       VARCHAR(11),
    country_code    CHAR(2),
    connection_type TEXT NOT NULL CHECK (connection_type IN (
        'fdx', 'psd2', 'cdr', 'swift', 'sftp', 'manual'
    )),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'active', 'expired', 'revoked', 'error'
    )),
    last_sync_at    TIMESTAMPTZ,
    -- All connection-type-specific fields in JSONB
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- Example connection_config for FDX (US):
    -- {
    --   "provider": "plaid",
    --   "access_token": "access-sandbox-xxx",
    --   "item_id": "plaid-item-xxx",
    --   "institution_id": "ins_109508",
    --   "products": ["transactions", "auth", "balance"],
    --   "webhook_url": "https://api.example.com/webhooks/plaid"
    -- }
    -- Example connection_config for PSD2 (EU):
    -- {
    --   "provider": "truelayer",
    --   "consent_id": "consent-xxx",
    --   "consent_expires_at": "2026-08-12T00:00:00Z",
    --   "aspsp_id": "de-deutsche-bank",
    --   "ais_scope": ["balances", "transactions"],
    --   "pis_scope": ["single-domestic-payment"],
    --   "fapi_profile": "fapi1_advanced"
    -- }
    -- Example connection_config for SWIFT:
    -- {
    --   "service": "fileact",
    --   "bic": "DEUTDEFFXXX",
    --   "message_types": ["MT940", "MT942", "camt.053"],
    --   "sftp_host": "swift.example.com",
    --   "sftp_path": "/incoming/statements"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conn_tenant ON bank_connection(tenant_id);
CREATE INDEX idx_conn_type ON bank_connection(connection_type);
CREATE INDEX idx_conn_config ON bank_connection USING GIN (connection_config);

CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    bank_connection_id UUID REFERENCES bank_connection(id),
    account_name    TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN (
        'operating', 'savings', 'money_market', 'escrow',
        'payroll', 'tax', 'investment', 'loan', 'credit_line'
    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Common account identifiers (relational because they are queried/joined)
    iban            VARCHAR(34),
    -- Provider-specific and jurisdiction-specific account data
    account_identifiers JSONB NOT NULL DEFAULT '{}',
    -- Example account_identifiers (US account):
    -- {
    --   "routing_number": "021000021",
    --   "account_number_masked": "****1234",
    --   "account_number_encrypted": "enc:xxx",
    --   "fdx_account_id": "fdx-acct-xxx"
    -- }
    -- Example account_identifiers (EU account):
    -- {
    --   "bic": "DEUTDEFFXXX",
    --   "account_number_masked": "****5678",
    --   "psd2_resource_id": "/accounts/acct-xxx"
    -- }
    -- Example account_identifiers (AU account):
    -- {
    --   "bsb": "062-000",
    --   "account_number_masked": "****9012",
    --   "cdr_account_id": "cdr-acct-xxx"
    -- }
    -- Provider-specific metadata from bank API
    provider_data   JSONB NOT NULL DEFAULT '{}',
    -- Example provider_data:
    -- {
    --   "plaid_account_id": "plaid-xxx",
    --   "plaid_type": "depository",
    --   "plaid_subtype": "checking",
    --   "institution_name": "Chase"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_acct_tenant ON bank_account(tenant_id);
CREATE INDEX idx_acct_entity ON bank_account(legal_entity_id);
CREATE INDEX idx_acct_iban ON bank_account(iban) WHERE iban IS NOT NULL;
CREATE INDEX idx_acct_identifiers ON bank_account USING GIN (account_identifiers);
```

## Cash Positioning & Balances

```sql
CREATE TABLE bank_account_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    balance_date    DATE NOT NULL,
    balance_type    TEXT NOT NULL CHECK (balance_type IN (
        'closing_booked', 'opening_booked', 'closing_available', 'intraday', 'expected'
    )),
    amount          NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    source          TEXT NOT NULL CHECK (source IN (
        'bank_api', 'camt053', 'camt052', 'mt940', 'mt942', 'csv', 'manual'
    )),
    -- Source-specific details (varies by bank and connection type)
    source_data     JSONB NOT NULL DEFAULT '{}',
    -- Example source_data for camt.053:
    -- {
    --   "statement_id": "CAMT053-20260511-001",
    --   "message_id": "MSGID-xxx",
    --   "creation_date_time": "2026-05-11T23:59:00Z"
    -- }
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bal_tenant ON bank_account_balance(tenant_id);
CREATE INDEX idx_bal_account_date ON bank_account_balance(bank_account_id, balance_date DESC);
CREATE UNIQUE INDEX idx_bal_unique ON bank_account_balance(
    bank_account_id, balance_date, balance_type
);

-- Cash position: aggregated, computed from balances
CREATE TABLE cash_position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    position_date   DATE NOT NULL,
    currency        CHAR(3) NOT NULL,
    total_balance   NUMERIC(18,4) NOT NULL,
    available_balance NUMERIC(18,4),
    account_count   INTEGER NOT NULL DEFAULT 0,
    base_currency_amount NUMERIC(18,4),
    fx_rate_used    NUMERIC(18,8),
    -- Breakdown by account type
    breakdown       JSONB NOT NULL DEFAULT '{}',
    -- Example breakdown:
    -- {
    --   "operating": 1500000.00,
    --   "savings": 800000.00,
    --   "money_market": 250000.00,
    --   "accounts": [
    --     {"account_id": "uuid", "name": "Chase Operating", "balance": 1500000.00}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pos_tenant ON cash_position(tenant_id);
CREATE UNIQUE INDEX idx_pos_unique ON cash_position(
    tenant_id, legal_entity_id, position_date, currency
);
```

## Bank Statements & Transactions

```sql
CREATE TABLE bank_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    statement_date  DATE NOT NULL,
    opening_balance NUMERIC(18,4) NOT NULL,
    closing_balance NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    entry_count     INTEGER NOT NULL DEFAULT 0,
    format          TEXT NOT NULL CHECK (format IN (
        'camt053', 'mt940', 'csv', 'fdx', 'psd2', 'manual'
    )),
    status          TEXT NOT NULL DEFAULT 'received' CHECK (status IN (
        'received', 'processed', 'reconciled', 'error'
    )),
    -- Format-specific metadata and raw content reference
    format_data     JSONB NOT NULL DEFAULT '{}',
    -- Example format_data for camt.053:
    -- {
    --   "message_id": "MSGID-xxx",
    --   "statement_id": "STMTID-xxx",
    --   "sequence_number": "00001",
    --   "creation_date_time": "2026-05-11T23:59:00Z",
    --   "raw_file_ref": "s3://statements/2026/05/11/acct-xxx.xml"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_stmt_tenant ON bank_statement(tenant_id);
CREATE INDEX idx_stmt_account_date ON bank_statement(bank_account_id, statement_date DESC);

CREATE TABLE bank_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_statement_id UUID REFERENCES bank_statement(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    -- Core transaction fields (always relational)
    amount          NUMERIC(18,4) NOT NULL,
    credit_debit    CHAR(1) NOT NULL CHECK (credit_debit IN ('C', 'D')),
    currency        CHAR(3) NOT NULL,
    value_date      DATE NOT NULL,
    booking_date    DATE NOT NULL,
    -- Counterparty (extracted from statement for reconciliation)
    counterparty_name TEXT,
    remittance_info TEXT,
    -- Categorisation (AI-assigned or manual)
    category        TEXT,                  -- e.g. 'receipts', 'payroll', 'rent', 'taxes'
    category_confidence NUMERIC(4,3),      -- ML confidence if AI-categorised
    -- Reconciliation
    reconciliation_status TEXT NOT NULL DEFAULT 'unmatched' CHECK (
        reconciliation_status IN ('unmatched', 'matched', 'manual', 'excluded')
    ),
    matched_payment_id UUID,
    matched_gl_id   UUID,
    -- Format-specific fields (varies by statement format)
    transaction_data JSONB NOT NULL DEFAULT '{}',
    -- Example transaction_data for camt.053 entry:
    -- {
    --   "entry_reference": "20260511001234",
    --   "bank_transaction_code": {"domain": "PMNT", "family": "RCDT", "sub_family": "ESCT"},
    --   "counterparty_iban": "DE89370400440532013000",
    --   "counterparty_bic": "COBADEFFXXX",
    --   "structured_remittance": {"creditor_reference": "RF23INV-001234"},
    --   "supplementary_data": {}
    -- }
    -- Example transaction_data for FDX:
    -- {
    --   "fdx_transaction_id": "fdx-txn-xxx",
    --   "merchant_name": "Office Supplies Inc",
    --   "merchant_category_code": "5943",
    --   "pending": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_txn_tenant ON bank_transaction(tenant_id);
CREATE INDEX idx_txn_account_date ON bank_transaction(bank_account_id, value_date DESC);
CREATE INDEX idx_txn_reconciliation ON bank_transaction(reconciliation_status)
    WHERE reconciliation_status = 'unmatched';
CREATE INDEX idx_txn_category ON bank_transaction(category);
CREATE INDEX idx_txn_data ON bank_transaction USING GIN (transaction_data);
```

## Payments

```sql
CREATE TABLE counterparty (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    counterparty_type TEXT NOT NULL CHECK (counterparty_type IN (
        'vendor', 'customer', 'bank', 'intercompany', 'government', 'other'
    )),
    country_code    CHAR(2),
    lei             VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Payment details vary by jurisdiction and payment type
    bank_accounts   JSONB NOT NULL DEFAULT '[]',
    -- Example bank_accounts:
    -- [
    --   {
    --     "id": "uuid",
    --     "label": "EUR Primary",
    --     "iban": "DE89370400440532013000",
    --     "bic": "COBADEFFXXX",
    --     "currency": "EUR",
    --     "is_default": true,
    --     "is_verified": true,
    --     "verified_at": "2026-01-15T00:00:00Z"
    --   },
    --   {
    --     "id": "uuid",
    --     "label": "USD Account",
    --     "routing_number": "021000021",
    --     "account_number_masked": "****5678",
    --     "currency": "USD",
    --     "is_default": false,
    --     "is_verified": true
    --   }
    -- ]
    contact_info    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cpty_tenant ON counterparty(tenant_id);
CREATE INDEX idx_cpty_bank_accounts ON counterparty USING GIN (bank_accounts);

CREATE TABLE payment_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    debit_account_id UUID NOT NULL REFERENCES bank_account(id),
    counterparty_id UUID REFERENCES counterparty(id),
    -- Core payment fields (always relational — queried, filtered, reported)
    amount          NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    payment_type    TEXT NOT NULL CHECK (payment_type IN (
        'ach', 'wire', 'swift', 'sepa', 'rtp', 'fednow',
        'book_transfer', 'check', 'open_banking'
    )),
    direction       TEXT NOT NULL CHECK (direction IN ('credit', 'debit')),
    value_date      DATE,
    end_to_end_id   TEXT,
    uetr            UUID,
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'pending_approval', 'approved', 'submitted',
        'accepted', 'rejected', 'settled', 'returned', 'cancelled'
    )),
    -- Approval tracking (denormalised for fast reads)
    approval_count  INTEGER NOT NULL DEFAULT 0,
    required_approvals INTEGER NOT NULL DEFAULT 1,
    -- Destination account (extracted from counterparty for this specific payment)
    credit_account  JSONB NOT NULL DEFAULT '{}',
    -- Example credit_account:
    -- {
    --   "iban": "DE89370400440532013000",
    --   "bic": "COBADEFFXXX",
    --   "account_holder": "Acme Supplies GmbH"
    -- }
    -- Payment-type-specific fields
    payment_details JSONB NOT NULL DEFAULT '{}',
    -- Example payment_details for SEPA:
    -- {
    --   "remittance_info": "Invoice #2026-001234",
    --   "purpose_code": "SUPP",
    --   "charge_bearer": "SLEV",
    --   "regulatory_reporting": {"code": "150", "amount": 150000}
    -- }
    -- Example payment_details for SWIFT wire:
    -- {
    --   "remittance_info": "PO-2026-5678",
    --   "purpose_code": "TRADE",
    --   "charge_bearer": "SHA",
    --   "intermediary_bank_bic": "CHASUS33XXX",
    --   "instruction_for_bank": "URGENT"
    -- }
    -- Status history (denormalised from status updates)
    status_history  JSONB NOT NULL DEFAULT '[]',
    -- Example status_history:
    -- [
    --   {"status": "draft", "at": "2026-05-12T10:00:00Z", "by": "user-uuid"},
    --   {"status": "pending_approval", "at": "2026-05-12T10:01:00Z", "by": "user-uuid"},
    --   {"status": "approved", "at": "2026-05-12T11:30:00Z", "by": "approver-uuid",
    --    "comment": "Approved per Q2 budget"},
    --   {"status": "submitted", "at": "2026-05-12T12:00:00Z", "by": "system",
    --    "bank_reference": "BANKREF-001"},
    --   {"status": "settled", "at": "2026-05-13T08:00:00Z", "by": "bank_api",
    --    "settlement_date": "2026-05-13"}
    -- ]
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pmt_tenant ON payment_order(tenant_id);
CREATE INDEX idx_pmt_entity ON payment_order(legal_entity_id);
CREATE INDEX idx_pmt_status ON payment_order(status);
CREATE INDEX idx_pmt_date ON payment_order(value_date);
CREATE INDEX idx_pmt_uetr ON payment_order(uetr) WHERE uetr IS NOT NULL;
CREATE INDEX idx_pmt_details ON payment_order USING GIN (payment_details);
```

## FX Rates, Exposure & Hedging

```sql
CREATE TABLE fx_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    base_currency   CHAR(3) NOT NULL,
    quote_currency  CHAR(3) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    rate_type       TEXT NOT NULL CHECK (rate_type IN (
        'spot', 'forward', 'ecb', 'internal', 'budget'
    )),
    rate_date       DATE NOT NULL,
    source          TEXT NOT NULL,
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fxr_pair_date ON fx_rate(base_currency, quote_currency, rate_date DESC);
CREATE UNIQUE INDEX idx_fxr_unique ON fx_rate(
    base_currency, quote_currency, rate_date, rate_type, source
);

CREATE TABLE fx_exposure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    exposure_date   DATE NOT NULL,
    currency        CHAR(3) NOT NULL,
    exposure_type   TEXT NOT NULL CHECK (exposure_type IN (
        'balance_sheet', 'forecast', 'committed', 'contingent'
    )),
    gross_amount    NUMERIC(18,4) NOT NULL,
    hedged_amount   NUMERIC(18,4) NOT NULL DEFAULT 0,
    net_amount      NUMERIC(18,4) GENERATED ALWAYS AS (gross_amount - hedged_amount) STORED,
    hedge_ratio     NUMERIC(5,4) GENERATED ALWAYS AS (
        CASE WHEN gross_amount = 0 THEN 0 ELSE hedged_amount / gross_amount END
    ) STORED,
    -- Exposure breakdown detail
    exposure_detail JSONB NOT NULL DEFAULT '{}',
    -- Example exposure_detail:
    -- {
    --   "sources": [
    --     {"type": "ar_receivable", "amount": 500000, "due_date": "2026-06-30"},
    --     {"type": "ap_payable", "amount": -200000, "due_date": "2026-06-15"},
    --     {"type": "intercompany", "amount": 300000, "entity": "ACME UK Ltd"}
    --   ],
    --   "hedges": [
    --     {"trade_id": "uuid", "type": "forward", "amount": 400000, "rate": 1.0850}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exp_tenant ON fx_exposure(tenant_id);
CREATE UNIQUE INDEX idx_exp_unique ON fx_exposure(
    tenant_id, legal_entity_id, exposure_date, currency, exposure_type
);

CREATE TABLE fx_trade (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    counterparty_id UUID REFERENCES counterparty(id),
    trade_type      TEXT NOT NULL CHECK (trade_type IN (
        'spot', 'forward', 'ndf', 'option', 'swap'
    )),
    buy_currency    CHAR(3) NOT NULL,
    buy_amount      NUMERIC(18,4) NOT NULL,
    sell_currency   CHAR(3) NOT NULL,
    sell_amount     NUMERIC(18,4) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    trade_date      DATE NOT NULL,
    settlement_date DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'open' CHECK (status IN (
        'open', 'partially_settled', 'settled', 'cancelled', 'expired'
    )),
    mark_to_market  NUMERIC(18,4),
    mtm_date        DATE,
    -- Trade-type-specific fields and ISDA/FpML data
    instrument_data JSONB NOT NULL DEFAULT '{}',
    -- Example instrument_data for forward:
    -- {
    --   "isda_agreement_ref": "ISDA-2024-JPM-001",
    --   "confirmation_ref": "CONF-20260512-001",
    --   "forward_points": 0.0025,
    --   "spot_rate_at_inception": 1.0844,
    --   "hedge_designation_id": "uuid"
    -- }
    -- Example instrument_data for option:
    -- {
    --   "option_type": "call",
    --   "strike_rate": 1.1000,
    --   "premium_amount": 15000.00,
    --   "premium_currency": "USD",
    --   "expiry_date": "2026-08-10",
    --   "exercise_style": "european",
    --   "delta": 0.45,
    --   "implied_vol": 0.082
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fxt_tenant ON fx_trade(tenant_id);
CREATE INDEX idx_fxt_entity ON fx_trade(legal_entity_id);
CREATE INDEX idx_fxt_settlement ON fx_trade(settlement_date);
CREATE INDEX idx_fxt_status ON fx_trade(status) WHERE status = 'open';
CREATE INDEX idx_fxt_instrument ON fx_trade USING GIN (instrument_data);
```

## Hedge Accounting

```sql
CREATE TABLE hedge_designation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    designation_ref TEXT NOT NULL,
    hedge_type      TEXT NOT NULL CHECK (hedge_type IN (
        'fair_value', 'cash_flow', 'net_investment'
    )),
    accounting_standard TEXT NOT NULL CHECK (accounting_standard IN ('asc815', 'ifrs9')),
    hedged_risk     TEXT NOT NULL,
    inception_date  DATE NOT NULL,
    expiry_date     DATE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'dedesignated', 'expired', 'matured'
    )),
    -- Hedge documentation (varies by standard and jurisdiction)
    documentation   JSONB NOT NULL DEFAULT '{}',
    -- Example documentation:
    -- {
    --   "hedged_item": {
    --     "description": "Forecasted EUR revenue Q3 2026",
    --     "amount": 1000000,
    --     "currency": "EUR",
    --     "expected_dates": ["2026-07-01", "2026-09-30"]
    --   },
    --   "hedging_instrument": {
    --     "trade_id": "uuid",
    --     "type": "forward",
    --     "notional": 1000000
    --   },
    --   "risk_management_objective": "Reduce variability of USD cash flows from EUR revenue",
    --   "effectiveness_method": "critical_terms_match",
    --   "sources_of_ineffectiveness": ["timing_mismatch", "credit_risk"]
    -- }
    -- Effectiveness test history (denormalised for audit access)
    effectiveness_tests JSONB NOT NULL DEFAULT '[]',
    -- Example effectiveness_tests:
    -- [
    --   {
    --     "test_date": "2026-06-30",
    --     "test_type": "retrospective",
    --     "method": "dollar_offset",
    --     "ratio": 0.96,
    --     "is_effective": true,
    --     "hedged_item_fv_change": -45000,
    --     "instrument_fv_change": 43200,
    --     "ineffectiveness": 1800
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hedge_tenant ON hedge_designation(tenant_id);
CREATE INDEX idx_hedge_status ON hedge_designation(status);
```

## Cash Flow Forecasting

```sql
CREATE TABLE forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID REFERENCES legal_entity(id),
    name            TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    forecast_date   DATE NOT NULL,
    horizon_start   DATE NOT NULL,
    horizon_end     DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'published', 'superseded', 'archived'
    )),
    -- Model configuration and results
    model_config    JSONB NOT NULL DEFAULT '{}',
    -- Example model_config:
    -- {
    --   "model_type": "random_forest",
    --   "lookback_days": 365,
    --   "features": ["day_of_week", "month", "payroll_flag", "historical_amount"],
    --   "hyperparameters": {"n_estimators": 100, "max_depth": 10},
    --   "accuracy": {"mape": 12.3, "rmse": 45000, "r2": 0.87}
    -- }
    -- Line items stored as JSONB array (avoids separate table for fast API reads)
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- Example line_items:
    -- [
    --   {
    --     "period_start": "2026-05-12",
    --     "period_end": "2026-05-18",
    --     "category": "receipts",
    --     "projected": 450000,
    --     "actual": 462000,
    --     "variance": 12000,
    --     "confidence_lower": 380000,
    --     "confidence_upper": 520000
    --   },
    --   {
    --     "period_start": "2026-05-12",
    --     "period_end": "2026-05-18",
    --     "category": "payroll",
    --     "projected": -280000,
    --     "actual": -280000,
    --     "variance": 0,
    --     "confidence_lower": -280000,
    --     "confidence_upper": -280000
    --   }
    -- ]
    -- Scenarios stored alongside the forecast
    scenarios       JSONB NOT NULL DEFAULT '[]',
    -- Example scenarios:
    -- [
    --   {"name": "Base", "type": "base", "adjustments": {}},
    --   {"name": "FX Shock -10%", "type": "stress",
    --    "adjustments": {"fx_shock_pct": -10, "affected_currencies": ["EUR", "GBP"]}}
    -- ]
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fcst_tenant ON forecast(tenant_id);
CREATE INDEX idx_fcst_date ON forecast(forecast_date DESC);
CREATE INDEX idx_fcst_status ON forecast(status);
```

## Debt & Covenants

```sql
CREATE TABLE debt_instrument (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    counterparty_id UUID REFERENCES counterparty(id),
    instrument_type TEXT NOT NULL CHECK (instrument_type IN (
        'term_loan', 'revolving_credit', 'bond', 'commercial_paper',
        'overdraft', 'letter_of_credit'
    )),
    name            TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    principal_amount NUMERIC(18,4) NOT NULL,
    outstanding_amount NUMERIC(18,4) NOT NULL,
    interest_rate_type TEXT NOT NULL CHECK (interest_rate_type IN ('fixed', 'floating')),
    interest_rate   NUMERIC(8,4),
    start_date      DATE NOT NULL,
    maturity_date   DATE NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Instrument-specific terms and covenants
    terms           JSONB NOT NULL DEFAULT '{}',
    -- Example terms:
    -- {
    --   "reference_rate": "SOFR",
    --   "spread_bps": 250,
    --   "payment_frequency": "quarterly",
    --   "day_count_convention": "ACT/360",
    --   "prepayment_penalty_pct": 2.0,
    --   "covenants": [
    --     {
    --       "id": "uuid",
    --       "name": "Leverage Ratio",
    --       "type": "leverage_ratio",
    --       "test_frequency": "quarterly",
    --       "threshold_operator": "lte",
    --       "threshold_value": 3.5,
    --       "cure_period_days": 30
    --     },
    --     {
    --       "id": "uuid",
    --       "name": "Interest Coverage",
    --       "type": "interest_coverage",
    --       "test_frequency": "quarterly",
    --       "threshold_operator": "gte",
    --       "threshold_value": 3.0
    --     }
    --   ]
    -- }
    -- Covenant test results history
    covenant_tests  JSONB NOT NULL DEFAULT '[]',
    -- Example covenant_tests:
    -- [
    --   {
    --     "covenant_id": "uuid",
    --     "test_date": "2026-03-31",
    --     "actual_value": 2.8,
    --     "threshold_value": 3.5,
    --     "is_compliant": true,
    --     "headroom_pct": 20.0
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_debt_tenant ON debt_instrument(tenant_id);
CREATE INDEX idx_debt_entity ON debt_instrument(legal_entity_id);
CREATE INDEX idx_debt_terms ON debt_instrument USING GIN (terms);
```

## Users, RBAC & Audit

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Role assignments and preferences in JSONB
    roles           JSONB NOT NULL DEFAULT '[]',
    -- Example roles:
    -- [
    --   {"role": "treasurer", "scope": "global"},
    --   {"role": "approver", "scope": "entity", "entity_id": "uuid", "limit": 500000}
    -- ]
    preferences     JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_user_email_tenant ON app_user(tenant_id, email);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    changes         JSONB NOT NULL DEFAULT '{}',
    -- Example changes:
    -- {
    --   "before": {"status": "draft", "amount": 100000},
    --   "after": {"status": "approved", "amount": 100000},
    --   "fields_changed": ["status"]
    -- }
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example context:
    -- {"ip": "192.168.1.1", "user_agent": "...", "session_id": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);

-- Alert configuration and events
CREATE TABLE alert_config (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    alert_type      TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- All alert configuration in JSONB (highly variable by alert type)
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config for low_balance:
    -- {
    --   "threshold": 100000,
    --   "currency": "USD",
    --   "accounts": ["uuid1", "uuid2"],
    --   "recipients": [{"type": "user", "id": "uuid"}, {"type": "email", "addr": "cfo@example.com"}],
    --   "channels": ["email", "slack"],
    --   "cooldown_minutes": 60
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_cfg_tenant ON alert_config(tenant_id);

-- Row-Level Security
ALTER TABLE legal_entity ENABLE ROW LEVEL SECURITY;
ALTER TABLE bank_account ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_order ENABLE ROW LEVEL SECURITY;
ALTER TABLE bank_statement ENABLE ROW LEVEL SECURITY;
ALTER TABLE cash_position ENABLE ROW LEVEL SECURITY;
ALTER TABLE fx_exposure ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON legal_entity
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
CREATE POLICY tenant_isolation ON bank_account
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
CREATE POLICY tenant_isolation ON payment_order
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

## Example JSONB Queries

```sql
-- Find all bank connections using Plaid as the FDX provider
SELECT id, bank_name, connection_config->>'provider' AS provider
FROM bank_connection
WHERE tenant_id = 'tenant-uuid'
  AND connection_type = 'fdx'
  AND connection_config @> '{"provider": "plaid"}';

-- Find all counterparties with a verified EUR bank account
SELECT id, name
FROM counterparty
WHERE tenant_id = 'tenant-uuid'
  AND bank_accounts @> '[{"currency": "EUR", "is_verified": true}]';

-- Get forecast line items for a specific category
SELECT
    f.name,
    f.forecast_date,
    line->>'category' AS category,
    (line->>'projected')::NUMERIC AS projected,
    (line->>'actual')::NUMERIC AS actual
FROM forecast f,
     jsonb_array_elements(f.line_items) AS line
WHERE f.tenant_id = 'tenant-uuid'
  AND f.status = 'published'
  AND line->>'category' = 'payroll';

-- Check covenant compliance from debt instrument terms
SELECT
    d.name AS instrument,
    cov->>'name' AS covenant,
    test->>'test_date' AS test_date,
    (test->>'actual_value')::NUMERIC AS actual,
    (test->>'threshold_value')::NUMERIC AS threshold,
    (test->>'is_compliant')::BOOLEAN AS compliant
FROM debt_instrument d,
     jsonb_array_elements(d.terms->'covenants') AS cov,
     jsonb_array_elements(d.covenant_tests) AS test
WHERE d.tenant_id = 'tenant-uuid'
  AND test->>'covenant_id' = cov->>'id';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Entities | 2 | tenant, legal_entity |
| Bank Accounts & Connectivity | 2 | bank_connection, bank_account |
| Cash Positioning | 2 | bank_account_balance, cash_position |
| Bank Statements | 2 | bank_statement, bank_transaction |
| Payments | 2 | counterparty, payment_order |
| FX & Hedging | 4 | fx_rate, fx_exposure, fx_trade, hedge_designation |
| Forecasting | 1 | forecast (line items and scenarios in JSONB) |
| Debt & Covenants | 1 | debt_instrument (covenants and tests in JSONB) |
| Users & Audit | 3 | app_user, audit_log, alert_config |
| **Total** | **19** | ~50% fewer tables than normalised model |

---

## Key Design Decisions

1. **JSONB for variable-by-jurisdiction fields** — `bank_connection.connection_config`, `bank_account.account_identifiers`, and `legal_entity.regulatory_data` use JSONB because their contents differ fundamentally between FDX (US), PSD2 (EU), CDR (AU), and SWIFT connections. A relational approach would require either a wide table with 50+ nullable columns or a separate table per connection type.

2. **Counterparty bank accounts as JSONB array** — rather than a separate `counterparty_bank_account` table, bank accounts are embedded in the counterparty record as a JSONB array. This eliminates a JOIN for the most common query pattern (show counterparty with their accounts) and supports containment queries (`@>`) for finding counterparties by currency or verification status.

3. **Payment status history embedded in JSONB** — the `payment_order.status_history` JSONB array replaces a separate `payment_status_history` table. For a typical payment with 4-6 status transitions, this keeps the full lifecycle in a single row read. The trade-off is that querying "all payments that were rejected today" requires JSON path queries rather than a simple WHERE clause on a relational table.

4. **Forecast line items as JSONB** — a 13-week forecast with 5 categories produces ~65 line items. Storing these as a JSONB array on the forecast row eliminates a separate table and 65 rows per forecast, dramatically simplifying forecast CRUD and API serialisation. For forecasts with thousands of line items (12-month daily), a separate table may be needed — but for the 13-week MVP scope, JSONB is efficient.

5. **Debt covenants embedded in instrument terms** — covenants are tightly coupled to their debt instruments (typically 2-5 per instrument). Embedding them in a `terms` JSONB column keeps the debt instrument as a self-contained document while still supporting GIN-indexed queries.

6. **AI transaction categorisation with confidence** — `bank_transaction.category` and `category_confidence` support ML-driven cash flow categorisation. The category is a relational column (frequently grouped/filtered), while the confidence score enables human review of low-confidence categorisations.

7. **GIN indexes on all JSONB columns** — every JSONB column that may be queried has a GIN index, ensuring containment queries (`@>`) and existence checks (`?`) perform well. GIN indexes are not as fast as B-tree indexes on scalar columns, but they provide adequate performance for the JSONB query patterns used here.

8. **Relational columns for query-critical fields** — despite the JSONB flexibility, fields that are frequently used in WHERE clauses, JOINs, GROUP BY, or ORDER BY are always relational columns: `tenant_id`, `legal_entity_id`, `currency`, `status`, `value_date`, `amount`. This ensures query plans use B-tree indexes for these critical paths.

9. **Hedge effectiveness tests denormalised into JSONB** — rather than a separate `hedge_effectiveness_test` table, test results are stored as a JSONB array on `hedge_designation.effectiveness_tests`. This keeps the full hedge documentation as a single auditable document, which aligns with how auditors review hedge accounting (they want the complete designation package, not scattered rows across tables).

10. **19 tables vs. 37** — this model achieves roughly 50% fewer tables than the normalised relational model while preserving all the same domain concepts. The reduced table count translates to fewer migrations, simpler ORM configuration, and faster onboarding for new developers — at the cost of requiring discipline around JSONB schema documentation and validation.
