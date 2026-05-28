# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Treasury Management System · Created: 2026-05-12

## Philosophy

The event-sourced model treats every change in the treasury system as an immutable domain event written to an append-only event store. The current state of any entity — a bank account balance, a payment order, an FX exposure — is derived by replaying its events in order, not by reading a mutable row. This is the Command Query Responsibility Segregation (CQRS) pattern: commands produce events (writes), and queries read from materialised projections (read models) that are rebuilt from the event stream.

This architecture is a natural fit for treasury management because treasury operations are inherently event-driven: a bank statement arrives (event), a payment is initiated (event), approved (event), submitted (event), settled (event). FX rates change (event). A covenant test is performed (event). Every one of these state changes must be auditable, and regulators and auditors frequently ask "what was the state of the cash position on date X?" — a question that event sourcing answers natively by replaying events up to that timestamp.

Square's Books system, Modern Treasury's ledger architecture, and most blockchain-based financial systems use this pattern. The immutable event log provides a complete, tamper-evident audit trail that satisfies the strictest regulatory requirements. The trade-off is increased complexity: developers must think in events rather than CRUD, read models must be maintained separately, and eventual consistency between the event store and projections must be managed. PostgreSQL serves as both the event store (using JSONB payloads in an append-only table) and the projection database (materialised read tables rebuilt from events).

**Best for:** Deployments where full audit trail integrity, temporal queries ("show me the position as of March 15"), and regulatory compliance are the highest priorities, and the team has experience with event-driven architectures.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every state change is preserved forever with no possibility of silent mutation
- Pro: Native temporal queries — replay events to any point in time without separate history tables
- Pro: Natural fit for treasury's event-driven workflow (statement received, payment approved, rate changed)
- Pro: Enables AI/ML training on the full event history — every cash flow pattern is captured
- Pro: Decoupled read models can be optimised independently for different query patterns
- Con: Higher implementation complexity — developers must think in events and projections, not CRUD
- Con: Eventual consistency between event store and read models requires careful handling
- Con: Event schema evolution (versioning) must be planned from day one
- Con: Read model rebuilds can be slow for large event volumes until materialisation is optimised
- Con: Debugging requires following event chains rather than inspecting a single row

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 20022 CAMT (camt.052/053/054) | Bank statement receipt is modelled as a `BankStatementReceived` event; each CAMT entry becomes a `BankTransactionRecorded` event |
| ISO 20022 PAIN (pain.001/002) | Payment initiation produces a `PaymentOrderCreated` event; each pain.002 status update produces a `PaymentStatusUpdated` event |
| ISO 4217 | Currency codes embedded in every monetary event payload |
| ISO 3166 | Country codes embedded in entity and jurisdiction events |
| ISO 17442 (LEI) | LEI stored in `LegalEntityRegistered` events |
| FDX API v6.x | Bank API sync produces `BankBalanceSynced` events with FDX-aligned account data |
| SWIFT gpi | UETR tracking produces `PaymentGpiStatusUpdated` events |
| ASC 815 / IFRS 9 | `HedgeDesignated`, `HedgeEffectivenessTestPerformed`, `HedgeDedesignated` events capture the full hedge accounting lifecycle |
| ISDA | `FxTradeExecuted` events include ISDA agreement references |
| FX Global Code | FX execution events capture all fields required by the 55 GFXC principles |

---

## Core Event Store

