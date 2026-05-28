# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Treasury Management System · Created: 2026-05-12

## Philosophy

The graph-relational hybrid model uses traditional PostgreSQL relational tables for operational CRUD (bank accounts, payments, balances) but adds a property graph layer for the relationship-heavy queries that define treasury management: entity ownership hierarchies, intercompany cash flows, FX exposure chains, counterparty networks, and conflict-of-interest analysis. The graph layer is implemented using PostgreSQL's native recursive CTEs and a lightweight `graph_edge` table, avoiding the need for a separate graph database like Neo4j while preserving the ability to answer questions like "show me all intercompany cash flows across the EMEA entity tree" or "which entities have indirect FX exposure to GBP through their subsidiaries?"

Treasury management is fundamentally about relationships: legal entities own other legal entities; entities hold accounts at banks; banks connect to other banks via correspondent relationships; FX exposures cascade through intercompany loans; hedge designations link instruments to exposures; covenants bind debt instruments to financial ratios that depend on data from multiple entities. In a normalised relational model, these multi-hop relationship queries require complex multi-table JOINs or recursive CTEs written ad hoc. In a graph-relational model, the graph layer makes relationship traversal a first-class operation.

This approach is inspired by financial intelligence platforms (anti-money-laundering systems, beneficial ownership registries, corporate registry graph databases) that model corporate structures as property graphs. The relational tables handle the 90% of queries that are straightforward lookups and aggregations, while the graph layer handles the 10% that involve relationship traversal — but those 10% are often the most valuable queries for a treasurer managing a multi-entity, multi-currency, multi-bank operation.

**Best for:** Multi-entity corporate treasuries with complex ownership hierarchies, intercompany lending, and cross-entity FX exposure management where understanding the full relationship graph is critical for decision-making.

**Trade-offs:**
- Pro: Multi-hop relationship queries (entity hierarchies, intercompany flows, exposure chains) are natural and performant
- Pro: Relationship changes (restructuring, acquisitions, divestitures) are graph edge operations — no schema migration needed
- Pro: Enables powerful analytics: intercompany netting, ownership-weighted cash positions, cascading covenant impact
- Pro: Natural fit for AI agents that need to reason about entity relationships and dependencies
- Pro: Graph queries can reveal hidden risks (circular intercompany flows, concentrated counterparty exposure)
- Con: Additional conceptual complexity — developers must understand both relational and graph paradigms
- Con: Graph edge table can grow large with many relationship types; requires careful indexing
- Con: Recursive CTE performance degrades for very deep hierarchies (>20 levels); rare in practice for corporate structures
- Con: No native graph query language in PostgreSQL — queries use recursive CTEs which are verbose compared to Cypher or Gremlin
- Con: Graph consistency (no orphan edges, no circular ownership) must be enforced at the application level

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 20022 CAMT/PAIN | Relational payment and statement tables align with CAMT/PAIN message structure |
| ISO 4217 | Currency codes on all monetary nodes and edges |
| ISO 3166 | Country codes on entity and jurisdiction nodes |
| ISO 17442 (LEI) | LEI stored on legal entity nodes; LEI hierarchy (direct/ultimate parent) modelled as graph edges |
| FDX API v6.x | Bank connectivity modelled as edges between entity nodes and bank nodes |
| SWIFT gpi | Payment tracking modelled as edges in the payment flow subgraph |
| GLEIF (Global LEI Foundation) | Entity ownership relationships mirror the GLEIF relationship record schema: direct parent, ultimate parent, fund management |
| ASC 815 / IFRS 9 | Hedge relationships modelled as typed edges between exposure nodes and instrument nodes |
| ISDA | Counterparty relationships modelled as edges with ISDA agreement metadata |

---

## Graph Layer

