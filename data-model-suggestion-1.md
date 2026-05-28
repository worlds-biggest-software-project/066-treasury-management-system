# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Treasury Management System · Created: 2026-05-12

## Philosophy

The entity-centric normalized relational model follows classical database design principles: every concept gets its own table, relationships are expressed through foreign keys and junction tables, and data integrity is enforced at the schema level through constraints, unique indexes, and referential integrity rules. This is the approach used by the vast majority of production financial systems, including the internal architectures described by Modern Treasury, Square's Books system, and enterprise ERP general ledger modules.

For a treasury management system, this means separate tables for organisations, legal entities, bank accounts, bank statements, cash positions, payment orders, FX exposures, hedging instruments, forecasts, and every supporting reference data concept. The schema is large (60+ tables) but each table has a clear, single purpose. Queries are straightforward JOINs. The double-entry accounting pattern is implemented explicitly with journal entries and postings that must balance to zero.

This approach is best suited for teams with strong relational database experience who value data integrity guarantees, need complex cross-entity reporting (e.g., "show me all FX exposures across all entities in the APAC region with hedging coverage below 70%"), and operate in regulatory environments where schema-level constraints provide audit confidence.

**Best for:** Production deployments where data integrity, complex cross-entity queries, and regulatory audit confidence are the top priorities.

**Trade-offs:**
- Pro: Maximum data integrity — constraints prevent invalid states at the database level
- Pro: Straightforward querying with standard SQL JOINs — no special query patterns needed
- Pro: Well-understood by most developers and DBAs; extensive tooling ecosystem
- Pro: Schema serves as living documentation of the domain model
- Con: High table count (60+) increases migration complexity and onboarding time
- Con: Adding jurisdiction-specific fields requires schema migrations rather than configuration
- Con: Many-to-many relationships produce numerous junction tables
- Con: Temporal queries ("what was the cash position on March 15?") require explicit history tables or SCD patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 20022 CAMT (camt.052/053/054) | `bank_statement` and `bank_statement_entry` tables mirror CAMT message structure; fields map to BkToCstmrStmt elements |
| ISO 20022 PAIN (pain.001/002) | `payment_order` table fields align with CstmrCdtTrfInitn structure; `payment_status` tracks pain.002 status codes |
| ISO 4217 | `currency` reference table uses ISO 4217 alphabetic and numeric codes |
| ISO 3166 | `country` and `jurisdiction` tables use ISO 3166-1 alpha-2 codes |
| ISO 17442 (LEI) | `legal_entity.lei` column stores Legal Entity Identifiers |
| FDX API v6.x | `bank_connection` table models FDX account linkage; `bank_account` fields align with FDX DepositAccount schema |
| Berlin Group NextGenPSD2 | `bank_connection` supports PSD2 AIS/PIS consent model with consent_id and expiry tracking |
| SWIFT gpi | `payment_order.uetr` stores Unique End-to-End Transaction Reference for cross-border payment tracking |
| FX Global Code | `fx_trade` table captures the 55 principles' data requirements: counterparty, execution venue, timestamp, rate |
| ASC 815 / IFRS 9 | `hedge_designation` and `hedge_effectiveness_test` tables implement hedge accounting documentation requirements |
| ISDA | `fx_trade` and `derivative_instrument` tables store ISDA Master Agreement references and confirmation data |

---

## Organisation & Entity Management

```sql
-- Tenant: the top-level organisation using the platform
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Legal entity: a subsidiary, branch, or division within the tenant
CREATE TABLE legal_entity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES legal_entity(id),  -- for entity hierarchy
    name            TEXT NOT NULL,
    short_name      TEXT,
    lei             VARCHAR(20),           -- ISO 17442 Legal Entity Identifier
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL,      -- ISO 3166-1 alpha-2
    jurisdiction    TEXT,
    functional_currency CHAR(3) NOT NULL,  -- ISO 4217
    entity_type     TEXT NOT NULL CHECK (entity_type IN (
        'holding', 'operating', 'spv', 'branch', 'joint_venture'
    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_legal_entity_tenant ON legal_entity(tenant_id);
CREATE INDEX idx_legal_entity_parent ON legal_entity(parent_id);
CREATE UNIQUE INDEX idx_legal_entity_lei ON legal_entity(lei) WHERE lei IS NOT NULL;

-- Reference data: currencies
CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,   -- ISO 4217 alphabetic code
    numeric_code    CHAR(3),               -- ISO 4217 numeric code
    name            TEXT NOT NULL,
    minor_units     SMALLINT NOT NULL DEFAULT 2,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- Reference data: countries
CREATE TABLE country (
    code            CHAR(2) PRIMARY KEY,   -- ISO 3166-1 alpha-2
    alpha3          CHAR(3) NOT NULL,      -- ISO 3166-1 alpha-3
    name            TEXT NOT NULL,
    region          TEXT                   -- e.g. 'EMEA', 'APAC', 'Americas'
);
```