```sql
-- The single source of truth: an append-only event store
-- All state in the system is derived from this table
CREATE TABLE event_store (
    -- Event identity
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sequence_number BIGSERIAL NOT NULL,     -- global ordering
    
    -- Aggregate identity (the entity this event belongs to)
    aggregate_type  TEXT NOT NULL,           -- e.g. 'BankAccount', 'PaymentOrder', 'CashPosition'
    aggregate_id    UUID NOT NULL,           -- the entity's UUID
    aggregate_version BIGINT NOT NULL,       -- per-aggregate version for optimistic concurrency
    
    -- Event metadata
    event_type      TEXT NOT NULL,           -- e.g. 'PaymentOrderCreated', 'BankStatementReceived'
    event_version   SMALLINT NOT NULL DEFAULT 1,  -- schema version for event evolution
    
    -- Tenant isolation
    tenant_id       UUID NOT NULL,
    
    -- Causation and correlation
    correlation_id  UUID,                    -- links related events across aggregates
    causation_id    UUID,                    -- the event or command that caused this event
    
    -- Actor
    actor_id        UUID,                    -- user who triggered the event (NULL for system events)
    actor_type      TEXT NOT NULL DEFAULT 'user' CHECK (actor_type IN (
        'user', 'system', 'bank_api', 'scheduler', 'ai_agent'
    )),
    
    -- Payload
    data            JSONB NOT NULL,          -- the event payload (domain-specific)
    metadata        JSONB NOT NULL DEFAULT '{}',  -- non-domain context (IP, user agent, etc.)
    
    -- Timestamp
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Optimistic concurrency constraint
    UNIQUE (aggregate_type, aggregate_id, aggregate_version)
);

-- Primary query patterns
CREATE INDEX idx_event_aggregate ON event_store(aggregate_type, aggregate_id, aggregate_version);
CREATE INDEX idx_event_tenant ON event_store(tenant_id);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_occurred ON event_store(occurred_at);
CREATE INDEX idx_event_correlation ON event_store(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_event_sequence ON event_store(sequence_number);

-- Partition by month for performance at scale
-- (Shown conceptually; production would use declarative partitioning)
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Row-Level Security
ALTER TABLE event_store ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON event_store
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

## Event Type Catalogue

```sql
-- Event type registry: documents all known event types and their schemas
-- This is a reference/documentation table, not enforced at write time
CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    aggregate_type  TEXT NOT NULL,
    description     TEXT NOT NULL,
    current_version SMALLINT NOT NULL DEFAULT 1,
    payload_schema  JSONB NOT NULL,        -- JSON Schema for the event payload
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed the registry with core event types
INSERT INTO event_type_registry (event_type, aggregate_type, description, payload_schema) VALUES

-- Organisation events
('TenantCreated', 'Tenant', 'A new tenant organisation was registered', '{"type":"object","properties":{"name":{"type":"string"},"base_currency":{"type":"string"}}}'),
('LegalEntityRegistered', 'LegalEntity', 'A legal entity was added to the tenant', '{"type":"object","properties":{"name":{"type":"string"},"lei":{"type":"string"},"country_code":{"type":"string"},"functional_currency":{"type":"string"}}}'),
('LegalEntityUpdated', 'LegalEntity', 'Legal entity details were modified', '{}'),
('LegalEntityDeactivated', 'LegalEntity', 'Legal entity was marked inactive', '{}'),

-- Bank account events
('BankAccountOpened', 'BankAccount', 'A new bank account was registered', '{}'),
('BankAccountClosed', 'BankAccount', 'A bank account was closed', '{}'),
('BankConnectionEstablished', 'BankAccount', 'API or SWIFT connectivity was established', '{}'),
('BankConnectionExpired', 'BankAccount', 'Bank API consent expired', '{}'),

-- Balance and statement events
('BankBalanceSynced', 'BankAccount', 'Balance was updated from bank API or statement', '{}'),
('BankStatementReceived', 'BankStatement', 'A bank statement file/message was received', '{}'),
('BankTransactionRecorded', 'BankStatement', 'An individual transaction from a statement was recorded', '{}'),

-- Cash position events
('CashPositionCalculated', 'CashPosition', 'Aggregated cash position was computed', '{}'),
('CashPositionAnomalyDetected', 'CashPosition', 'AI detected an anomaly in cash flow pattern', '{}'),

-- Payment events
('PaymentOrderCreated', 'PaymentOrder', 'A new payment instruction was created', '{}'),
('PaymentOrderApproved', 'PaymentOrder', 'Payment was approved by an authorised user', '{}'),
('PaymentOrderRejected', 'PaymentOrder', 'Payment was rejected by an approver', '{}'),
('PaymentOrderSubmitted', 'PaymentOrder', 'Payment was submitted to the bank', '{}'),
('PaymentStatusUpdated', 'PaymentOrder', 'Bank returned a status update (pain.002)', '{}'),
('PaymentSettled', 'PaymentOrder', 'Payment was confirmed settled', '{}'),
('PaymentReturned', 'PaymentOrder', 'Payment was returned by the bank', '{}'),
('PaymentGpiStatusUpdated', 'PaymentOrder', 'SWIFT gpi tracking update received', '{}'),

-- FX events
('FxRateRecorded', 'FxRate', 'An exchange rate observation was recorded', '{}'),
('FxExposureCalculated', 'FxExposure', 'Net currency exposure was computed', '{}'),
('FxTradeExecuted', 'FxTrade', 'An FX trade (forward, spot, swap) was executed', '{}'),
('FxTradeSettled', 'FxTrade', 'An FX trade reached settlement', '{}'),
('FxTradeCancelled', 'FxTrade', 'An FX trade was cancelled', '{}'),