```sql
-- Graph node: a lightweight registry of all entities in the graph
-- The actual data lives in the relational tables; this table provides graph identity
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       TEXT NOT NULL CHECK (node_type IN (
        'legal_entity', 'bank_account', 'bank', 'counterparty',
        'payment_order', 'fx_trade', 'fx_exposure', 'debt_instrument',
        'hedge_designation', 'forecast', 'currency'
    )),
    -- Reference to the relational table row
    entity_id       UUID NOT NULL,          -- FK to the domain table (not enforced at DB level for flexibility)
    -- Cached label for graph visualisation (denormalised)
    label           TEXT NOT NULL,
    -- Node properties (indexed for graph-scoped queries)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties for a legal_entity node:
    -- {
    --   "country_code": "DE",
    --   "functional_currency": "EUR",
    --   "entity_type": "operating",
    --   "lei": "5493001KJTIIGC8Y1R12"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gn_tenant ON graph_node(tenant_id);
CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_entity ON graph_node(entity_id);
CREATE INDEX idx_gn_properties ON graph_node USING GIN (properties);
CREATE UNIQUE INDEX idx_gn_tenant_entity ON graph_node(tenant_id, node_type, entity_id);

-- Graph edge: a typed, directed relationship between two nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    -- Source and target nodes
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    -- Relationship type
    edge_type       TEXT NOT NULL CHECK (edge_type IN (
        -- Ownership and structure
        'owns',                  -- legal_entity → legal_entity (parent owns subsidiary)
        'ultimate_parent_of',    -- legal_entity → legal_entity (GLEIF ultimate parent)
        
        -- Banking relationships
        'holds_account_at',      -- legal_entity → bank (entity has account at bank)
        'account_of',            -- bank_account → legal_entity
        'connected_via',         -- bank_account → bank (connectivity)
        
        -- Financial relationships
        'owes',                  -- legal_entity → legal_entity (intercompany loan)
        'lends_to',              -- counterparty → legal_entity (debt instrument)
        'trades_with',           -- legal_entity → counterparty (FX counterparty)
        'isda_agreement',        -- legal_entity → counterparty (ISDA master agreement)
        
        -- Treasury operations
        'pays',                  -- legal_entity → counterparty (payment flow)
        'receives_from',         -- legal_entity → counterparty (receipt flow)
        'intercompany_transfer', -- legal_entity → legal_entity (intercompany cash movement)
        
        -- FX and hedging
        'exposed_to',            -- legal_entity → currency (FX exposure)
        'hedges',                -- fx_trade → fx_exposure (hedge relationship)
        'designated_under',      -- fx_trade → hedge_designation
        
        -- Debt and covenants
        'secured_by',            -- debt_instrument → legal_entity (guarantor)
        'covenant_on',           -- debt_instrument → legal_entity (covenant applies to)
        
        -- Correspondent banking
        'correspondent_of'      -- bank → bank (correspondent relationship)
    )),
    -- Edge properties (weight, amount, dates, metadata)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties for 'owns' edge:
    -- {
    --   "ownership_pct": 100.0,
    --   "effective_date": "2020-01-15",
    --   "voting_rights_pct": 100.0,
    --   "consolidation_method": "full"
    -- }
    -- Example properties for 'intercompany_transfer' edge:
    -- {
    --   "amount": 500000,
    --   "currency": "EUR",
    --   "frequency": "monthly",
    --   "last_transfer_date": "2026-05-01",
    --   "purpose": "management_fee"
    -- }
    -- Example properties for 'exposed_to' edge:
    -- {
    --   "gross_amount": 2000000,
    --   "hedged_amount": 1500000,
    --   "net_amount": 500000,
    --   "hedge_ratio": 0.75,
    --   "exposure_type": "balance_sheet"
    -- }
    -- Temporal validity
    valid_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    valid_to        DATE,                   -- NULL = currently valid
    
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Prevent duplicate active edges of the same type between the same nodes
    UNIQUE (source_node_id, target_node_id, edge_type, valid_from)
);
CREATE INDEX idx_ge_tenant ON graph_edge(tenant_id);
CREATE INDEX idx_ge_source ON graph_edge(source_node_id);
CREATE INDEX idx_ge_target ON graph_edge(target_node_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_properties ON graph_edge USING GIN (properties);
CREATE INDEX idx_ge_active ON graph_edge(is_active) WHERE is_active = true;
CREATE INDEX idx_ge_valid ON graph_edge(valid_from, valid_to);
```

## Relational Domain Tables

The relational tables below store the operational data. Each significant entity also has a corresponding `graph_node` row for relationship traversal.