## Bank Accounts & Connectivity

```sql
-- Bank: a financial institution the tenant has relationships with
CREATE TABLE bank (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    swift_bic       VARCHAR(11),           -- SWIFT BIC code
    country_code    CHAR(2) REFERENCES country(code),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bank_tenant ON bank(tenant_id);

-- Bank connection: API or SWIFT connectivity configuration
CREATE TABLE bank_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_id         UUID NOT NULL REFERENCES bank(id),
    connection_type TEXT NOT NULL CHECK (connection_type IN (
        'fdx', 'psd2', 'cdr', 'swift', 'sftp', 'manual'
    )),
    -- FDX / Open Banking fields
    provider        TEXT,                  -- e.g. 'plaid', 'truelayer', 'direct'
    access_token    TEXT,                  -- encrypted at rest
    consent_id      TEXT,                  -- PSD2 consent identifier
    consent_expires_at TIMESTAMPTZ,
    -- SWIFT fields
    swift_service   TEXT,                  -- e.g. 'fin', 'fileact'
    -- Status
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'active', 'expired', 'revoked', 'error'
    )),
    last_sync_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bank_connection_tenant ON bank_connection(tenant_id);
CREATE INDEX idx_bank_connection_bank ON bank_connection(bank_id);

-- Bank account: an account held at a bank by a legal entity
CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    bank_id         UUID NOT NULL REFERENCES bank(id),
    bank_connection_id UUID REFERENCES bank_connection(id),
    account_name    TEXT NOT NULL,
    account_number  TEXT,                  -- masked in application layer
    iban            VARCHAR(34),
    sort_code       VARCHAR(10),
    routing_number  VARCHAR(9),
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    account_type    TEXT NOT NULL CHECK (account_type IN (
        'operating', 'savings', 'money_market', 'escrow',
        'payroll', 'tax', 'investment', 'loan', 'credit_line'
    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    opened_at       DATE,
    closed_at       DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bank_account_tenant ON bank_account(tenant_id);
CREATE INDEX idx_bank_account_entity ON bank_account(legal_entity_id);
CREATE INDEX idx_bank_account_bank ON bank_account(bank_id);
CREATE UNIQUE INDEX idx_bank_account_iban ON bank_account(iban) WHERE iban IS NOT NULL;
```

## Cash Positioning

```sql
-- Bank account balance: point-in-time balance snapshot
-- One row per account per day (end-of-day from camt.053) or intraday (camt.052)
CREATE TABLE bank_account_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    balance_date    DATE NOT NULL,
    balance_type    TEXT NOT NULL CHECK (balance_type IN (
        'closing_booked',       -- camt.053 CLBD
        'opening_booked',       -- camt.053 OPBD
        'closing_available',    -- camt.053 CLAV
        'intraday',             -- camt.052
        'expected'              -- forward value date
    )),
    amount          NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    source          TEXT NOT NULL CHECK (source IN (
        'bank_api', 'camt053', 'camt052', 'mt940', 'mt942', 'csv', 'manual'
    )),
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_balance_tenant ON bank_account_balance(tenant_id);
CREATE INDEX idx_balance_account_date ON bank_account_balance(bank_account_id, balance_date DESC);
CREATE UNIQUE INDEX idx_balance_unique ON bank_account_balance(
    bank_account_id, balance_date, balance_type
);

-- Cash position: aggregated position per entity per currency per date
CREATE TABLE cash_position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    position_date   DATE NOT NULL,
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    total_balance   NUMERIC(18,4) NOT NULL,
    available_balance NUMERIC(18,4),
    account_count   INTEGER NOT NULL DEFAULT 0,
    base_currency_amount NUMERIC(18,4),   -- converted to tenant base currency
    fx_rate_used    NUMERIC(18,8),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_position_tenant ON cash_position(tenant_id);
CREATE UNIQUE INDEX idx_position_unique ON cash_position(
    tenant_id, legal_entity_id, position_date, currency
);
```