-- Hedge accounting events
('HedgeDesignated', 'HedgeDesignation', 'A formal hedge relationship was designated under ASC 815 / IFRS 9', '{}'),
('HedgeEffectivenessTestPerformed', 'HedgeDesignation', 'Periodic effectiveness test was completed', '{}'),
('HedgeDedesignated', 'HedgeDesignation', 'Hedge relationship was dedesignated', '{}'),

-- Forecast events
('ForecastGenerated', 'Forecast', 'A cash flow forecast was generated', '{}'),
('ForecastPublished', 'Forecast', 'A forecast was published for stakeholder review', '{}'),
('ForecastActualsUpdated', 'Forecast', 'Actual values were filled in against forecast lines', '{}'),
('ForecastModelTrained', 'ForecastModel', 'An ML forecasting model was retrained', '{}'),

-- Covenant events
('CovenantDefined', 'Covenant', 'A financial covenant was defined for a debt instrument', '{}'),
('CovenantTestPerformed', 'Covenant', 'Periodic covenant compliance test was completed', '{}'),
('CovenantBreachWarning', 'Covenant', 'AI detected approaching covenant breach', '{}'),

-- Reconciliation events
('ReconciliationMatchFound', 'Reconciliation', 'Bank entry was matched to GL transaction', '{}'),
('ReconciliationMatchRejected', 'Reconciliation', 'A proposed match was rejected', '{}');
```

## Example Event Payloads

```sql
-- Example: PaymentOrderCreated event
-- INSERT INTO event_store (aggregate_type, aggregate_id, aggregate_version, event_type,
--                          tenant_id, actor_id, data) VALUES
-- ('PaymentOrder', '550e8400-e29b-41d4-a716-446655440000', 1, 'PaymentOrderCreated',
--  'tenant-uuid', 'user-uuid',
--  '{
--    "legal_entity_id": "entity-uuid",
--    "debit_account_id": "account-uuid",
--    "counterparty": {
--      "name": "Acme Supplies Ltd",
--      "iban": "DE89370400440532013000",
--      "swift_bic": "COBADEFFXXX"
--    },
--    "amount": 150000.00,
--    "currency": "EUR",
--    "payment_type": "sepa",
--    "direction": "credit",
--    "value_date": "2026-05-15",
--    "end_to_end_id": "INV-2026-001234",
--    "remittance_info": "Invoice #2026-001234"
--  }');

-- Example: BankBalanceSynced event
-- INSERT INTO event_store (aggregate_type, aggregate_id, aggregate_version, event_type,
--                          tenant_id, actor_type, data) VALUES
-- ('BankAccount', 'account-uuid', 42, 'BankBalanceSynced',
--  'tenant-uuid', 'bank_api',
--  '{
--    "balance_type": "closing_booked",
--    "amount": 2450000.00,
--    "currency": "USD",
--    "balance_date": "2026-05-11",
--    "source": "fdx",
--    "provider": "plaid",
--    "previous_balance": 2380000.00,
--    "change": 70000.00
--  }');

-- Example: FxTradeExecuted event
-- INSERT INTO event_store (aggregate_type, aggregate_id, aggregate_version, event_type,
--                          tenant_id, actor_id, data) VALUES
-- ('FxTrade', 'trade-uuid', 1, 'FxTradeExecuted',
--  'tenant-uuid', 'user-uuid',
--  '{
--    "legal_entity_id": "entity-uuid",
--    "trade_type": "forward",
--    "buy_currency": "USD",
--    "buy_amount": 1000000.00,
--    "sell_currency": "EUR",
--    "sell_amount": 920000.00,
--    "rate": 1.0869,
--    "trade_date": "2026-05-12",
--    "settlement_date": "2026-08-12",
--    "counterparty": "JPMorgan Chase",
--    "isda_agreement_ref": "ISDA-2024-JPM-001",
--    "hedge_designation_id": "hedge-uuid"
--  }');