```sql
-- Tenant
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Legal entity
CREATE TABLE legal_entity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph layer
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_entity_tenant ON legal_entity(tenant_id);
CREATE INDEX idx_entity_parent ON legal_entity(parent_id);
CREATE UNIQUE INDEX idx_entity_lei ON legal_entity(lei) WHERE lei IS NOT NULL;

-- Bank
CREATE TABLE bank (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    name            TEXT NOT NULL,
    swift_bic       VARCHAR(11),
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bank_tenant ON bank(tenant_id);

-- Bank connection
CREATE TABLE bank_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_id         UUID NOT NULL REFERENCES bank(id),
    connection_type TEXT NOT NULL CHECK (connection_type IN (
        'fdx', 'psd2', 'cdr', 'swift', 'sftp', 'manual'
    )),
    status          TEXT NOT NULL DEFAULT 'pending',
    connection_config JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conn_tenant ON bank_connection(tenant_id);

-- Bank account
CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    bank_id         UUID NOT NULL REFERENCES bank(id),
    bank_connection_id UUID REFERENCES bank_connection(id),
    account_name    TEXT NOT NULL,
    iban            VARCHAR(34),
    currency        CHAR(3) NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN (
        'operating', 'savings', 'money_market', 'escrow',
        'payroll', 'tax', 'investment', 'loan', 'credit_line'
    )),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_acct_tenant ON bank_account(tenant_id);
CREATE INDEX idx_acct_entity ON bank_account(legal_entity_id);
CREATE UNIQUE INDEX idx_acct_iban ON bank_account(iban) WHERE iban IS NOT NULL;

-- Counterparty
CREATE TABLE counterparty (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    name            TEXT NOT NULL,
    counterparty_type TEXT NOT NULL CHECK (counterparty_type IN (
        'vendor', 'customer', 'bank', 'intercompany', 'government', 'other'
    )),
    country_code    CHAR(2),
    lei             VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    bank_accounts   JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cpty_tenant ON counterparty(tenant_id);
```