## Bank Statements & Transactions

```sql
-- Bank statement: a statement file/message received from a bank
-- Maps to camt.053 BkToCstmrStmt or MT940
CREATE TABLE bank_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    statement_id    TEXT,                  -- bank-assigned statement identifier
    sequence_number TEXT,                  -- bank sequence number
    statement_date  DATE NOT NULL,
    opening_balance NUMERIC(18,4) NOT NULL,
    closing_balance NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    entry_count     INTEGER NOT NULL DEFAULT 0,
    format          TEXT NOT NULL CHECK (format IN (
        'camt053', 'mt940', 'csv', 'fdx', 'psd2', 'manual'
    )),
    raw_content     TEXT,                  -- original XML/MT/CSV content
    status          TEXT NOT NULL DEFAULT 'received' CHECK (status IN (
        'received', 'processed', 'reconciled', 'error'
    )),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_statement_tenant ON bank_statement(tenant_id);
CREATE INDEX idx_statement_account ON bank_statement(bank_account_id, statement_date DESC);

-- Bank statement entry: individual transaction within a statement
-- Maps to camt.053 Ntry element or MT940 :61: field
CREATE TABLE bank_statement_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_statement_id UUID NOT NULL REFERENCES bank_statement(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    entry_reference TEXT,                  -- bank transaction reference
    amount          NUMERIC(18,4) NOT NULL,
    credit_debit    CHAR(1) NOT NULL CHECK (credit_debit IN ('C', 'D')),
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    value_date      DATE NOT NULL,
    booking_date    DATE NOT NULL,
    counterparty_name TEXT,
    counterparty_account TEXT,
    remittance_info TEXT,                  -- unstructured remittance
    bank_transaction_code TEXT,            -- ISO 20022 BankTransactionCode
    -- Reconciliation fields
    reconciliation_status TEXT NOT NULL DEFAULT 'unmatched' CHECK (
        reconciliation_status IN ('unmatched', 'matched', 'manual', 'excluded')
    ),
    matched_payment_id UUID,              -- FK to payment_order if matched
    matched_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_entry_tenant ON bank_statement_entry(tenant_id);
CREATE INDEX idx_entry_statement ON bank_statement_entry(bank_statement_id);
CREATE INDEX idx_entry_account_date ON bank_statement_entry(bank_account_id, value_date);
CREATE INDEX idx_entry_reconciliation ON bank_statement_entry(reconciliation_status)
    WHERE reconciliation_status = 'unmatched';
```

## Payments