-- Example: CovenantTestPerformed event
-- INSERT INTO event_store (aggregate_type, aggregate_id, aggregate_version, event_type,
--                          tenant_id, actor_type, data) VALUES
-- ('Covenant', 'covenant-uuid', 5, 'CovenantTestPerformed',
--  'tenant-uuid', 'system',
--  '{
--    "debt_instrument_id": "loan-uuid",
--    "covenant_type": "leverage_ratio",
--    "test_date": "2026-03-31",
--    "actual_value": 2.8,
--    "threshold_value": 3.5,
--    "threshold_operator": "lte",
--    "is_compliant": true,
--    "headroom_pct": 20.0,
--    "next_test_date": "2026-06-30"
--  }');
```

## Materialised Read Models (Projections)

These tables are derived entirely from the event store. They can be rebuilt from scratch by replaying all events.

```sql
-- Projection: current bank account state
CREATE TABLE proj_bank_account (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    legal_entity_id UUID NOT NULL,
    bank_name       TEXT NOT NULL,
    swift_bic       VARCHAR(11),
    account_name    TEXT NOT NULL,
    account_number_masked TEXT,
    iban            VARCHAR(34),
    currency        CHAR(3) NOT NULL,
    account_type    TEXT NOT NULL,
    connection_type TEXT,
    connection_status TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_balance    NUMERIC(18,4),
    last_balance_date DATE,
    last_sync_at    TIMESTAMPTZ,
    version         BIGINT NOT NULL,       -- latest aggregate version applied
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_bank_tenant ON proj_bank_account(tenant_id);
CREATE INDEX idx_proj_bank_entity ON proj_bank_account(legal_entity_id);

-- Projection: current cash position (latest per entity/currency)
CREATE TABLE proj_cash_position (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    legal_entity_id UUID NOT NULL,
    legal_entity_name TEXT NOT NULL,
    position_date   DATE NOT NULL,
    currency        CHAR(3) NOT NULL,
    total_balance   NUMERIC(18,4) NOT NULL,
    available_balance NUMERIC(18,4),
    account_count   INTEGER NOT NULL,
    base_currency_amount NUMERIC(18,4),
    fx_rate_used    NUMERIC(18,8),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, legal_entity_id, position_date, currency)
);
CREATE INDEX idx_proj_position_tenant ON proj_cash_position(tenant_id);

-- Projection: current payment order state
CREATE TABLE proj_payment_order (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    legal_entity_id UUID NOT NULL,
    debit_account_id UUID NOT NULL,
    counterparty_name TEXT,
    credit_iban     TEXT,
    amount          NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    payment_type    TEXT NOT NULL,
    direction       TEXT NOT NULL,
    value_date      DATE,
    end_to_end_id   TEXT,
    uetr            UUID,
    status          TEXT NOT NULL,
    -- Status timeline (populated from events)
    created_at      TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    submitted_at    TIMESTAMPTZ,
    settled_at      TIMESTAMPTZ,
    -- Approvals (denormalised from events)
    approval_count  INTEGER NOT NULL DEFAULT 0,
    approvers       JSONB NOT NULL DEFAULT '[]',
    -- Latest status detail
    last_status_reason TEXT,
    last_status_source TEXT,
    version         BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_payment_tenant ON proj_payment_order(tenant_id);
CREATE INDEX idx_proj_payment_status ON proj_payment_order(status);
CREATE INDEX idx_proj_payment_date ON proj_payment_order(value_date);

-- Projection: FX exposure summary
CREATE TABLE proj_fx_exposure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    legal_entity_id UUID NOT NULL,
    exposure_date   DATE NOT NULL,
    currency        CHAR(3) NOT NULL,
    exposure_type   TEXT NOT NULL,
    gross_amount    NUMERIC(18,4) NOT NULL,
    hedged_amount   NUMERIC(18,4) NOT NULL DEFAULT 0,
    net_amount      NUMERIC(18,4) NOT NULL,
    hedge_ratio     NUMERIC(5,4) NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, legal_entity_id, exposure_date, currency, exposure_type)
);
CREATE INDEX idx_proj_exposure_tenant ON proj_fx_exposure(tenant_id);