## Cash Positioning (Relational)

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
    source          TEXT NOT NULL,
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bal_account_date ON bank_account_balance(bank_account_id, balance_date DESC);
CREATE UNIQUE INDEX idx_bal_unique ON bank_account_balance(
    bank_account_id, balance_date, balance_type
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_pos_unique ON cash_position(
    tenant_id, legal_entity_id, position_date, currency
);
```

## Payments (Relational)

```sql
CREATE TABLE payment_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    debit_account_id UUID NOT NULL REFERENCES bank_account(id),
    counterparty_id UUID REFERENCES counterparty(id),
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
    payment_details JSONB NOT NULL DEFAULT '{}',
    status_history  JSONB NOT NULL DEFAULT '[]',
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pmt_tenant ON payment_order(tenant_id);
CREATE INDEX idx_pmt_entity ON payment_order(legal_entity_id);
CREATE INDEX idx_pmt_status ON payment_order(status);
CREATE INDEX idx_pmt_date ON payment_order(value_date);
```

## Bank Statements (Relational)

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
    format          TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'received',
    format_data     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_stmt_account_date ON bank_statement(bank_account_id, statement_date DESC);

CREATE TABLE bank_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    bank_statement_id UUID REFERENCES bank_statement(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    amount          NUMERIC(18,4) NOT NULL,
    credit_debit    CHAR(1) NOT NULL CHECK (credit_debit IN ('C', 'D')),
    currency        CHAR(3) NOT NULL,
    value_date      DATE NOT NULL,
    booking_date    DATE NOT NULL,
    counterparty_name TEXT,
    remittance_info TEXT,
    category        TEXT,
    reconciliation_status TEXT NOT NULL DEFAULT 'unmatched',
    matched_payment_id UUID,
    transaction_data JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_txn_account_date ON bank_transaction(bank_account_id, value_date DESC);
CREATE INDEX idx_txn_reconciliation ON bank_transaction(reconciliation_status)
    WHERE reconciliation_status = 'unmatched';
```

## FX, Hedging & Debt (Relational with Graph Links)

```sql
CREATE TABLE fx_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    base_currency   CHAR(3) NOT NULL,
    quote_currency  CHAR(3) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    rate_type       TEXT NOT NULL,
    rate_date       DATE NOT NULL,
    source          TEXT NOT NULL,
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fxr_pair_date ON fx_rate(base_currency, quote_currency, rate_date DESC);

CREATE TABLE fx_exposure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    exposure_date   DATE NOT NULL,
    currency        CHAR(3) NOT NULL,
    exposure_type   TEXT NOT NULL,
    gross_amount    NUMERIC(18,4) NOT NULL,
    hedged_amount   NUMERIC(18,4) NOT NULL DEFAULT 0,
    net_amount      NUMERIC(18,4) GENERATED ALWAYS AS (gross_amount - hedged_amount) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_exp_tenant ON fx_exposure(tenant_id);

CREATE TABLE fx_trade (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    counterparty_id UUID REFERENCES counterparty(id),
    trade_type      TEXT NOT NULL,
    buy_currency    CHAR(3) NOT NULL,
    buy_amount      NUMERIC(18,4) NOT NULL,
    sell_currency   CHAR(3) NOT NULL,
    sell_amount     NUMERIC(18,4) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    trade_date      DATE NOT NULL,
    settlement_date DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'open',
    mark_to_market  NUMERIC(18,4),
    instrument_data JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fxt_tenant ON fx_trade(tenant_id);
CREATE INDEX idx_fxt_settlement ON fx_trade(settlement_date);

CREATE TABLE hedge_designation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    designation_ref TEXT NOT NULL,
    hedge_type      TEXT NOT NULL,
    accounting_standard TEXT NOT NULL,
    hedged_risk     TEXT NOT NULL,
    inception_date  DATE NOT NULL,
    expiry_date     DATE,
    status          TEXT NOT NULL DEFAULT 'active',
    documentation   JSONB NOT NULL DEFAULT '{}',
    effectiveness_tests JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hedge_tenant ON hedge_designation(tenant_id);

CREATE TABLE debt_instrument (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    legal_entity_id UUID NOT NULL REFERENCES legal_entity(id),
    counterparty_id UUID REFERENCES counterparty(id),
    instrument_type TEXT NOT NULL,
    name            TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    principal_amount NUMERIC(18,4) NOT NULL,
    outstanding_amount NUMERIC(18,4) NOT NULL,
    interest_rate_type TEXT NOT NULL,
    interest_rate   NUMERIC(8,4),
    start_date      DATE NOT NULL,
    maturity_date   DATE NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    terms           JSONB NOT NULL DEFAULT '{}',
    covenant_tests  JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_debt_tenant ON debt_instrument(tenant_id);
```

## Forecasting (Relational)

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
    status          TEXT NOT NULL DEFAULT 'draft',
    model_config    JSONB NOT NULL DEFAULT '{}',
    line_items      JSONB NOT NULL DEFAULT '[]',
    scenarios       JSONB NOT NULL DEFAULT '[]',
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fcst_tenant ON forecast(tenant_id);
CREATE INDEX idx_fcst_date ON forecast(forecast_date DESC);
```

## Users & Audit (Relational)

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    roles           JSONB NOT NULL DEFAULT '[]',
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
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);

-- Row-Level Security on all tenant-scoped tables
ALTER TABLE graph_node ENABLE ROW LEVEL SECURITY;
ALTER TABLE graph_edge ENABLE ROW LEVEL SECURITY;
ALTER TABLE legal_entity ENABLE ROW LEVEL SECURITY;
ALTER TABLE bank_account ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_order ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON graph_node
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
CREATE POLICY tenant_isolation ON graph_edge
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
CREATE POLICY tenant_isolation ON legal_entity
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

## Graph Query Patterns

### Entity Ownership Hierarchy (Recursive CTE)

```sql
-- Get the full ownership tree from the holding company down
WITH RECURSIVE entity_tree AS (
    -- Anchor: start from the holding company
    SELECT
        gn.id AS node_id,
        gn.entity_id,
        gn.label AS entity_name,
        (gn.properties->>'country_code') AS country,
        (gn.properties->>'functional_currency') AS currency,
        0 AS depth,
        ARRAY[gn.label] AS path,
        100.0::NUMERIC AS effective_ownership_pct
    FROM graph_node gn
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'legal_entity'
      AND gn.properties->>'entity_type' = 'holding'
      AND gn.is_active = true

    UNION ALL

    -- Recursive: follow 'owns' edges to subsidiaries
    SELECT
        child.id AS node_id,
        child.entity_id,
        child.label AS entity_name,
        (child.properties->>'country_code') AS country,
        (child.properties->>'functional_currency') AS currency,
        parent.depth + 1,
        parent.path || child.label,
        parent.effective_ownership_pct *
            (ge.properties->>'ownership_pct')::NUMERIC / 100.0
    FROM entity_tree parent
    JOIN graph_edge ge ON ge.source_node_id = parent.node_id
        AND ge.edge_type = 'owns'
        AND ge.is_active = true
    JOIN graph_node child ON child.id = ge.target_node_id
        AND child.is_active = true
)
SELECT
    entity_name,
    country,
    currency,
    depth,
    ROUND(effective_ownership_pct, 2) AS ownership_pct,
    array_to_string(path, ' → ') AS hierarchy_path
FROM entity_tree
ORDER BY depth, entity_name;

-- Result example:
-- | entity_name          | country | currency | depth | ownership_pct | hierarchy_path                              |
-- |----------------------|---------|----------|-------|---------------|---------------------------------------------|
-- | ACME Holdings Inc    | US      | USD      | 0     | 100.00        | ACME Holdings Inc                           |
-- | ACME Europe GmbH     | DE      | EUR      | 1     | 100.00        | ACME Holdings Inc → ACME Europe GmbH        |
-- | ACME UK Ltd          | GB      | GBP      | 1     | 100.00        | ACME Holdings Inc → ACME UK Ltd             |
-- | ACME France SAS      | FR      | EUR      | 2     | 80.00         | ACME Holdings Inc → ACME Europe GmbH → ACME France SAS |
```

### Consolidated Cash Position Across Entity Tree

```sql
-- Ownership-weighted cash position for the entire corporate tree
WITH RECURSIVE entity_tree AS (
    SELECT gn.entity_id, 100.0::NUMERIC AS ownership_pct
    FROM graph_node gn
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'legal_entity'
      AND gn.properties->>'entity_type' = 'holding'
      AND gn.is_active = true
    UNION ALL
    SELECT child.entity_id,
           parent.ownership_pct * (ge.properties->>'ownership_pct')::NUMERIC / 100.0
    FROM entity_tree parent
    JOIN graph_node pn ON pn.entity_id = parent.entity_id AND pn.node_type = 'legal_entity'
    JOIN graph_edge ge ON ge.source_node_id = pn.id AND ge.edge_type = 'owns' AND ge.is_active = true
    JOIN graph_node child ON child.id = ge.target_node_id AND child.is_active = true
)
SELECT
    cp.currency,
    SUM(cp.total_balance) AS total_balance,
    SUM(cp.total_balance * et.ownership_pct / 100.0) AS ownership_weighted_balance,
    COUNT(DISTINCT cp.legal_entity_id) AS entity_count
FROM entity_tree et
JOIN cash_position cp ON cp.legal_entity_id = et.entity_id
WHERE cp.position_date = CURRENT_DATE
GROUP BY cp.currency
ORDER BY ownership_weighted_balance DESC;
```

### Intercompany Cash Flow Network

```sql
-- Show all intercompany cash flows as a network for netting analysis
SELECT
    src.label AS from_entity,
    tgt.label AS to_entity,
    (ge.properties->>'amount')::NUMERIC AS amount,
    ge.properties->>'currency' AS currency,
    ge.properties->>'frequency' AS frequency,
    ge.properties->>'purpose' AS purpose
FROM graph_edge ge
JOIN graph_node src ON src.id = ge.source_node_id
JOIN graph_node tgt ON tgt.id = ge.target_node_id
WHERE ge.tenant_id = 'tenant-uuid'
  AND ge.edge_type = 'intercompany_transfer'
  AND ge.is_active = true
ORDER BY (ge.properties->>'amount')::NUMERIC DESC;
```

### Cascading FX Exposure Through Entity Hierarchy

```sql
-- For each currency, show total FX exposure across the entity tree
-- including indirect exposure through subsidiaries
WITH RECURSIVE entity_tree AS (
    SELECT gn.entity_id, gn.label, 100.0::NUMERIC AS ownership_pct
    FROM graph_node gn
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'legal_entity'
      AND gn.properties->>'entity_type' = 'holding'
      AND gn.is_active = true
    UNION ALL
    SELECT child.entity_id, child.label,
           parent.ownership_pct * (ge.properties->>'ownership_pct')::NUMERIC / 100.0
    FROM entity_tree parent
    JOIN graph_node pn ON pn.entity_id = parent.entity_id AND pn.node_type = 'legal_entity'
    JOIN graph_edge ge ON ge.source_node_id = pn.id AND ge.edge_type = 'owns' AND ge.is_active = true
    JOIN graph_node child ON child.id = ge.target_node_id AND child.is_active = true
)
SELECT
    fe.currency,
    SUM(fe.gross_amount) AS total_gross_exposure,
    SUM(fe.hedged_amount) AS total_hedged,
    SUM(fe.net_amount) AS total_net_exposure,
    SUM(fe.gross_amount * et.ownership_pct / 100.0) AS weighted_gross,
    SUM(fe.net_amount * et.ownership_pct / 100.0) AS weighted_net,
    COUNT(DISTINCT et.entity_id) AS exposed_entity_count
FROM entity_tree et
JOIN fx_exposure fe ON fe.legal_entity_id = et.entity_id
    AND fe.exposure_date = CURRENT_DATE
GROUP BY fe.currency
HAVING SUM(fe.net_amount) != 0
ORDER BY ABS(SUM(fe.net_amount * et.ownership_pct / 100.0)) DESC;
```

### Counterparty Concentration Analysis

```sql
-- Find counterparties with relationships across multiple entities
-- (concentrated counterparty risk)
SELECT
    cp.label AS counterparty,
    COUNT(DISTINCT ge.source_node_id) AS relationship_count,
    array_agg(DISTINCT ge.edge_type) AS relationship_types,
    array_agg(DISTINCT src.label) AS related_entities,
    SUM(CASE
        WHEN ge.edge_type = 'trades_with' THEN (ge.properties->>'notional')::NUMERIC
        ELSE 0
    END) AS total_fx_notional,
    SUM(CASE
        WHEN ge.edge_type = 'lends_to' THEN (ge.properties->>'outstanding')::NUMERIC
        ELSE 0
    END) AS total_debt_exposure
FROM graph_node cp
JOIN graph_edge ge ON (ge.target_node_id = cp.id OR ge.source_node_id = cp.id)
    AND ge.is_active = true
JOIN graph_node src ON src.id = CASE
    WHEN ge.target_node_id = cp.id THEN ge.source_node_id
    ELSE ge.target_node_id
END
WHERE cp.tenant_id = 'tenant-uuid'
  AND cp.node_type = 'counterparty'
  AND cp.is_active = true
GROUP BY cp.id, cp.label
HAVING COUNT(DISTINCT ge.source_node_id) > 1
ORDER BY relationship_count DESC;
```

### Hedge Relationship Graph

```sql
-- Visualise the hedge relationship graph:
-- exposure → designated_under → hedge_designation ← hedges ← fx_trade
SELECT
    exp_node.label AS exposure,
    (exp_edge.properties->>'currency') AS currency,
    (exp_edge.properties->>'net_amount')::NUMERIC AS net_exposure,
    hedge_node.label AS hedge_designation,
    trade_node.label AS hedging_instrument,
    (trade_edge.properties->>'notional')::NUMERIC AS hedge_notional,
    (trade_edge.properties->>'rate')::NUMERIC AS hedge_rate
FROM graph_node exp_node
JOIN graph_edge exp_edge ON exp_edge.source_node_id = exp_node.id
    AND exp_edge.edge_type = 'exposed_to' AND exp_edge.is_active = true
JOIN graph_edge des_edge ON des_edge.source_node_id = exp_node.id
    AND des_edge.edge_type = 'designated_under' AND des_edge.is_active = true
JOIN graph_node hedge_node ON hedge_node.id = des_edge.target_node_id
JOIN graph_edge trade_edge ON trade_edge.target_node_id = hedge_node.id
    AND trade_edge.edge_type = 'hedges' AND trade_edge.is_active = true
JOIN graph_node trade_node ON trade_node.id = trade_edge.source_node_id
WHERE exp_node.tenant_id = 'tenant-uuid'
  AND exp_node.node_type = 'fx_exposure';
```

### Temporal Graph Query: Ownership Structure at a Past Date

```sql
-- What was the entity ownership structure on January 1, 2026?
WITH RECURSIVE entity_tree AS (
    SELECT gn.id AS node_id, gn.entity_id, gn.label, 0 AS depth
    FROM graph_node gn
    WHERE gn.tenant_id = 'tenant-uuid'
      AND gn.node_type = 'legal_entity'
      AND gn.properties->>'entity_type' = 'holding'
    UNION ALL
    SELECT child.id, child.entity_id, child.label, parent.depth + 1
    FROM entity_tree parent
    JOIN graph_edge ge ON ge.source_node_id = parent.node_id
        AND ge.edge_type = 'owns'
        AND ge.valid_from <= '2026-01-01'
        AND (ge.valid_to IS NULL OR ge.valid_to > '2026-01-01')
    JOIN graph_node child ON child.id = ge.target_node_id
)
SELECT entity_id, label, depth FROM entity_tree ORDER BY depth;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Organisation | 2 | tenant, legal_entity |
| Banking | 4 | bank, bank_connection, bank_account, bank_account_balance |
| Cash Positioning | 1 | cash_position |
| Bank Statements | 2 | bank_statement, bank_transaction |
| Payments | 2 | counterparty, payment_order |
| FX & Hedging | 4 | fx_rate, fx_exposure, fx_trade, hedge_designation |
| Debt | 1 | debt_instrument |
| Forecasting | 1 | forecast |
| Users & Audit | 2 | app_user, audit_log |
| **Total** | **21** | Plus 2 graph tables = 23 effective tables |

---

## Key Design Decisions

1. **PostgreSQL-native graph layer** — the graph is implemented with two tables (`graph_node`, `graph_edge`) using recursive CTEs for traversal. This avoids introducing a separate graph database (Neo4j, Amazon Neptune) which would add operational complexity, data synchronisation challenges, and an additional technology for the team to learn. PostgreSQL recursive CTEs handle the corporate hierarchy depths typical in treasury (5-10 levels) with excellent performance.

2. **Graph nodes reference relational entities** — `graph_node.entity_id` points to the relational table row. The graph layer does not replace the relational tables; it augments them with relationship traversal capability. This means simple CRUD operations (update a bank account, create a payment) use the relational tables directly, while relationship queries (ownership hierarchy, counterparty concentration) use the graph layer.

3. **Temporal edges with valid_from/valid_to** — graph edges have temporal validity, enabling questions like "what was the ownership structure on date X?" This is critical for treasury operations during corporate restructurings, acquisitions, and divestitures. When a subsidiary is sold, the `owns` edge gets a `valid_to` date rather than being deleted, preserving historical accuracy.

4. **JSONB edge properties for flexible weights** — edge properties (ownership percentage, intercompany transfer amounts, ISDA agreement references) are stored in JSONB because they vary by edge type. An `owns` edge has `ownership_pct`; an `intercompany_transfer` edge has `amount`, `currency`, and `frequency`. JSONB handles this polymorphism naturally.

5. **Ownership-weighted calculations** — the graph model enables ownership-weighted cash position aggregation, which is critical for consolidated treasury reporting. A 60%-owned subsidiary's EUR 1M cash position contributes EUR 600K to the consolidated view. This calculation is natural in a recursive CTE over the ownership graph but complex in a flat relational model.

6. **Intercompany netting analysis** — the `intercompany_transfer` edges create a directed graph of cash flows between entities. Graph algorithms (cycle detection, flow minimisation) can identify netting opportunities: if Entity A pays Entity B EUR 500K monthly and Entity B pays Entity A EUR 300K monthly, the net position is a single EUR 200K payment. This analysis is trivial in a graph model but requires complex self-joins in a relational model.

7. **Counterparty risk concentration** — by modelling all counterparty relationships as graph edges (trades_with, lends_to, isda_agreement), the system can instantly identify concentration risks: "JPMorgan is our FX counterparty for 3 entities, our syndicated loan lead for 2 facilities, and our primary cash management bank for 5 accounts — total exposure $15M." This multi-dimensional risk view is a natural graph traversal.

8. **GLEIF-aligned ownership model** — the `owns` and `ultimate_parent_of` edge types align with the GLEIF (Global LEI Foundation) relationship record schema, which defines direct and ultimate parent relationships for all LEI-registered entities. This alignment means the graph can be populated from GLEIF data and cross-referenced with the LEI registry.

9. **Hedge relationship visualisation** — the graph model makes hedge accounting relationships (exposure -> hedge designation -> hedging instrument -> counterparty) visually inspectable and programmatically traversable. Auditors can follow the relationship chain from an exposure to its designated hedge to the underlying FX trade to the bank counterparty — all as a graph path.

10. **AI agent graph reasoning** — the graph structure is particularly well-suited for AI agents (via MCP server) that need to reason about entity relationships. An LLM can answer "which entities have indirect GBP exposure through their subsidiaries?" by traversing the ownership graph and aggregating FX exposures — a query that would require the AI to formulate complex recursive SQL in a purely relational model but is expressed naturally as "follow the owns edges and collect exposed_to edges."