```sql
-- Counterparty: external parties the tenant makes/receives payments from
CREATE TABLE counterparty (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    short_name      TEXT,
    counterparty_type TEXT NOT NULL CHECK (counterparty_type IN (
        'vendor', 'customer', 'bank', 'intercompany', 'government', 'other'
    )),
    country_code    CHAR(2) REFERENCES country(code),
    lei             VARCHAR(20),           -- ISO 17442 if applicable
    tax_id          TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_counterparty_tenant ON counterparty(tenant_id);

-- Counterparty bank account: bank details for a counterparty
CREATE TABLE counterparty_bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    counterparty_id UUID NOT NULL REFERENCES counterparty(id),
    account_name    TEXT,
    iban            VARCHAR(34),
    account_number  TEXT,
    routing_number  VARCHAR(9),
    sort_code       VARCHAR(10),
    swift_bic       VARCHAR(11),
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    country_code    CHAR(2) REFERENCES country(code),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_verified     BOOLEAN NOT NULL DEFAULT false,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cpty_bank_tenant ON counterparty_bank_account(tenant_id);
CREATE INDEX idx_cpty_bank_cpty ON counterparty_bank_account(counterparty_id);

-- Payment order: a payment instruction
-- Aligns with ISO 20022 pain.001 CstmrCdtTrfInitn
CREATE TABLE payment_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    -- Source account
    debit_account_id UUID NOT NULL REFERENCES bank_account(id),
    -- Destination
    counterparty_id UUID REFERENCES counterparty(id),
    credit_account_id UUID REFERENCES counterparty_bank_account(id),
    -- Payment details
    amount          NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    payment_type    TEXT NOT NULL CHECK (payment_type IN (
        'ach', 'wire', 'swift', 'sepa', 'rtp', 'fednow',
        'book_transfer', 'check', 'open_banking'
    )),
    direction       TEXT NOT NULL CHECK (direction IN ('credit', 'debit')),
    priority        TEXT NOT NULL DEFAULT 'normal' CHECK (priority IN (
        'normal', 'urgent', 'express'
    )),
    value_date      DATE,
    -- ISO 20022 / SWIFT references
    end_to_end_id   TEXT,                  -- pain.001 EndToEndId
    uetr            UUID,                  -- SWIFT gpi tracking reference
    instruction_id  TEXT,                  -- pain.001 InstrId
    -- Remittance
    remittance_info TEXT,
    purpose_code    VARCHAR(10),           -- ISO 20022 purpose code
    -- Status lifecycle
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'pending_approval', 'approved', 'submitted',
        'accepted', 'rejected', 'settled', 'returned', 'cancelled'
    )),
    submitted_at    TIMESTAMPTZ,
    settled_at      TIMESTAMPTZ,
    -- Creator
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payment_tenant ON payment_order(tenant_id);
CREATE INDEX idx_payment_entity ON payment_order(legal_entity_id);
CREATE INDEX idx_payment_status ON payment_order(status);
CREATE INDEX idx_payment_value_date ON payment_order(value_date);
CREATE INDEX idx_payment_uetr ON payment_order(uetr) WHERE uetr IS NOT NULL;

-- Payment approval: approval workflow for payment orders
CREATE TABLE payment_approval (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    payment_order_id UUID NOT NULL REFERENCES payment_order(id),
    approver_id     UUID NOT NULL,
    approval_level  SMALLINT NOT NULL DEFAULT 1,
    decision        TEXT NOT NULL CHECK (decision IN ('approved', 'rejected')),
    comments        TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_approval_payment ON payment_approval(payment_order_id);

-- Payment status history: tracks pain.002 status updates
CREATE TABLE payment_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    payment_order_id UUID NOT NULL REFERENCES payment_order(id),
    status          TEXT NOT NULL,
    reason_code     TEXT,                  -- ISO 20022 reason code
    reason_text     TEXT,
    source          TEXT NOT NULL,         -- 'bank_api', 'pain002', 'swift_gpi', 'manual'
    raw_message     TEXT,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payment_status_payment ON payment_status_history(payment_order_id);
```

## FX Rates & Exposure