-- Projection: latest forecast with line items (denormalised for fast reads)
CREATE TABLE proj_forecast_summary (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    forecast_model_name TEXT,
    legal_entity_id UUID,
    name            TEXT NOT NULL,
    currency        CHAR(3) NOT NULL,
    forecast_date   DATE NOT NULL,
    horizon_start   DATE NOT NULL,
    horizon_end     DATE NOT NULL,
    status          TEXT NOT NULL,
    total_projected NUMERIC(18,4),
    total_actual    NUMERIC(18,4),
    total_variance  NUMERIC(18,4),
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- Example line_items:
    -- [
    --   {"period_start":"2026-05-12","period_end":"2026-05-18","category":"receipts",
    --    "projected":450000,"actual":null,"confidence_lower":380000,"confidence_upper":520000}
    -- ]
    version         BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_forecast_tenant ON proj_forecast_summary(tenant_id);
CREATE INDEX idx_proj_forecast_date ON proj_forecast_summary(forecast_date DESC);

-- Projection: covenant compliance status
CREATE TABLE proj_covenant_status (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    debt_instrument_name TEXT NOT NULL,
    covenant_name   TEXT NOT NULL,
    covenant_type   TEXT NOT NULL,
    last_test_date  DATE,
    last_actual_value NUMERIC(18,4),
    threshold_value NUMERIC(18,4) NOT NULL,
    threshold_operator TEXT NOT NULL,
    is_compliant    BOOLEAN,
    headroom_pct    NUMERIC(8,4),
    next_test_date  DATE,
    test_frequency  TEXT NOT NULL,
    breach_warning  BOOLEAN NOT NULL DEFAULT false,
    version         BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_covenant_tenant ON proj_covenant_status(tenant_id);

-- Projection: alert/anomaly feed (denormalised for dashboard)
CREATE TABLE proj_alert_feed (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    event_id        UUID NOT NULL,          -- source event in event_store
    alert_type      TEXT NOT NULL,
    severity        TEXT NOT NULL,
    title           TEXT NOT NULL,
    message         TEXT NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    is_acknowledged BOOLEAN NOT NULL DEFAULT false,
    occurred_at     TIMESTAMPTZ NOT NULL,
    acknowledged_at TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_alert_tenant ON proj_alert_feed(tenant_id);
CREATE INDEX idx_proj_alert_unack ON proj_alert_feed(tenant_id, is_acknowledged)
    WHERE is_acknowledged = false;
```

## Projection Tracking & Rebuild Infrastructure

```sql
-- Projection checkpoint: tracks where each projection has processed up to
CREATE TABLE projection_checkpoint (
    projection_name TEXT PRIMARY KEY,       -- e.g. 'proj_bank_account', 'proj_cash_position'
    last_sequence   BIGINT NOT NULL DEFAULT 0,
    last_event_id   UUID,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'rebuilding', 'paused', 'error'
    )),
    error_message   TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Snapshot store: periodic aggregate snapshots to avoid full replay
CREATE TABLE aggregate_snapshot (
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version BIGINT NOT NULL,       -- aggregate version at snapshot time
    state           JSONB NOT NULL,         -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);
CREATE INDEX idx_snapshot_type ON aggregate_snapshot(aggregate_type);
```

## Reference Data (Non-Event-Sourced)

```sql
-- These tables are static reference data, not event-sourced
CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,
    numeric_code    CHAR(3),
    name            TEXT NOT NULL,
    minor_units     SMALLINT NOT NULL DEFAULT 2,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE country (
    code            CHAR(2) PRIMARY KEY,
    alpha3          CHAR(3) NOT NULL,
    name            TEXT NOT NULL,
    region          TEXT
);
```

## Tenant & User Management (Lightweight Relational)

```sql
-- Tenants and users are managed relationally for auth performance
-- but their lifecycle events are also written to the event store

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_user_email_tenant ON app_user(tenant_id, email);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_role_name_tenant ON role(tenant_id, name);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    legal_entity_id UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);
```

## Key Query Patterns

### Replaying an Aggregate's History

```sql
-- Reconstruct the full history of a payment order
SELECT event_type, data, occurred_at, actor_id
FROM event_store
WHERE aggregate_type = 'PaymentOrder'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY aggregate_version;
```

### Temporal Query: Cash Position at a Past Date

```sql
-- What was the cash position as of March 15, 2026?
-- Replay all CashPositionCalculated events up to that date
SELECT
    (data->>'legal_entity_id')::UUID AS legal_entity_id,
    data->>'currency' AS currency,
    (data->>'total_balance')::NUMERIC AS total_balance,
    occurred_at
FROM event_store
WHERE event_type = 'CashPositionCalculated'
  AND tenant_id = 'tenant-uuid'
  AND occurred_at <= '2026-03-15 23:59:59+00'
ORDER BY occurred_at DESC;
-- (Application code: take the latest event per entity/currency combination)
```

### Cross-Aggregate Correlation

```sql
-- Find all events related to a specific payment lifecycle
SELECT event_type, aggregate_type, aggregate_id, data, occurred_at
FROM event_store
WHERE correlation_id = 'correlation-uuid'
ORDER BY sequence_number;
-- Returns: PaymentOrderCreated, PaymentOrderApproved, PaymentOrderSubmitted,
--          PaymentStatusUpdated, BankTransactionRecorded, ReconciliationMatchFound
```

### AI/ML Training Data Export

```sql
-- Export all cash flow events for ML model training
SELECT
    tenant_id,
    (data->>'bank_account_id')::UUID AS account_id,
    data->>'currency' AS currency,
    (data->>'amount')::NUMERIC AS amount,
    data->>'credit_debit' AS direction,
    (data->>'value_date')::DATE AS value_date,
    data->>'bank_transaction_code' AS category,
    occurred_at
FROM event_store
WHERE event_type = 'BankTransactionRecorded'
  AND tenant_id = 'tenant-uuid'
  AND occurred_at >= now() - INTERVAL '2 years'
ORDER BY occurred_at;
```

### Event Store Statistics

```sql
-- Monitor event store health and volume
SELECT
    event_type,
    COUNT(*) AS event_count,
    MIN(occurred_at) AS earliest,
    MAX(occurred_at) AS latest
FROM event_store
WHERE tenant_id = 'tenant-uuid'
GROUP BY event_type
ORDER BY event_count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store, event_type_registry, projection_checkpoint |
| Snapshot Store | 1 | aggregate_snapshot |
| Read Projections | 7 | proj_bank_account, proj_cash_position, proj_payment_order, proj_fx_exposure, proj_forecast_summary, proj_covenant_status, proj_alert_feed |
| Reference Data | 2 | currency, country |
| Auth & Tenancy | 4 | tenant, app_user, role, user_role |
| **Total** | **17** | Dramatically fewer tables; complexity is in events, not schema |

---

## Key Design Decisions

1. **Single event store table with JSONB payloads** — rather than separate tables per event type, all events share one table. The `event_type` column identifies the event, and the `data` JSONB column carries the payload. This simplifies infrastructure (one table to partition, backup, and replicate) while JSONB indexing provides query flexibility. PostgreSQL's JSONB performance is excellent for this pattern.

2. **Optimistic concurrency via aggregate versioning** — the `UNIQUE (aggregate_type, aggregate_id, aggregate_version)` constraint prevents two concurrent commands from producing conflicting events for the same aggregate. This is the standard event sourcing concurrency control, matching the pattern described by Modern Treasury's ledger architecture.

3. **Correlation and causation IDs** — `correlation_id` links all events that belong to the same business process (e.g., a payment from creation to settlement), while `causation_id` links an event to the specific event or command that caused it. This enables full traceability across aggregate boundaries.

4. **Actor tracking with type discrimination** — every event records who or what produced it: a human user, a system process, a bank API sync, a scheduled job, or an AI agent. This is essential for audit trails where regulators need to know whether a human or an automated process initiated an action.

5. **Projections are disposable** — all `proj_*` tables can be dropped and rebuilt from the event store. The `projection_checkpoint` table tracks how far each projection has processed, enabling catch-up after downtime and full rebuilds when projection schemas change.

6. **Aggregate snapshots for replay performance** — for aggregates with hundreds or thousands of events (e.g., a bank account with years of balance syncs), the `aggregate_snapshot` table stores periodic state snapshots. Replay starts from the latest snapshot rather than the first event, dramatically reducing reconstruction time.

7. **Time-range partitioning of the event store** — the event store should be partitioned by `occurred_at` (monthly or quarterly). This ensures that temporal queries ("events in Q1 2026") only scan relevant partitions, and old partitions can be moved to cold storage. PostgreSQL declarative partitioning handles this natively.

8. **Reference data is not event-sourced** — currencies and countries are static reference data that changes rarely and has no audit trail requirement. Making them event-sourced would add complexity without value.

9. **Event schema versioning** — the `event_version` column on each event allows payload schemas to evolve over time. When the application reads an older event version, it applies an upcaster function to transform it to the current schema. This avoids the need for backfill migrations on immutable events.

10. **Natural fit for MCP server** — the event store makes it trivial to expose treasury operations to AI agents via MCP: a `TreasuryQueryTool` reads from projections, and a `TreasuryCommandTool` produces events. The immutable event log means AI actions are fully auditable — you can always see exactly what the AI agent did and why.