```sql
-- FX rate: exchange rate snapshots
CREATE TABLE fx_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    base_currency   CHAR(3) NOT NULL REFERENCES currency(code),
    quote_currency  CHAR(3) NOT NULL REFERENCES currency(code),
    rate            NUMERIC(18,8) NOT NULL,
    rate_type       TEXT NOT NULL CHECK (rate_type IN (
        'spot', 'forward', 'ecb', 'internal', 'budget'
    )),
    rate_date       DATE NOT NULL,
    source          TEXT NOT NULL,         -- 'ecb', 'bloomberg', 'manual'
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fx_rate_pair_date ON fx_rate(base_currency, quote_currency, rate_date DESC);
CREATE UNIQUE INDEX idx_fx_rate_unique ON fx_rate(
    base_currency, quote_currency, rate_date, rate_type, source
);

-- FX exposure: net currency exposure per entity
CREATE TABLE fx_exposure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    exposure_date   DATE NOT NULL,
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    exposure_type   TEXT NOT NULL CHECK (exposure_type IN (
        'balance_sheet', 'forecast', 'committed', 'contingent'
    )),
    gross_amount    NUMERIC(18,4) NOT NULL,  -- in exposure currency
    hedged_amount   NUMERIC(18,4) NOT NULL DEFAULT 0,
    net_amount      NUMERIC(18,4) GENERATED ALWAYS AS (gross_amount - hedged_amount) STORED,
    hedge_ratio     NUMERIC(5,4) GENERATED ALWAYS AS (
        CASE WHEN gross_amount = 0 THEN 0
             ELSE hedged_amount / gross_amount
        END
    ) STORED,
    base_currency_equivalent NUMERIC(18,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exposure_tenant ON fx_exposure(tenant_id);
CREATE UNIQUE INDEX idx_exposure_unique ON fx_exposure(
    tenant_id, legal_entity_id, exposure_date, currency, exposure_type
);

-- FX trade: hedging instrument (forward, option, swap)
CREATE TABLE fx_trade (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    counterparty_id UUID REFERENCES counterparty(id),
    trade_type      TEXT NOT NULL CHECK (trade_type IN (
        'spot', 'forward', 'ndf', 'option', 'swap'
    )),
    buy_currency    CHAR(3) NOT NULL REFERENCES currency(code),
    buy_amount      NUMERIC(18,4) NOT NULL,
    sell_currency   CHAR(3) NOT NULL REFERENCES currency(code),
    sell_amount     NUMERIC(18,4) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    trade_date      DATE NOT NULL,
    settlement_date DATE NOT NULL,
    maturity_date   DATE,
    -- ISDA reference
    isda_agreement_ref TEXT,
    confirmation_ref   TEXT,
    -- Status
    status          TEXT NOT NULL DEFAULT 'open' CHECK (status IN (
        'open', 'partially_settled', 'settled', 'cancelled', 'expired'
    )),
    -- Hedge accounting linkage
    hedge_designation_id UUID,            -- FK to hedge_designation
    mark_to_market  NUMERIC(18,4),
    mtm_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fx_trade_tenant ON fx_trade(tenant_id);
CREATE INDEX idx_fx_trade_entity ON fx_trade(legal_entity_id);
CREATE INDEX idx_fx_trade_settlement ON fx_trade(settlement_date);
CREATE INDEX idx_fx_trade_status ON fx_trade(status) WHERE status = 'open';
```

## Hedge Accounting (ASC 815 / IFRS 9)

```sql
-- Hedge designation: formal documentation of a hedging relationship
CREATE TABLE hedge_designation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    designation_ref TEXT NOT NULL,
    hedge_type      TEXT NOT NULL CHECK (hedge_type IN (
        'fair_value', 'cash_flow', 'net_investment'
    )),
    accounting_standard TEXT NOT NULL CHECK (accounting_standard IN (
        'asc815', 'ifrs9'
    )),
    hedged_risk     TEXT NOT NULL,         -- e.g. 'fx_risk', 'interest_rate_risk'
    hedged_item_description TEXT NOT NULL,
    hedging_instrument_description TEXT NOT NULL,
    inception_date  DATE NOT NULL,
    expiry_date     DATE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'dedesignated', 'expired', 'matured'
    )),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hedge_tenant ON hedge_designation(tenant_id);

-- Hedge effectiveness test: periodic testing results
CREATE TABLE hedge_effectiveness_test (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    hedge_designation_id UUID NOT NULL REFERENCES hedge_designation(id),
    test_date       DATE NOT NULL,
    test_type       TEXT NOT NULL CHECK (test_type IN (
        'prospective', 'retrospective'
    )),
    method          TEXT NOT NULL,         -- e.g. 'dollar_offset', 'regression', 'critical_terms'
    effectiveness_ratio NUMERIC(8,4),      -- e.g. 0.95 = 95%
    is_effective    BOOLEAN NOT NULL,
    hedged_item_fair_value_change NUMERIC(18,4),
    hedging_instrument_fair_value_change NUMERIC(18,4),
    ineffectiveness_amount NUMERIC(18,4),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hedge_test_designation ON hedge_effectiveness_test(hedge_designation_id);
```

## Cash Flow Forecasting

```sql
-- Forecast model: a named forecasting model configuration
CREATE TABLE forecast_model (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    model_type      TEXT NOT NULL CHECK (model_type IN (
        'manual', 'linear_regression', 'random_forest', 'lstm', 'ensemble'
    )),
    horizon_weeks   INTEGER NOT NULL DEFAULT 13,
    training_config JSONB NOT NULL DEFAULT '{}',
    -- Example training_config:
    -- {
    --   "lookback_days": 365,
    --   "features": ["day_of_week", "month", "payroll_flag", "historical_amount"],
    --   "hyperparameters": {"n_estimators": 100, "max_depth": 10}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_trained_at TIMESTAMPTZ,
    accuracy_metrics JSONB,
    -- Example accuracy_metrics:
    -- {"mape": 12.3, "rmse": 45000, "r2": 0.87, "training_samples": 730}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_forecast_model_tenant ON forecast_model(tenant_id);

-- Forecast: a specific forecast run
CREATE TABLE forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    forecast_model_id UUID REFERENCES forecast_model(id),
    legal_entity_id UUID REFERENCES legal_entity(id),  -- NULL = all entities
    name            TEXT NOT NULL,
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    forecast_date   DATE NOT NULL,         -- date the forecast was generated
    horizon_start   DATE NOT NULL,
    horizon_end     DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'published', 'superseded', 'archived'
    )),
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_forecast_tenant ON forecast(tenant_id);
CREATE INDEX idx_forecast_date ON forecast(forecast_date DESC);

-- Forecast line item: individual period within a forecast
CREATE TABLE forecast_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    forecast_id     UUID NOT NULL REFERENCES forecast(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    category        TEXT NOT NULL,         -- e.g. 'receipts', 'payroll', 'rent', 'taxes'
    projected_amount NUMERIC(18,4) NOT NULL,
    actual_amount   NUMERIC(18,4),         -- filled in when actuals are available
    variance        NUMERIC(18,4) GENERATED ALWAYS AS (
        actual_amount - projected_amount
    ) STORED,
    confidence_lower NUMERIC(18,4),        -- 95% confidence interval
    confidence_upper NUMERIC(18,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_forecast_line_forecast ON forecast_line(forecast_id);
CREATE INDEX idx_forecast_line_period ON forecast_line(period_start);

-- Forecast scenario: stress test / what-if overlay on a forecast
CREATE TABLE forecast_scenario (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    forecast_id     UUID NOT NULL REFERENCES forecast(id),
    name            TEXT NOT NULL,         -- e.g. 'Base', 'Downside', 'FX Shock'
    scenario_type   TEXT NOT NULL CHECK (scenario_type IN (
        'base', 'upside', 'downside', 'stress', 'custom'
    )),
    assumptions     JSONB NOT NULL DEFAULT '{}',
    -- Example assumptions:
    -- {
    --   "fx_shock_pct": -10,
    --   "revenue_change_pct": -15,
    --   "interest_rate_delta_bps": 200
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scenario_forecast ON forecast_scenario(forecast_id);
```

## Debt & Covenant Monitoring

```sql
-- Debt instrument: loans, credit lines, bonds
CREATE TABLE debt_instrument (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    counterparty_id UUID REFERENCES counterparty(id),  -- lender
    instrument_type TEXT NOT NULL CHECK (instrument_type IN (
        'term_loan', 'revolving_credit', 'bond', 'commercial_paper',
        'overdraft', 'letter_of_credit'
    )),
    name            TEXT NOT NULL,
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    principal_amount NUMERIC(18,4) NOT NULL,
    outstanding_amount NUMERIC(18,4) NOT NULL,
    interest_rate_type TEXT NOT NULL CHECK (interest_rate_type IN (
        'fixed', 'floating'
    )),
    interest_rate   NUMERIC(8,4),          -- fixed rate or current floating
    reference_rate  TEXT,                   -- e.g. 'SOFR', 'EURIBOR', 'SONIA'
    spread_bps      INTEGER,               -- basis points over reference
    start_date      DATE NOT NULL,
    maturity_date   DATE NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_debt_tenant ON debt_instrument(tenant_id);
CREATE INDEX idx_debt_entity ON debt_instrument(legal_entity_id);

-- Covenant: financial covenant attached to a debt instrument
CREATE TABLE covenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    debt_instrument_id UUID NOT NULL REFERENCES debt_instrument(id),
    name            TEXT NOT NULL,
    covenant_type   TEXT NOT NULL CHECK (covenant_type IN (
        'leverage_ratio', 'interest_coverage', 'current_ratio',
        'debt_to_equity', 'minimum_liquidity', 'maximum_capex', 'custom'
    )),
    test_frequency  TEXT NOT NULL CHECK (test_frequency IN (
        'monthly', 'quarterly', 'semi_annual', 'annual'
    )),
    threshold_operator TEXT NOT NULL CHECK (threshold_operator IN (
        'gte', 'lte', 'gt', 'lt', 'eq', 'between'
    )),
    threshold_value NUMERIC(18,4) NOT NULL,
    threshold_upper NUMERIC(18,4),         -- for 'between' operator
    cure_period_days INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_covenant_debt ON covenant(debt_instrument_id);

-- Covenant test result: periodic compliance check
CREATE TABLE covenant_test_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    covenant_id     UUID NOT NULL REFERENCES covenant(id),
    test_date       DATE NOT NULL,
    actual_value    NUMERIC(18,4) NOT NULL,
    threshold_value NUMERIC(18,4) NOT NULL,
    is_compliant    BOOLEAN NOT NULL,
    headroom_pct    NUMERIC(8,4),          -- % headroom to breach
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_covenant_test_covenant ON covenant_test_result(covenant_id);
CREATE INDEX idx_covenant_test_date ON covenant_test_result(test_date DESC);
```

## ERP Integration & Reconciliation

```sql
-- ERP connection: configuration for ERP data source
CREATE TABLE erp_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    erp_type        TEXT NOT NULL CHECK (erp_type IN (
        'erpnext', 'netsuite', 'quickbooks', 'sap', 'dynamics365',
        'xero', 'custom_api', 'csv'
    )),
    name            TEXT NOT NULL,
    connection_config JSONB NOT NULL DEFAULT '{}',  -- encrypted credentials
    last_sync_at    TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_erp_tenant ON erp_connection(tenant_id);

-- GL transaction: general ledger entries synced from ERP for reconciliation
CREATE TABLE gl_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    erp_connection_id UUID NOT NULL REFERENCES erp_connection(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    external_id     TEXT NOT NULL,         -- ERP transaction ID
    journal_date    DATE NOT NULL,
    gl_account      TEXT NOT NULL,
    description     TEXT,
    amount          NUMERIC(18,4) NOT NULL,
    credit_debit    CHAR(1) NOT NULL CHECK (credit_debit IN ('C', 'D')),
    currency        CHAR(3) NOT NULL REFERENCES currency(code),
    -- Reconciliation
    reconciliation_status TEXT NOT NULL DEFAULT 'unmatched',
    matched_entry_id UUID REFERENCES bank_statement_entry(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gl_tenant ON gl_transaction(tenant_id);
CREATE INDEX idx_gl_entity ON gl_transaction(legal_entity_id);
CREATE INDEX idx_gl_date ON gl_transaction(journal_date);
CREATE INDEX idx_gl_external ON gl_transaction(erp_connection_id, external_id);
```

## Users, RBAC & Audit

```sql
-- User: platform user
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    auth_provider_id TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_user_email_tenant ON app_user(tenant_id, email);

-- Role: RBAC role definition
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,         -- e.g. 'treasurer', 'analyst', 'approver', 'admin'
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions:
    -- ["cash_position:read", "payment:create", "payment:approve", "forecast:write"]
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_role_name_tenant ON role(tenant_id, name);

-- User-role assignment
CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    legal_entity_id UUID REFERENCES legal_entity(id),  -- optional entity-scoped access
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, legal_entity_id)
);

-- Audit log: immutable record of all significant actions
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          TEXT NOT NULL,         -- e.g. 'payment.created', 'payment.approved', 'position.viewed'
    resource_type   TEXT NOT NULL,         -- e.g. 'payment_order', 'bank_account'
    resource_id     UUID,
    details         JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);

-- Row-Level Security policies for multi-tenancy
ALTER TABLE legal_entity ENABLE ROW LEVEL SECURITY;
ALTER TABLE bank_account ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_order ENABLE ROW LEVEL SECURITY;
ALTER TABLE bank_statement ENABLE ROW LEVEL SECURITY;
ALTER TABLE cash_position ENABLE ROW LEVEL SECURITY;
ALTER TABLE fx_exposure ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

-- Example RLS policy (applied to all tenant-scoped tables):
CREATE POLICY tenant_isolation ON legal_entity
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
CREATE POLICY tenant_isolation ON bank_account
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
CREATE POLICY tenant_isolation ON payment_order
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

## Notification & Alert Management

```sql
-- Alert rule: configurable threshold alerts
CREATE TABLE alert_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    alert_type      TEXT NOT NULL CHECK (alert_type IN (
        'low_balance', 'high_balance', 'fx_threshold',
        'covenant_warning', 'anomaly', 'payment_failed',
        'forecast_variance', 'custom'
    )),
    conditions      JSONB NOT NULL,
    -- Example conditions for fx_threshold:
    -- {"currency": "EUR", "threshold_pct": 5, "direction": "either"}
    recipients      JSONB NOT NULL DEFAULT '[]',   -- user IDs or email addresses
    channel         TEXT NOT NULL DEFAULT 'email' CHECK (channel IN (
        'email', 'slack', 'webhook', 'in_app'
    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_rule_tenant ON alert_rule(tenant_id);

-- Alert event: fired alert instance
CREATE TABLE alert_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    alert_rule_id   UUID NOT NULL REFERENCES alert_rule(id),
    severity        TEXT NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           TEXT NOT NULL,
    message         TEXT NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_event_tenant ON alert_event(tenant_id);
CREATE INDEX idx_alert_event_created ON alert_event(created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Entity | 4 | tenant, legal_entity, currency, country |
| Bank Accounts & Connectivity | 3 | bank, bank_connection, bank_account |
| Cash Positioning | 2 | bank_account_balance, cash_position |
| Bank Statements | 2 | bank_statement, bank_statement_entry |
| Payments | 4 | counterparty, counterparty_bank_account, payment_order, payment_approval |
| Payment Tracking | 1 | payment_status_history |
| FX Rates & Exposure | 3 | fx_rate, fx_exposure, fx_trade |
| Hedge Accounting | 2 | hedge_designation, hedge_effectiveness_test |
| Forecasting | 4 | forecast_model, forecast, forecast_line, forecast_scenario |
| Debt & Covenants | 3 | debt_instrument, covenant, covenant_test_result |
| ERP Integration | 2 | erp_connection, gl_transaction |
| Users & RBAC | 3 | app_user, role, user_role |
| Audit & Alerts | 4 | audit_log, alert_rule, alert_event, (RLS policies) |
| **Total** | **37** | Core tables; junction/history tables may be added |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, prevents enumeration attacks, and simplifies multi-tenant data migration. UUIDs are generated server-side via `gen_random_uuid()`.

2. **Tenant-scoped with PostgreSQL Row-Level Security** — every tenant-scoped table carries a `tenant_id` column with RLS policies enforcing isolation at the database level, not just the application layer. This provides defense-in-depth against data leakage bugs.

3. **ISO 20022 message structure alignment** — `bank_statement` and `bank_statement_entry` tables map directly to CAMT.053 BkToCstmrStmt and Ntry elements. `payment_order` maps to PAIN.001 CstmrCdtTrfInitn. This means parsing ISO 20022 XML into the database is a direct mapping exercise.

4. **Separate balance and position tables** — `bank_account_balance` stores raw per-account balance snapshots from bank feeds; `cash_position` stores aggregated per-entity-per-currency totals. This separation allows the raw bank data to be preserved while the position can be recalculated.

5. **Generated columns for FX exposure calculations** — `fx_exposure.net_amount` and `hedge_ratio` are computed columns, ensuring they are always consistent with `gross_amount` and `hedged_amount` without application-layer synchronisation.

6. **Hedge accounting as first-class entities** — `hedge_designation` and `hedge_effectiveness_test` are separate tables rather than metadata on FX trades, because ASC 815 and IFRS 9 require formal documentation and periodic testing as distinct auditable records.

7. **Payment status as a separate history table** — rather than just tracking current status on `payment_order`, the `payment_status_history` table preserves every status transition from PAIN.002 or SWIFT gpi updates, enabling full payment lifecycle audit.

8. **Forecast model configuration in JSONB** — while the core forecast structure is relational (model, forecast, line items), the ML training configuration and accuracy metrics use JSONB because these structures vary by model type and evolve rapidly during development.

9. **Counterparty as a separate entity from legal entities** — the tenant's own subsidiaries are `legal_entity` rows; external parties (vendors, customers, banks) are `counterparty` rows. This prevents confusion between "our entities" and "their entities" in payment and FX trade contexts.

10. **SWIFT gpi UETR on payment orders** — the Unique End-to-End Transaction Reference is stored directly on `payment_order` for cross-border payment tracking, with an index for real-time gpi status queries.
