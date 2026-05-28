# Treasury Management System -- Development Plan

> Project: Treasury Management System (Candidate #66)
> Created: 2026-05-25
> Status: Planning

---

## Technology Decisions

### Language & Runtime: TypeScript on Node.js (v22 LTS)

**Rationale:** TypeScript provides type safety across the full stack (API, business logic, SDK generation) while maintaining the largest available developer pool for an open-source project. Treasury systems process structured financial messages (ISO 20022 XML, JSONB payloads, FDX API responses) where TypeScript's type system catches data mapping errors at compile time. Node.js v22 LTS offers native ESM, stable worker threads for ML inference offloading, and the broadest ecosystem of Open Banking and financial API client libraries (Plaid SDK, Stripe patterns). The mid-market target audience (CFOs at $50M--$500M companies) will rely on community contributors; TypeScript has the lowest barrier to contribution among type-safe languages.

### Framework: Fastify

**Rationale:** Fastify over Express for its JSON Schema-based request/response validation (aligning with OAS 3.1 publication requirements from standards.md), superior throughput for the high-volume bank statement ingestion pipeline, and first-class TypeScript support. Fastify's plugin architecture maps cleanly to the modular domain boundaries (cash positioning, payments, forecasting, FX risk). Its built-in serialization outperforms Express by 2--3x for JSON-heavy treasury API responses.

### Database: PostgreSQL 16

**Rationale:** PostgreSQL is the only viable choice for a treasury system requiring: NUMERIC(18,4) precision for financial amounts, Row-Level Security for multi-tenant isolation, JSONB with GIN indexes for jurisdiction-variant bank connection configurations, partitioning for high-volume event and balance history tables, and recursive CTEs for entity hierarchy traversal. All four data model suggestions assume PostgreSQL. The hybrid relational + JSONB approach (Data Model Suggestion 3) is selected for MVP -- it delivers 19 tables (vs. 37 normalized) while preserving relational columns for all query-critical fields and JSONB for jurisdiction-variant data (FDX vs. PSD2 vs. CDR connection configs, payment-type-specific details, instrument-specific terms).

### Data Model: Hybrid Relational + JSONB (Suggestion 3), with Event Sourcing for Audit-Critical Paths

**Rationale:** The hybrid model (Suggestion 3) provides the fastest path to MVP with ~50% fewer tables than fully normalized (Suggestion 1), while retaining relational integrity for query-critical fields. However, pure CRUD loses the immutable audit trail that treasury regulators require. The solution is a targeted hybrid: core operational tables follow Suggestion 3's hybrid relational+JSONB pattern, while an append-only `audit_event` table (inspired by Suggestion 2's event store) captures immutable records of all payment approvals, balance changes, and FX trades. This is not full CQRS -- it is audit logging with event structure. The graph layer (Suggestion 4) is deferred to Phase 8 when multi-entity hierarchy queries justify its complexity.

### ML/AI: Python sidecar with scikit-learn and PyTorch

**Rationale:** Cash flow forecasting (random forest, LSTM) and anomaly detection models are implemented in Python -- the language with the deepest ML ecosystem (scikit-learn, PyTorch, statsmodels). The Python service runs as a sidecar, called from the TypeScript API via gRPC or HTTP. This separation allows ML engineers to iterate on models independently of the API team. The MDPI 2025 study validating 37% error reduction used Python implementations of random forest and MLP neural networks; replicating their methodology requires the same tooling.

### LLM Integration: Anthropic Claude API via MCP Server

**Rationale:** The natural language treasury query feature ("How much cash do we have in Europe today?") and autonomous FX monitoring require LLM integration. Exposing the TMS as an MCP server (as recommended in standards.md) enables any MCP-compatible AI assistant to query cash positions, retrieve forecasts, and initiate payment approvals. No existing commercial TMS offers MCP integration -- this is a first-mover differentiator. The MCP server pattern also supports the autonomous FX risk monitoring agent described in the AI-native opportunity section of research.md.

### Frontend: Next.js 15 with shadcn/ui

**Rationale:** Next.js provides server-side rendering for the cash position dashboard (critical for real-time balance display), API routes co-located with the frontend for rapid iteration, and the largest React component ecosystem. shadcn/ui provides accessible, customizable UI primitives that can be themed to match corporate treasury aesthetics (professional, data-dense) without building a design system from scratch. The treasury dashboard requires complex data visualization (multi-currency position charts, forecast vs. actuals overlays, FX exposure heatmaps) -- Next.js + Recharts/D3 handles this well.

### Bank Connectivity: Plaid (US/FDX), TrueLayer (EU/PSD2), direct CDR (AU)

**Rationale:** Building direct FDX/PSD2/CDR integrations from scratch would consume 6+ months of MVP development time. Plaid provides connectivity to 12,000+ US banks via a single integration; TrueLayer covers EU/UK banks under PSD2. These aggregators handle OAuth consent flows, token refresh, and bank-specific API variations. Direct CDR integration is simpler (fewer banks, well-documented standards). CSV import provides the universal fallback for banks not covered by any API. This matches Trovata's connectivity model (identified in research.md as the reference architecture for API-first bank connectivity).

### Authentication: NextAuth.js with OIDC

**Rationale:** Corporate treasury users expect SSO via their company's identity provider (Azure AD, Okta, Google Workspace). NextAuth.js supports all major OIDC providers out of the box. For smaller companies without SSO, email/password with MFA provides a fallback. Payment approval workflows require strong authentication -- TOTP-based MFA is mandatory for payment approvers.

### Deployment: Docker Compose (self-hosted), managed cloud option via Kubernetes

**Rationale:** The mid-market target audience splits between companies that want on-premises control (regulated industries, data sovereignty requirements) and those that prefer managed cloud. Docker Compose provides a single-command self-hosted deployment; Kubernetes manifests support the managed cloud offering. PostgreSQL, the Node.js API, the Python ML sidecar, and Redis (for job queues and caching) are the four containers.

### Job Queue: BullMQ on Redis

**Rationale:** Treasury operations include many async workflows: bank statement polling (every 15 minutes), balance aggregation, forecast model retraining, payment status polling, FX rate updates, alert evaluation, and reconciliation matching. BullMQ provides reliable job queuing with retry logic, dead letter queues, rate limiting, and cron-based scheduling -- all required for production treasury operations. Redis also serves as the cache layer for real-time cash position queries.

---

## Project Structure

```
treasury-management-system/
  apps/
    api/                        # Fastify API server (TypeScript)
      src/
        modules/
          auth/                 # Authentication, RBAC, session management
          tenant/               # Multi-tenant management
          entity/               # Legal entity hierarchy
          bank-connection/      # Bank API connectivity (Plaid, TrueLayer, CSV)
          bank-account/         # Bank account CRUD and balance sync
          cash-position/        # Cash position aggregation and dashboard data
          bank-statement/       # Statement ingestion and parsing (CAMT, MT940, CSV)
          payment/              # Payment orders, approvals, bank submission
          reconciliation/       # Bank-to-GL matching engine
          fx-rate/              # FX rate ingestion and management
          fx-exposure/          # FX exposure calculation and monitoring
          fx-trade/             # FX trade management (forwards, spots, swaps)
          hedge/                # Hedge designation and effectiveness testing
          forecast/             # Cash flow forecast management and API
          debt/                 # Debt instruments and covenant monitoring
          alert/                # Alert rules, evaluation, and notification
          audit/                # Audit event logging and query
          mcp/                  # MCP server for AI agent integration
        common/
          database/             # PostgreSQL connection, migrations, RLS
          queue/                # BullMQ job definitions and workers
          parsers/              # ISO 20022 XML, MT940, CSV parsers
          security/             # Encryption, token management, PCI compliance
        config/
        server.ts
    web/                        # Next.js frontend
      src/
        app/
          (auth)/               # Login, SSO callback, MFA setup
          (dashboard)/          # Main treasury dashboard
            cash-position/      # Cash position views
            payments/           # Payment management UI
            forecasts/          # Forecast views and scenario planning
            fx/                 # FX exposure and trade management
            bank-accounts/      # Bank account management
            reports/            # Reporting and analytics
            settings/           # Tenant, user, alert configuration
          api/                  # Next.js API routes (BFF pattern)
        components/
          ui/                   # shadcn/ui primitives
          treasury/             # Domain-specific components
          charts/               # Recharts/D3 visualizations
    ml/                         # Python ML sidecar
      src/
        models/
          cash_forecast/        # Random forest and LSTM forecast models
          anomaly_detection/    # Cash flow anomaly detection
          categorization/       # Transaction categorization
        api/                    # FastAPI or gRPC server
        training/               # Model training pipelines
  packages/
    shared/                     # Shared TypeScript types and utilities
      src/
        types/                  # Domain types (ISO 20022, payment, forecast)
        parsers/                # Shared parsing utilities
        validation/             # Zod schemas for API validation
    sdk/                        # Generated TypeScript SDK for API consumers
    iso20022/                   # ISO 20022 message type definitions and parsers
  infrastructure/
    docker/                     # Docker Compose for local dev and self-hosted
    k8s/                        # Kubernetes manifests for managed cloud
    migrations/                 # PostgreSQL migration files (node-pg-migrate)
    seeds/                      # Reference data seeds (currencies, countries)
  docs/
    api/                        # OpenAPI 3.1 specification
    architecture/               # Architecture decision records
    deployment/                 # Deployment guides
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Bank Connectivity & Cash Positioning
    |
    +---------------------------+
    |                           |
    v                           v
Phase 3: Payments &         Phase 4: Cash Flow
  Reconciliation              Forecasting (ML)
    |                           |
    +---------------------------+
    |
    v
Phase 5: FX Risk & Exposure Management
    |
    v
Phase 6: Debt, Covenants & Alerts
    |
    +---------------------------+
    |                           |
    v                           v
Phase 7: AI/LLM Integration  Phase 8: Advanced Treasury
  & MCP Server                  (Hedge Accounting, Graph Layer)
    |                           |
    +---------------------------+
    |
    v
Phase 9: ISO 20022 & SWIFT Integration
    |
    v
Phase 10: Production Hardening & Deployment
```

---

## Phase 1: Foundation & Core Infrastructure

**Goal:** Establish the project skeleton, database schema, authentication, multi-tenancy, and legal entity management. All subsequent phases build on this foundation.

**Duration:** 4--6 weeks

### Task 1.1: Project Scaffolding & Monorepo Setup

**What:** Initialize the monorepo with apps/api (Fastify), apps/web (Next.js), apps/ml (Python), and packages/shared. Configure TypeScript, ESLint, Prettier, Vitest, and CI pipeline.

**Design:**

```typescript
// apps/api/src/server.ts
import Fastify from 'fastify';
import { registerModules } from './modules';
import { databasePlugin } from './common/database/plugin';
import { authPlugin } from './modules/auth/plugin';

const app = Fastify({
  logger: true,
  ajv: { customOptions: { allErrors: true } },
});

// Register core plugins
app.register(databasePlugin);
app.register(authPlugin);

// Register domain modules
app.register(registerModules, { prefix: '/api/v1' });

await app.listen({ port: 3001, host: '0.0.0.0' });
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: treasury
      POSTGRES_USER: treasury
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  api:
    build: ./apps/api
    ports: ["3001:3001"]
    depends_on: [postgres, redis]
    environment:
      DATABASE_URL: postgresql://treasury:${DB_PASSWORD}@postgres:5432/treasury
      REDIS_URL: redis://redis:6379

  web:
    build: ./apps/web
    ports: ["3000:3000"]
    depends_on: [api]

  ml:
    build: ./apps/ml
    ports: ["8000:8000"]
    depends_on: [postgres]

volumes:
  pgdata:
```

**Testing:**
- Verify `docker compose up` starts all services and they are reachable
- Verify TypeScript compilation succeeds with strict mode
- Verify Vitest runs with zero tests passing (scaffold)
- Verify ESLint and Prettier pass on empty codebase
- Verify hot reload works for api and web services

### Task 1.2: Database Schema -- Core Tables & Migration System

**What:** Implement the foundational database tables (tenant, legal_entity, currency, country, app_user, role, audit_log) using node-pg-migrate. Seed reference data (ISO 4217 currencies, ISO 3166 countries). Implement Row-Level Security policies.

**Design:**

```sql
-- migrations/001_foundation.sql
-- Tenant, legal_entity, currency, country tables
-- As defined in Data Model Suggestion 3, Organisation & Entities section

-- Seed ISO 4217 currencies (subset shown)
INSERT INTO currency (code, numeric_code, name, minor_units) VALUES
('USD', '840', 'US Dollar', 2),
('EUR', '978', 'Euro', 2),
('GBP', '826', 'Pound Sterling', 2),
('JPY', '392', 'Japanese Yen', 0),
('CHF', '756', 'Swiss Franc', 2),
('AUD', '036', 'Australian Dollar', 2),
('CAD', '124', 'Canadian Dollar', 2),
('CNY', '156', 'Chinese Yuan', 2),
('HKD', '344', 'Hong Kong Dollar', 2),
('SGD', '702', 'Singapore Dollar', 2);
-- Full seed includes all 180 active ISO 4217 codes

-- Seed ISO 3166 countries (subset shown)
INSERT INTO country (code, alpha3, name, region) VALUES
('US', 'USA', 'United States', 'Americas'),
('GB', 'GBR', 'United Kingdom', 'EMEA'),
('DE', 'DEU', 'Germany', 'EMEA'),
('FR', 'FRA', 'France', 'EMEA'),
('JP', 'JPN', 'Japan', 'APAC'),
('AU', 'AUS', 'Australia', 'APAC'),
('SG', 'SGP', 'Singapore', 'APAC');

-- Row-Level Security
ALTER TABLE legal_entity ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON legal_entity
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

```typescript
// apps/api/src/common/database/rls.ts
import { Pool, PoolClient } from 'pg';

export async function withTenantContext<T>(
  pool: Pool,
  tenantId: string,
  fn: (client: PoolClient) => Promise<T>
): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query("SET LOCAL app.current_tenant = $1", [tenantId]);
    return await fn(client);
  } finally {
    client.release();
  }
}
```

**Testing:**
- Migration runs cleanly on empty database: `node-pg-migrate up` exits 0
- Migration rollback works: `node-pg-migrate down` exits 0, schema is empty
- Currency seed contains all 180 active ISO 4217 codes; verify `SELECT COUNT(*) FROM currency WHERE is_active = true` = 180
- Country seed contains all 249 ISO 3166-1 entries
- RLS prevents cross-tenant reads: insert data for tenant A, set context to tenant B, verify `SELECT * FROM legal_entity` returns zero rows
- RLS allows same-tenant reads: set context to tenant A, verify rows returned
- `withTenantContext` sets and clears the session variable correctly
- Unique constraint on `legal_entity(lei)` prevents duplicate LEIs
- Entity hierarchy: `parent_id` self-reference works for 5 levels deep

### Task 1.3: Authentication & Authorization

**What:** Implement NextAuth.js with OIDC provider support (Azure AD, Google, Okta), email/password fallback with bcrypt, TOTP-based MFA, JWT session management, and RBAC with permission checks.

**Design:**

```typescript
// apps/api/src/modules/auth/permissions.ts
export const PERMISSIONS = {
  CASH_POSITION_READ: 'cash_position:read',
  CASH_POSITION_WRITE: 'cash_position:write',
  PAYMENT_CREATE: 'payment:create',
  PAYMENT_APPROVE: 'payment:approve',
  PAYMENT_SUBMIT: 'payment:submit',
  FORECAST_READ: 'forecast:read',
  FORECAST_WRITE: 'forecast:write',
  FX_TRADE_CREATE: 'fx_trade:create',
  ADMIN: 'admin:*',
} as const;

export const DEFAULT_ROLES = {
  admin: Object.values(PERMISSIONS),
  treasurer: [
    PERMISSIONS.CASH_POSITION_READ,
    PERMISSIONS.CASH_POSITION_WRITE,
    PERMISSIONS.PAYMENT_CREATE,
    PERMISSIONS.PAYMENT_APPROVE,
    PERMISSIONS.PAYMENT_SUBMIT,
    PERMISSIONS.FORECAST_READ,
    PERMISSIONS.FORECAST_WRITE,
    PERMISSIONS.FX_TRADE_CREATE,
  ],
  analyst: [
    PERMISSIONS.CASH_POSITION_READ,
    PERMISSIONS.PAYMENT_CREATE,
    PERMISSIONS.FORECAST_READ,
  ],
  approver: [
    PERMISSIONS.CASH_POSITION_READ,
    PERMISSIONS.PAYMENT_APPROVE,
  ],
  viewer: [
    PERMISSIONS.CASH_POSITION_READ,
    PERMISSIONS.FORECAST_READ,
  ],
} as const;

// apps/api/src/modules/auth/guard.ts
export function requirePermission(permission: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const user = request.user;
    if (!user) return reply.status(401).send({ error: 'Unauthorized' });
    const hasPermission = user.permissions.includes(permission)
      || user.permissions.includes('admin:*');
    if (!hasPermission) return reply.status(403).send({ error: 'Forbidden' });
  };
}
```

**Testing:**
- OIDC login with mock provider completes successfully, creates app_user record
- Email/password login with correct credentials returns JWT
- Email/password login with wrong credentials returns 401
- TOTP enrollment generates valid QR code; verification with correct code succeeds
- TOTP verification with incorrect code returns 401
- JWT expiry: expired token returns 401
- RBAC: user with `analyst` role can GET /cash-positions but not POST /payments/approve
- RBAC: user with `treasurer` role can POST /payments/approve
- RBAC: entity-scoped role (approver for Entity A only) cannot approve payments for Entity B
- Session invalidation: revoked session returns 401 on next request

### Task 1.4: Legal Entity Management API

**What:** CRUD API for legal entities with hierarchy support, including entity tree retrieval via recursive CTE.

**Design:**

```typescript
// apps/api/src/modules/entity/routes.ts
import { FastifyPluginAsync } from 'fastify';
import { requirePermission } from '../auth/guard';

const entityRoutes: FastifyPluginAsync = async (app) => {
  // GET /entities - list all entities for tenant (flat)
  app.get('/', {
    preHandler: [requirePermission('cash_position:read')],
    schema: {
      querystring: {
        type: 'object',
        properties: {
          country_code: { type: 'string', minLength: 2, maxLength: 2 },
          entity_type: { type: 'string' },
          is_active: { type: 'boolean' },
        },
      },
    },
  }, async (request) => {
    return entityService.list(request.tenantId, request.query);
  });

  // GET /entities/tree - entity hierarchy tree
  app.get('/tree', {
    preHandler: [requirePermission('cash_position:read')],
  }, async (request) => {
    return entityService.getTree(request.tenantId);
  });

  // POST /entities - create entity
  app.post('/', {
    preHandler: [requirePermission('admin:*')],
    schema: {
      body: {
        type: 'object',
        required: ['name', 'country_code', 'functional_currency', 'entity_type'],
        properties: {
          name: { type: 'string' },
          parent_id: { type: 'string', format: 'uuid' },
          lei: { type: 'string', maxLength: 20 },
          country_code: { type: 'string', minLength: 2, maxLength: 2 },
          functional_currency: { type: 'string', minLength: 3, maxLength: 3 },
          entity_type: {
            type: 'string',
            enum: ['holding', 'operating', 'spv', 'branch', 'joint_venture'],
          },
          regulatory_data: { type: 'object' },
        },
      },
    },
  }, async (request) => {
    return entityService.create(request.tenantId, request.body);
  });
};
```

```sql
-- Entity tree query using recursive CTE
WITH RECURSIVE entity_tree AS (
    SELECT id, parent_id, name, country_code, functional_currency,
           entity_type, 0 AS depth, ARRAY[name] AS path
    FROM legal_entity
    WHERE tenant_id = $1 AND parent_id IS NULL AND is_active = true
    UNION ALL
    SELECT e.id, e.parent_id, e.name, e.country_code, e.functional_currency,
           e.entity_type, t.depth + 1, t.path || e.name
    FROM legal_entity e
    JOIN entity_tree t ON e.parent_id = t.id
    WHERE e.is_active = true
)
SELECT * FROM entity_tree ORDER BY path;
```

**Testing:**
- POST /entities creates entity and returns 201 with UUID
- POST /entities with invalid country_code returns 400
- POST /entities with duplicate LEI returns 409
- GET /entities returns all active entities for tenant
- GET /entities?country_code=DE filters correctly
- GET /entities/tree returns nested hierarchy with depth and path
- GET /entities/tree with 5-level hierarchy returns correct parent-child relationships
- PUT /entities/:id updates entity fields
- DELETE /entities/:id sets is_active=false (soft delete)
- Cross-tenant isolation: tenant B cannot see tenant A's entities

### Task 1.5: Audit Event Logging

**What:** Implement an append-only audit event system that captures all significant actions (entity creation, payment approval, balance updates) with actor, timestamp, and change details.

**Design:**

```typescript
// apps/api/src/modules/audit/service.ts
interface AuditEvent {
  tenantId: string;
  userId: string | null;
  action: string;           // e.g. 'entity.created', 'payment.approved'
  resourceType: string;     // e.g. 'legal_entity', 'payment_order'
  resourceId: string;
  changes: Record<string, unknown>;
  context: {
    ip?: string;
    userAgent?: string;
    source?: 'user' | 'system' | 'bank_api' | 'ai_agent';
  };
}

export class AuditService {
  async log(event: AuditEvent): Promise<void> {
    await this.db.query(
      `INSERT INTO audit_log (tenant_id, user_id, action, resource_type,
         resource_id, changes, context)
       VALUES ($1, $2, $3, $4, $5, $6, $7)`,
      [event.tenantId, event.userId, event.action, event.resourceType,
       event.resourceId, event.changes, event.context]
    );
  }

  async query(tenantId: string, filters: AuditQueryFilters): Promise<AuditEvent[]> {
    // Supports filtering by resource_type, resource_id, user_id, date range, action
  }
}
```

**Testing:**
- Audit log entry created on entity creation with correct action and changes
- Audit log entry created on entity update with before/after values in changes
- Audit log query by resource_type returns only matching entries
- Audit log query by date range filters correctly
- Audit log is append-only: UPDATE and DELETE operations on audit_log table fail (enforced by DB trigger or application)
- Audit log entries for one tenant not visible to another tenant (RLS)
- High-volume audit logging: 1000 events inserted in <2 seconds

### Definition of Done -- Phase 1

- [ ] All services start via `docker compose up` with zero manual configuration
- [ ] PostgreSQL schema matches Data Model Suggestion 3 for foundation tables
- [ ] 180 ISO 4217 currencies and 249 ISO 3166 countries seeded
- [ ] Authentication works with at least one OIDC provider and email/password
- [ ] MFA (TOTP) enrollment and verification functional
- [ ] RBAC enforced on all API endpoints with 5 default roles
- [ ] RLS enforced on all tenant-scoped tables
- [ ] Legal entity CRUD with hierarchy tree query working
- [ ] Audit logging captures all create/update/delete operations
- [ ] API documentation published as OpenAPI 3.1 spec
- [ ] Test coverage >80% for auth and entity modules
- [ ] CI pipeline: lint, type-check, test, build all pass

---

## Phase 2: Bank Connectivity & Cash Positioning

**Goal:** Connect to bank accounts via Open Banking APIs (Plaid/FDX for US, TrueLayer/PSD2 for EU) and CSV import. Aggregate balances into a real-time cash position dashboard.

**Duration:** 5--7 weeks

**Dependencies:** Phase 1 (database, auth, entities)

### Task 2.1: Bank Connection Management

**What:** CRUD for bank connections with support for FDX (Plaid), PSD2 (TrueLayer), and manual/CSV connection types. Implement OAuth consent flows for API-based connections. Store connection-specific configuration in JSONB.

**Design:**

```typescript
// apps/api/src/modules/bank-connection/service.ts
export class BankConnectionService {
  async initiateConnection(
    tenantId: string,
    params: {
      connectionType: 'fdx' | 'psd2' | 'cdr' | 'manual';
      bankName: string;
      countryCode: string;
      provider?: 'plaid' | 'truelayer' | 'direct';
    }
  ): Promise<{ connectionId: string; redirectUrl?: string }> {
    if (params.connectionType === 'fdx' && params.provider === 'plaid') {
      // Create Plaid Link token
      const linkToken = await this.plaidClient.linkTokenCreate({
        user: { client_user_id: tenantId },
        client_name: 'Treasury Management System',
        products: ['transactions', 'auth', 'balance'],
        country_codes: ['US'],
        language: 'en',
      });
      // Store pending connection
      const conn = await this.db.query(
        `INSERT INTO bank_connection (tenant_id, bank_name, country_code,
           connection_type, status, connection_config)
         VALUES ($1, $2, $3, 'fdx', 'pending', $4) RETURNING id`,
        [tenantId, params.bankName, params.countryCode,
         { provider: 'plaid', link_token: linkToken.link_token }]
      );
      return { connectionId: conn.rows[0].id, redirectUrl: linkToken.link_token };
    }
    // Similar for PSD2/TrueLayer...
  }

  async completeConnection(
    tenantId: string,
    connectionId: string,
    publicToken: string  // Plaid public_token from Link callback
  ): Promise<void> {
    // Exchange Plaid public token for access token
    const exchangeResponse = await this.plaidClient.itemPublicTokenExchange({
      public_token: publicToken,
    });
    // Update connection with access token (encrypted at rest)
    await this.db.query(
      `UPDATE bank_connection
       SET status = 'active',
           connection_config = connection_config || $1,
           updated_at = now()
       WHERE id = $2 AND tenant_id = $3`,
      [{ access_token: encrypt(exchangeResponse.access_token),
         item_id: exchangeResponse.item_id }, connectionId, tenantId]
    );
  }
}
```

**Testing:**
- POST /bank-connections with type=fdx returns Plaid Link token
- Complete connection callback exchanges token and stores encrypted access_token
- POST /bank-connections with type=manual creates connection with status 'active'
- GET /bank-connections lists all connections for tenant with status
- Connection with expired consent shows status='expired'
- Access tokens are encrypted at rest: raw token not visible in database query
- Plaid webhook handler updates connection status on errors
- PSD2 consent expiry date tracked; alert generated 7 days before expiry

### Task 2.2: Bank Account Discovery & Registration

**What:** After a bank connection is established, discover and register bank accounts from the connected bank. Map Plaid/TrueLayer account data to the bank_account table. Support manual account registration for CSV connections.

**Design:**

```typescript
// apps/api/src/modules/bank-account/sync.ts
export class BankAccountSyncService {
  async discoverAccounts(
    tenantId: string,
    connectionId: string
  ): Promise<DiscoveredAccount[]> {
    const connection = await this.getConnection(tenantId, connectionId);
    if (connection.connection_type === 'fdx') {
      const accounts = await this.plaidClient.accountsGet({
        access_token: decrypt(connection.connection_config.access_token),
      });
      return accounts.accounts.map(acct => ({
        accountName: acct.name,
        currency: acct.balances.iso_currency_code || 'USD',
        accountType: this.mapPlaidType(acct.type, acct.subtype),
        iban: null,
        accountIdentifiers: {
          plaid_account_id: acct.account_id,
          account_number_masked: acct.mask ? `****${acct.mask}` : null,
          fdx_account_id: acct.account_id,
        },
        providerData: {
          plaid_type: acct.type,
          plaid_subtype: acct.subtype,
          institution_name: connection.bank_name,
        },
        currentBalance: acct.balances.current,
      }));
    }
    // Similar for PSD2...
  }

  async registerAccount(
    tenantId: string,
    legalEntityId: string,
    connectionId: string,
    account: DiscoveredAccount
  ): Promise<string> {
    const result = await this.db.query(
      `INSERT INTO bank_account (tenant_id, legal_entity_id,
         bank_connection_id, account_name, currency, account_type,
         iban, account_identifiers, provider_data)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9) RETURNING id`,
      [tenantId, legalEntityId, connectionId, account.accountName,
       account.currency, account.accountType, account.iban,
       account.accountIdentifiers, account.providerData]
    );
    return result.rows[0].id;
  }
}
```

**Testing:**
- Account discovery from Plaid sandbox returns at least 1 account
- Discovered account maps Plaid type/subtype to TMS account_type correctly
- Registered account has correct currency, identifiers, and provider_data
- Manual account registration accepts IBAN, routing number, sort code
- IBAN uniqueness enforced: duplicate IBAN returns 409
- Account linked to correct legal entity
- Account deactivation (soft delete) prevents it from appearing in balance sync

### Task 2.3: Balance Synchronization

**What:** Scheduled job (BullMQ) polls bank APIs for latest balances every 15 minutes. Stores balance snapshots in bank_account_balance table. Supports both intraday and end-of-day balance types.

**Design:**

```typescript
// apps/api/src/modules/bank-account/balance-sync.worker.ts
import { Worker, Queue } from 'bullmq';

const balanceSyncQueue = new Queue('balance-sync', { connection: redis });

// Schedule: every 15 minutes for active connections
balanceSyncQueue.add('sync-all', {}, {
  repeat: { pattern: '*/15 * * * *' },
});

const worker = new Worker('balance-sync', async (job) => {
  const connections = await db.query(
    `SELECT bc.id, bc.connection_type, bc.connection_config, bc.tenant_id
     FROM bank_connection bc
     WHERE bc.status = 'active'`
  );

  for (const conn of connections.rows) {
    try {
      if (conn.connection_type === 'fdx') {
        const accounts = await plaidClient.accountsGet({
          access_token: decrypt(conn.connection_config.access_token),
        });
        for (const acct of accounts.accounts) {
          await db.query(
            `INSERT INTO bank_account_balance
               (tenant_id, bank_account_id, balance_date, balance_type,
                amount, currency, source, source_data)
             VALUES ($1,
               (SELECT id FROM bank_account
                WHERE account_identifiers @> $2 AND tenant_id = $1),
               CURRENT_DATE, 'closing_available',
               $3, $4, 'bank_api', $5)
             ON CONFLICT (bank_account_id, balance_date, balance_type)
             DO UPDATE SET amount = $3, captured_at = now()`,
            [conn.tenant_id,
             JSON.stringify({ plaid_account_id: acct.account_id }),
             acct.balances.current,
             acct.balances.iso_currency_code || 'USD',
             { plaid_balance_type: 'current', available: acct.balances.available }]
          );
        }
      }
    } catch (error) {
      await db.query(
        `UPDATE bank_connection SET status = 'error',
           connection_config = connection_config || $1, updated_at = now()
         WHERE id = $2`,
        [{ last_error: error.message }, conn.id]
      );
    }
  }
}, { connection: redis });
```

**Testing:**
- Balance sync job runs on 15-minute schedule
- Plaid sandbox balance sync stores correct amount and currency
- Duplicate balance for same account/date/type is upserted (not duplicated)
- Connection error sets connection status to 'error' with error message
- Inactive connections are skipped
- Balance sync creates audit_log entry for each balance update
- Balance history: 30 days of balances queryable for a single account
- Performance: syncing 50 accounts completes in <30 seconds

### Task 2.4: Cash Position Aggregation

**What:** Aggregate bank account balances into cash positions per entity per currency per date. Convert to base currency using latest FX rates. Provide the API endpoint that powers the cash position dashboard.

**Design:**

```typescript
// apps/api/src/modules/cash-position/service.ts
export class CashPositionService {
  async calculate(tenantId: string, positionDate: Date): Promise<CashPosition[]> {
    const result = await this.db.query(`
      WITH latest_balances AS (
        SELECT DISTINCT ON (bank_account_id)
          ba.legal_entity_id,
          bab.bank_account_id,
          bab.amount,
          bab.currency,
          bab.balance_date
        FROM bank_account_balance bab
        JOIN bank_account ba ON ba.id = bab.bank_account_id
        WHERE bab.tenant_id = $1
          AND bab.balance_date <= $2
          AND bab.balance_type = 'closing_available'
        ORDER BY bank_account_id, balance_date DESC
      )
      SELECT
        le.id AS legal_entity_id,
        le.name AS entity_name,
        lb.currency,
        SUM(lb.amount) AS total_balance,
        COUNT(*) AS account_count
      FROM latest_balances lb
      JOIN legal_entity le ON le.id = lb.legal_entity_id
      GROUP BY le.id, le.name, lb.currency
      ORDER BY le.name, lb.currency
    `, [tenantId, positionDate]);

    // Convert to base currency
    const tenant = await this.getTenant(tenantId);
    for (const row of result.rows) {
      if (row.currency !== tenant.base_currency) {
        const rate = await this.fxRateService.getRate(
          row.currency, tenant.base_currency, positionDate
        );
        row.base_currency_amount = row.total_balance * rate;
        row.fx_rate_used = rate;
      } else {
        row.base_currency_amount = row.total_balance;
        row.fx_rate_used = 1;
      }
    }

    // Upsert into cash_position table
    for (const pos of result.rows) {
      await this.db.query(`
        INSERT INTO cash_position (tenant_id, legal_entity_id, position_date,
          currency, total_balance, account_count, base_currency_amount,
          fx_rate_used, breakdown)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        ON CONFLICT (tenant_id, legal_entity_id, position_date, currency)
        DO UPDATE SET total_balance = $5, account_count = $6,
          base_currency_amount = $7, fx_rate_used = $8, breakdown = $9,
          updated_at = now()
      `, [tenantId, pos.legal_entity_id, positionDate, pos.currency,
          pos.total_balance, pos.account_count, pos.base_currency_amount,
          pos.fx_rate_used, pos.breakdown]);
    }

    return result.rows;
  }

  async getDashboardData(tenantId: string): Promise<DashboardData> {
    // Returns: positions by entity, positions by currency,
    // total in base currency, trend (last 30 days)
  }
}
```

**Testing:**
- Cash position calculated correctly for single entity, single currency
- Cash position calculated correctly for single entity, multiple currencies
- Cash position calculated correctly for multiple entities
- Base currency conversion uses latest available FX rate
- Dashboard endpoint returns aggregated totals matching sum of positions
- Position recalculation after new balance update reflects new amounts
- Position for past date uses balances as-of that date (not latest)
- Performance: position calculation for 10 entities, 50 accounts completes in <5 seconds

### Task 2.5: CSV Bank Statement Import

**What:** For banks without API connectivity, provide CSV upload for bank statement import. Parse CSV, create bank_statement and bank_transaction records, update account balances.

**Design:**

```typescript
// apps/api/src/modules/bank-statement/csv-parser.ts
export class CsvStatementParser {
  async parse(
    tenantId: string,
    bankAccountId: string,
    csvContent: string,
    formatConfig: CsvFormatConfig
  ): Promise<ParsedStatement> {
    const rows = parseCSV(csvContent, {
      dateColumn: formatConfig.dateColumn,
      amountColumn: formatConfig.amountColumn,
      descriptionColumn: formatConfig.descriptionColumn,
      dateFormat: formatConfig.dateFormat,      // e.g. 'YYYY-MM-DD', 'DD/MM/YYYY'
      decimalSeparator: formatConfig.decimalSeparator, // '.' or ','
      creditDebitMode: formatConfig.creditDebitMode,   // 'signed' or 'separate_columns'
    });

    const transactions = rows.map(row => ({
      amount: Math.abs(row.amount),
      creditDebit: row.amount >= 0 ? 'C' : 'D',
      currency: row.currency || formatConfig.defaultCurrency,
      valueDate: row.date,
      bookingDate: row.date,
      counterpartyName: row.counterparty || null,
      remittanceInfo: row.description,
      reconciliationStatus: 'unmatched',
      transactionData: { raw_csv_row: row.rawIndex },
    }));

    return {
      statementDate: transactions[transactions.length - 1]?.valueDate || new Date(),
      openingBalance: formatConfig.openingBalance,
      closingBalance: formatConfig.closingBalance || this.calculateClosing(
        formatConfig.openingBalance, transactions
      ),
      entryCount: transactions.length,
      transactions,
    };
  }
}
```

**Testing:**
- CSV upload with US date format (MM/DD/YYYY) parses correctly
- CSV upload with EU date format (DD/MM/YYYY) parses correctly
- CSV with signed amounts (negative = debit) maps credit_debit correctly
- CSV with separate credit/debit columns maps correctly
- CSV with comma decimal separator (EU format) parses amounts correctly
- Statement and transactions inserted into database
- Balance updated after statement import
- Duplicate CSV upload (same transactions) detects duplicates and skips
- Invalid CSV (missing required columns) returns 400 with descriptive error
- Large CSV (10,000 rows) processes in <30 seconds

### Task 2.6: Cash Position Dashboard (Frontend)

**What:** Build the primary treasury dashboard in Next.js: consolidated cash position view with entity drill-down, currency breakdown, and 30-day trend chart.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/cash-position/page.tsx
export default async function CashPositionPage() {
  return (
    <div className="grid gap-6">
      {/* Global cash position summary card */}
      <CashPositionSummary />

      <div className="grid grid-cols-2 gap-6">
        {/* Position by currency - donut chart */}
        <CurrencyBreakdownChart />
        {/* Position by entity - bar chart */}
        <EntityPositionChart />
      </div>

      {/* 30-day cash position trend - line chart */}
      <CashTrendChart days={30} />

      {/* Detailed position table with entity/currency breakdown */}
      <CashPositionTable />
    </div>
  );
}

// apps/web/src/components/treasury/CashPositionSummary.tsx
// Shows: total cash in base currency, # of accounts, # of entities,
//        last sync time, day-over-day change
```

**Testing:**
- Dashboard loads and displays total cash in base currency
- Currency breakdown chart shows correct proportions
- Entity breakdown shows correct per-entity totals
- 30-day trend chart renders with correct data points
- Drill-down to entity shows individual bank account balances
- Dashboard auto-refreshes when new balance sync completes
- Empty state (no accounts connected) shows onboarding prompt
- Mobile responsive: dashboard readable on tablet-width screens

### Definition of Done -- Phase 2

- [ ] At least one bank connection type (Plaid sandbox) working end-to-end
- [ ] Bank accounts discovered and registered from connected banks
- [ ] Balances synced automatically every 15 minutes
- [ ] Cash position aggregated per entity per currency
- [ ] CSV bank statement import working with at least 3 date formats
- [ ] Cash position dashboard displaying real-time data
- [ ] 30-day trend chart rendering correctly
- [ ] Base currency conversion working for multi-currency positions
- [ ] All bank credentials encrypted at rest
- [ ] Test coverage >80% for bank-connection and cash-position modules

---

## Phase 3: Payments & Reconciliation

**Goal:** Payment order creation with dual-approval workflow, bank submission via Open Banking APIs, and automated bank-to-GL reconciliation.

**Duration:** 5--7 weeks

**Dependencies:** Phase 2 (bank accounts, balances, statement import)

### Task 3.1: Counterparty Management

**What:** CRUD for counterparties (vendors, customers, intercompany entities) with bank account details stored in JSONB. Counterparty verification workflow.

**Design:**

```typescript
// apps/api/src/modules/payment/counterparty.service.ts
// CRUD following Data Model Suggestion 3's counterparty table
// Bank accounts stored as JSONB array on counterparty record
// Verification workflow: unverified -> pending_review -> verified
// Penny test verification via Plaid Auth for US accounts
```

**Testing:**
- Create counterparty with EUR and USD bank accounts
- Retrieve counterparty with bank accounts in response
- Update counterparty bank account details
- Search counterparties by name (partial match)
- Filter counterparties by verified EUR accounts
- Counterparty deactivation prevents use in new payments
- Duplicate IBAN across counterparties allowed (different entities may pay same account)

### Task 3.2: Payment Order Lifecycle

**What:** Payment order creation, dual-approval workflow, status tracking with history in JSONB. Status lifecycle: draft -> pending_approval -> approved -> submitted -> accepted/rejected -> settled/returned.

**Design:**

```typescript
// apps/api/src/modules/payment/service.ts
export class PaymentService {
  async create(tenantId: string, userId: string, params: CreatePaymentParams) {
    // Validate: debit account belongs to tenant, has sufficient balance
    // Validate: counterparty exists and is active
    // Validate: currency matches debit account or is cross-currency
    // Create payment_order with status 'draft'
    // Log audit event
  }

  async submitForApproval(tenantId: string, paymentId: string, userId: string) {
    // Change status to 'pending_approval'
    // Determine required approval count from tenant settings
    // Append to status_history JSONB array
    // Notify approvers via alert system
  }

  async approve(tenantId: string, paymentId: string, approverId: string, comment?: string) {
    // Verify approver has 'payment:approve' permission
    // Verify approver is not the payment creator (four-eyes principle)
    // Increment approval_count
    // If approval_count >= required_approvals, change status to 'approved'
    // Append approval to status_history
    // Log audit event
  }

  async submitToBank(tenantId: string, paymentId: string) {
    // Verify status is 'approved'
    // Determine payment channel (Open Banking, SWIFT, ACH)
    // Submit via appropriate bank API
    // Change status to 'submitted'
    // Store bank reference in payment_details
    // Log audit event
  }
}
```

**Testing:**
- Payment creation with valid data returns 201
- Payment creation with insufficient balance returns 400
- Payment creation with inactive counterparty returns 400
- Submit for approval changes status and notifies approvers
- Approval by payment creator rejected (four-eyes principle)
- First approval with 2-required: status remains pending_approval
- Second approval: status changes to approved
- Rejection resets status to draft with rejection reason
- Bank submission changes status to submitted with bank reference
- Status history JSONB contains complete lifecycle
- Audit log contains entries for every status transition
- Concurrent approval race condition handled (optimistic locking)

### Task 3.3: Payment Submission via Open Banking

**What:** Submit approved payments to banks via Open Banking payment initiation APIs (Plaid Transfer for US, TrueLayer Payments for EU/UK).

**Design:**

```typescript
// apps/api/src/modules/payment/bank-submission.ts
export class PaymentBankSubmissionService {
  async submit(tenantId: string, payment: PaymentOrder): Promise<SubmissionResult> {
    const debitAccount = await this.getAccount(payment.debit_account_id);
    const connection = await this.getConnection(debitAccount.bank_connection_id);

    switch (connection.connection_type) {
      case 'fdx':
        return this.submitViaPlaid(payment, connection);
      case 'psd2':
        return this.submitViaTrueLayer(payment, connection);
      default:
        return this.generatePaymentFile(payment); // ISO 20022 pain.001 XML
    }
  }

  private async generatePaymentFile(payment: PaymentOrder): Promise<SubmissionResult> {
    // Generate pain.001 XML for manual bank upload
    // Store file reference for download
    return { method: 'file', fileUrl: '/api/v1/payments/{id}/pain001.xml' };
  }
}
```

**Testing:**
- US payment via Plaid Transfer sandbox succeeds with transfer ID
- EU SEPA payment via TrueLayer sandbox succeeds
- Payment to non-API bank generates pain.001 XML file
- Generated pain.001 XML validates against ISO 20022 schema
- Payment status updates received from bank via webhook
- Failed bank submission rolls payment status back to 'approved'
- Payment reference stored in payment_details JSONB

### Task 3.4: Bank Reconciliation Engine

**What:** Automated matching of bank statement entries (from Task 2.5) to GL transactions (imported from ERP) and payment orders. Support for exact match, fuzzy match, and manual matching.

**Design:**

```typescript
// apps/api/src/modules/reconciliation/engine.ts
export class ReconciliationEngine {
  async autoReconcile(tenantId: string, bankAccountId: string): Promise<ReconciliationResult> {
    // Step 1: Exact match on amount + date + reference
    const exactMatches = await this.db.query(`
      UPDATE bank_transaction bt
      SET reconciliation_status = 'matched',
          matched_payment_id = po.id
      FROM payment_order po
      WHERE bt.tenant_id = $1
        AND bt.bank_account_id = $2
        AND bt.reconciliation_status = 'unmatched'
        AND ABS(bt.amount - po.amount) < 0.01
        AND bt.value_date = po.value_date
        AND bt.remittance_info LIKE '%' || po.end_to_end_id || '%'
      RETURNING bt.id
    `, [tenantId, bankAccountId]);

    // Step 2: Fuzzy match on amount + date range (+-2 days)
    const fuzzyMatches = await this.findFuzzyMatches(tenantId, bankAccountId);

    // Step 3: Return unmatched for manual review
    const unmatched = await this.getUnmatched(tenantId, bankAccountId);

    return {
      exactMatches: exactMatches.rowCount,
      fuzzyMatches: fuzzyMatches.length,
      unmatched: unmatched.length,
    };
  }
}
```

**Testing:**
- Exact match: transaction with matching amount, date, and reference auto-reconciles
- Fuzzy match: transaction with matching amount but date +-1 day suggested as match
- No match: transaction with no corresponding payment remains unmatched
- Manual match: user can manually match unmatched transaction to GL entry
- Manual exclude: user can mark transaction as excluded from reconciliation
- Reconciliation runs after statement import automatically
- Reconciliation statistics: matched count, unmatched count, match rate
- Undo reconciliation: matched transaction can be unmatched

### Task 3.5: ERP Integration (GL Transaction Import)

**What:** Import general ledger transactions from ERP systems (initial: CSV import, with QuickBooks and Xero API connectors planned) for reconciliation.

**Design:**

```typescript
// apps/api/src/modules/reconciliation/erp-connection.service.ts
// ERP connection CRUD following Data Model Suggestion 3's erp_connection concept
// CSV import for GL transactions
// GL transaction records stored for reconciliation matching
```

**Testing:**
- ERP connection created with type 'csv'
- GL transactions imported from CSV with correct mapping
- GL transactions available for reconciliation matching
- Duplicate GL transaction import detected and skipped
- GL transaction linked to correct legal entity

### Definition of Done -- Phase 3

- [ ] Counterparty CRUD with bank account management working
- [ ] Payment order creation, approval workflow (2-level), and status tracking
- [ ] Four-eyes principle enforced (creator cannot approve)
- [ ] Payment submission via at least one bank API (Plaid Transfer sandbox)
- [ ] pain.001 XML generation for manual bank submission
- [ ] Automated reconciliation matching (exact + fuzzy)
- [ ] Manual reconciliation interface working
- [ ] ERP GL transaction import via CSV
- [ ] Full audit trail for payment lifecycle
- [ ] Test coverage >80% for payment and reconciliation modules

---

## Phase 4: Cash Flow Forecasting (ML)

**Goal:** ML-powered cash flow forecasting with random forest and LSTM models, rolling 13-week and 12-month projections, forecast vs. actuals comparison, and scenario planning.

**Duration:** 6--8 weeks

**Dependencies:** Phase 2 (bank transactions provide training data)

### Task 4.1: Transaction Categorization (ML)

**What:** Classify bank transactions into cash flow categories (receipts, payroll, rent, taxes, etc.) using a trained ML model. Human-in-the-loop for low-confidence classifications.

**Design:**

```python
# apps/ml/src/models/categorization/model.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.feature_extraction.text import TfidfVectorizer
import numpy as np

class TransactionCategorizer:
    def __init__(self):
        self.vectorizer = TfidfVectorizer(max_features=5000)
        self.classifier = RandomForestClassifier(n_estimators=100)

    def train(self, transactions: list[dict]):
        """Train on manually categorized transactions."""
        texts = [t['remittance_info'] + ' ' + (t['counterparty_name'] or '')
                 for t in transactions]
        labels = [t['category'] for t in transactions]
        X = self.vectorizer.fit_transform(texts)
        self.classifier.fit(X, labels)

    def predict(self, transaction: dict) -> tuple[str, float]:
        """Return (category, confidence)."""
        text = transaction['remittance_info'] + ' ' + (transaction.get('counterparty_name') or '')
        X = self.vectorizer.transform([text])
        proba = self.classifier.predict_proba(X)[0]
        category = self.classifier.classes_[np.argmax(proba)]
        confidence = float(np.max(proba))
        return category, confidence
```

**Testing:**
- Model trains on 500+ labeled transactions without error
- Prediction returns category and confidence for new transaction
- High-confidence predictions (>0.85) match expected category >80% of time
- Low-confidence predictions (<0.5) flagged for human review
- Retraining with additional labeled data improves accuracy
- API endpoint: POST /ml/categorize returns category + confidence
- Batch categorization of 1000 transactions completes in <10 seconds

### Task 4.2: Cash Flow Forecast Model (Random Forest)

**What:** Train a random forest model on historical categorized cash flows to produce rolling 13-week and 12-month forecasts with confidence intervals.

**Design:**

```python
# apps/ml/src/models/cash_forecast/random_forest.py
from sklearn.ensemble import RandomForestRegressor
import pandas as pd
import numpy as np

class CashFlowForecaster:
    def __init__(self):
        self.model = RandomForestRegressor(
            n_estimators=200,
            max_depth=15,
            min_samples_leaf=5,
            random_state=42,
        )

    def prepare_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """Engineer features from historical cash flow data."""
        df['day_of_week'] = df['date'].dt.dayofweek
        df['month'] = df['date'].dt.month
        df['week_of_year'] = df['date'].dt.isocalendar().week
        df['is_month_end'] = df['date'].dt.is_month_end.astype(int)
        df['is_quarter_end'] = df['date'].dt.is_quarter_end.astype(int)
        # Lag features
        for lag in [7, 14, 28, 91]:
            df[f'amount_lag_{lag}'] = df.groupby('category')['amount'].shift(lag)
        # Rolling statistics
        for window in [7, 30, 90]:
            df[f'amount_rolling_mean_{window}'] = (
                df.groupby('category')['amount']
                .transform(lambda x: x.rolling(window).mean())
            )
        return df

    def forecast(self, category: str, horizon_weeks: int = 13) -> list[ForecastLine]:
        """Generate weekly forecast with 95% confidence intervals."""
        # Use individual tree predictions for confidence intervals
        predictions = np.array([
            tree.predict(X_future) for tree in self.model.estimators_
        ])
        mean_pred = predictions.mean(axis=0)
        lower = np.percentile(predictions, 2.5, axis=0)
        upper = np.percentile(predictions, 97.5, axis=0)

        return [
            ForecastLine(
                period_start=week_start,
                period_end=week_end,
                category=category,
                projected=float(mean_pred[i]),
                confidence_lower=float(lower[i]),
                confidence_upper=float(upper[i]),
            )
            for i, (week_start, week_end) in enumerate(future_weeks)
        ]
```

**Testing:**
- Model trains on 12 months of historical data (minimum 365 data points)
- 13-week forecast produces 13 line items per category
- 12-month forecast produces 12 line items per category
- Confidence intervals widen for more distant periods
- MAPE (Mean Absolute Percentage Error) < 25% on held-out test data
- Forecast accuracy metrics stored in model_config JSONB
- Forecast reproducibility: same input data produces same forecast (deterministic seed)
- Feature importance ranking available (explainability)

### Task 4.3: LSTM Forecast Model

**What:** Implement LSTM (Long Short-Term Memory) neural network as an alternative forecast model for capturing non-linear temporal patterns in cash flows.

**Design:**

```python
# apps/ml/src/models/cash_forecast/lstm.py
import torch
import torch.nn as nn

class CashFlowLSTM(nn.Module):
    def __init__(self, input_size=10, hidden_size=64, num_layers=2, output_size=13):
        super().__init__()
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers,
                           batch_first=True, dropout=0.2)
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x):
        lstm_out, _ = self.lstm(x)
        return self.fc(lstm_out[:, -1, :])  # Use last timestep output
```

**Testing:**
- LSTM trains on 12 months of data; loss converges within 100 epochs
- LSTM forecast produces 13-week projections
- LSTM MAPE comparable to or better than random forest on test data
- Ensemble of RF + LSTM produces lower error than either alone
- Model checkpoint saved and loadable for inference without retraining
- GPU training works when CUDA available; CPU fallback functional

### Task 4.4: Forecast API & Management

**What:** API endpoints for generating, viewing, and comparing forecasts. Forecast vs. actuals comparison with variance analysis. Scenario planning (stress tests).

**Design:**

```typescript
// apps/api/src/modules/forecast/routes.ts
// POST /forecasts/generate - trigger forecast generation
// GET /forecasts - list forecasts for tenant
// GET /forecasts/:id - get forecast with line items
// GET /forecasts/:id/accuracy - get accuracy metrics (actuals vs projected)
// POST /forecasts/:id/scenarios - add stress test scenario
// GET /forecasts/compare?ids=a,b - compare two forecasts side by side
```

**Testing:**
- Generate forecast triggers ML sidecar and stores result
- Forecast stored in database with line_items JSONB
- Forecast accuracy endpoint fills in actuals when available
- Variance calculated correctly (actual - projected)
- Scenario planning: FX shock of -10% adjusts affected line items
- Multiple forecasts compared side by side with variance highlighted
- Published forecast becomes the active forecast; supersedes previous
- Forecast deletion requires admin permission

### Task 4.5: Forecast Dashboard (Frontend)

**What:** Dashboard showing forecast projections with actuals overlay, confidence intervals, accuracy metrics, and scenario comparison.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/forecasts/page.tsx
// - Forecast selector (active vs historical)
// - Line chart: projected vs actuals with confidence band
// - Category breakdown (stacked bar)
// - Accuracy metrics card (MAPE, RMSE, R2)
// - Scenario comparison toggle
```

**Testing:**
- Forecast chart renders with correct projected amounts
- Actuals overlay appears when data is available
- Confidence interval band visible on chart
- Category breakdown shows correct per-category projections
- Accuracy metrics display correctly
- Switching between forecasts updates chart
- Empty state: no forecast available shows generation prompt

### Definition of Done -- Phase 4

- [ ] Transaction categorization model trained and serving predictions
- [ ] Random forest forecast model producing 13-week and 12-month projections
- [ ] LSTM model trained and producing comparable forecasts
- [ ] Forecast confidence intervals calculated from model
- [ ] MAPE < 25% on test data (matching MDPI 2025 baseline)
- [ ] Forecast vs. actuals comparison with variance analysis
- [ ] Scenario planning: at least 3 scenario types (base, downside, FX shock)
- [ ] Forecast dashboard rendering projections with actuals overlay
- [ ] ML model retraining triggered manually or on schedule
- [ ] Model accuracy metrics persisted and queryable
- [ ] Test coverage >80% for forecast module

---

## Phase 5: FX Risk & Exposure Management

**Goal:** FX rate ingestion, exposure calculation per entity per currency, FX trade management (forwards, spots), and real-time exposure monitoring with alerts.

**Duration:** 4--6 weeks

**Dependencies:** Phase 2 (cash positions), Phase 3 (payment data)

### Task 5.1: FX Rate Ingestion

**What:** Scheduled ingestion of FX rates from free API sources (ECB, exchange rate APIs). Support for spot, forward, and internal/budget rates. FX rate history for historical position conversion.

**Design:**

```typescript
// apps/api/src/modules/fx-rate/service.ts
// BullMQ job: fetch ECB rates daily at 16:00 CET
// Store spot rates for all major currency pairs
// API: GET /fx-rates?base=USD&date=2026-05-25
// API: POST /fx-rates (manual/internal rate entry)
// History: GET /fx-rates/history?pair=EURUSD&from=2026-01-01&to=2026-05-25
```

**Testing:**
- ECB rate fetch returns rates for 30+ currency pairs
- Rates stored with correct date, source, and rate_type
- Rate query returns latest rate for a currency pair
- Historical rate query returns time series
- Manual rate entry (budget rates) stored correctly
- Missing rate for a pair returns 404 with helpful message
- Rate ingestion job idempotent: re-run does not create duplicates

### Task 5.2: FX Exposure Calculation

**What:** Calculate net FX exposure per entity per currency by aggregating bank balances, outstanding receivables, payables, and forecasted cash flows in each currency.

**Design:**

```typescript
// apps/api/src/modules/fx-exposure/service.ts
export class FxExposureService {
  async calculate(tenantId: string, entityId: string, date: Date): Promise<FxExposure[]> {
    // For each non-functional currency held by the entity:
    // 1. Balance sheet exposure: bank account balances in that currency
    // 2. Committed exposure: outstanding payment orders in that currency
    // 3. Forecast exposure: projected cash flows from forecast
    // Net exposure = sum of all sources
    // Hedged amount = sum of open FX trades hedging this exposure
    // Net unhedged = net exposure - hedged amount
  }
}
```

**Testing:**
- Exposure calculated correctly for entity with USD accounts and EUR balances
- Hedged amount deducted from gross exposure
- Hedge ratio calculated correctly (hedged / gross)
- Exposure across all entities aggregated at tenant level
- Exposure for functional currency is zero (no FX risk)
- Alert triggered when exposure exceeds configured threshold

### Task 5.3: FX Trade Management

**What:** CRUD for FX trades (spot, forward, NDF, swap). Link trades to exposures for hedge coverage tracking. Mark-to-market calculation.

**Design:**

```typescript
// apps/api/src/modules/fx-trade/service.ts
// POST /fx-trades - create trade with ISDA reference
// GET /fx-trades - list open trades
// GET /fx-trades/:id - trade details with MTM
// POST /fx-trades/:id/settle - mark trade as settled
// POST /fx-trades/:id/mtm - recalculate mark-to-market
```

**Testing:**
- Create FX forward with buy/sell currencies, rate, and settlement date
- Trade linked to FX exposure updates hedged_amount
- Mark-to-market calculated using current vs. inception spot rate
- Settled trade changes status and records settlement details
- Open trades list excludes settled and cancelled trades
- ISDA agreement reference stored in instrument_data JSONB

### Task 5.4: FX Exposure Dashboard (Frontend)

**What:** Dashboard showing net FX exposure by currency, hedge coverage ratios, open FX trades, and exposure trend.

**Testing:**
- Exposure table shows net amount per currency with hedge ratio
- Color coding: unhedged > 30% of gross shown in red
- FX trade list shows open trades with MTM values
- Exposure trend chart shows 30-day history

### Definition of Done -- Phase 5

- [ ] FX rates ingested daily from ECB or equivalent source
- [ ] FX exposure calculated per entity per currency
- [ ] FX trades (spot, forward) CRUD functional
- [ ] Hedge coverage tracked and displayed
- [ ] Mark-to-market calculation working
- [ ] Exposure threshold alerts configurable and firing
- [ ] FX exposure dashboard rendering correctly
- [ ] Test coverage >80% for FX modules

---

## Phase 6: Debt, Covenants & Alert System

**Goal:** Debt instrument tracking, covenant compliance monitoring with automated testing, and a configurable alert/notification system.

**Duration:** 4--5 weeks

**Dependencies:** Phase 2 (cash positions for covenant calculations)

### Task 6.1: Debt Instrument Management

**What:** CRUD for debt instruments (term loans, revolving credit, bonds) with terms stored in JSONB. Track outstanding amounts, interest rates, maturity dates.

**Design:**

```typescript
// Following Data Model Suggestion 3's debt_instrument table
// Covenants embedded in terms JSONB
// Covenant test results stored in covenant_tests JSONB array
// API: POST /debt-instruments, GET /debt-instruments, GET /debt-instruments/:id
```

**Testing:**
- Create term loan with fixed rate, maturity date, and 2 covenants
- Create revolving credit with floating rate (SOFR + spread)
- Outstanding amount tracked separately from principal
- Covenant definitions stored in terms JSONB with correct thresholds
- Maturity date approaching triggers alert (configurable: 30/60/90 days)

### Task 6.2: Covenant Compliance Monitoring

**What:** Automated covenant compliance testing based on configured frequencies (monthly, quarterly). Calculate actual financial ratios from cash position and debt data. Alert on approaching or breached covenants.

**Design:**

```typescript
// apps/api/src/modules/debt/covenant-monitor.ts
export class CovenantMonitor {
  async testCovenant(tenantId: string, debtId: string, covenantId: string): Promise<CovenantTestResult> {
    const covenant = await this.getCovenant(debtId, covenantId);
    const actualValue = await this.calculateRatio(tenantId, covenant.type);
    const isCompliant = this.checkCompliance(actualValue, covenant);
    const headroomPct = this.calculateHeadroom(actualValue, covenant);

    // Append test result to covenant_tests JSONB
    // If headroom < 20%, trigger warning alert
    // If not compliant, trigger critical alert
    return { actualValue, isCompliant, headroomPct };
  }
}
```

**Testing:**
- Leverage ratio calculated correctly from outstanding debt / EBITDA proxy
- Minimum liquidity covenant tested against current cash position
- Compliant covenant: headroom percentage calculated correctly
- Near-breach (headroom < 20%): warning alert generated
- Actual breach: critical alert generated with cure period details
- Test result appended to covenant_tests JSONB array
- Scheduled testing runs at configured frequency (monthly/quarterly)

### Task 6.3: Alert & Notification System

**What:** Configurable alert rules with multiple channels (email, Slack webhook, in-app). Alert types: low balance, high balance, FX threshold, covenant warning, payment failure, forecast variance, anomaly detection.

**Design:**

```typescript
// apps/api/src/modules/alert/service.ts
// Alert rule CRUD with conditions in JSONB
// Alert evaluation engine runs on BullMQ schedule
// Notification dispatch: email via SendGrid, Slack via webhook, in-app via WebSocket
// Alert acknowledgement and history tracking
```

**Testing:**
- Low balance alert fires when account drops below threshold
- FX threshold alert fires when rate moves more than configured %
- Covenant warning alert fires when headroom < 20%
- Email notification sent with correct content
- Slack webhook fires with formatted message
- In-app notification appears in notification panel
- Alert acknowledged and does not re-fire within cooldown period
- Alert history queryable by type, date range, severity

### Definition of Done -- Phase 6

- [ ] Debt instruments CRUD with covenant definitions
- [ ] Automated covenant compliance testing at configured frequency
- [ ] Covenant headroom and breach detection working
- [ ] Alert system with at least 3 alert types and 2 channels (email + in-app)
- [ ] Alert acknowledgement and cooldown functional
- [ ] Covenant compliance dashboard showing status per instrument
- [ ] Test coverage >80% for debt and alert modules

---

## Phase 7: AI/LLM Integration & MCP Server

**Goal:** Natural language treasury queries via LLM, autonomous FX monitoring agent, and MCP server for AI assistant integration.

**Duration:** 4--6 weeks

**Dependencies:** Phase 2 (cash positions), Phase 4 (forecasts), Phase 5 (FX data)

### Task 7.1: MCP Server Implementation

**What:** Expose treasury data and operations as an MCP server, enabling any MCP-compatible AI assistant to query cash positions, retrieve forecasts, monitor FX exposures, and initiate payment approvals.

**Design:**

```typescript
// apps/api/src/modules/mcp/server.ts
import { McpServer } from '@modelcontextprotocol/sdk/server';

const mcpServer = new McpServer({
  name: 'treasury-mcp',
  version: '1.0.0',
});

// Tools
mcpServer.tool('get_cash_position', {
  description: 'Get current cash position across all entities and currencies',
  inputSchema: {
    type: 'object',
    properties: {
      entity_name: { type: 'string', description: 'Filter by entity name (optional)' },
      currency: { type: 'string', description: 'Filter by currency code (optional)' },
      as_of_date: { type: 'string', format: 'date', description: 'Position as of date (optional)' },
    },
  },
}, async (params) => {
  const positions = await cashPositionService.getDashboardData(
    params.tenantId, params
  );
  return { content: [{ type: 'text', text: JSON.stringify(positions, null, 2) }] };
});

mcpServer.tool('get_forecast', { /* ... */ });
mcpServer.tool('get_fx_exposure', { /* ... */ });
mcpServer.tool('list_pending_payments', { /* ... */ });
mcpServer.tool('get_covenant_status', { /* ... */ });

// Resources
mcpServer.resource('treasury://positions/current', { /* ... */ });
mcpServer.resource('treasury://forecasts/active', { /* ... */ });
```

**Testing:**
- MCP server responds to tool discovery request with all tools listed
- get_cash_position returns correct data for tenant
- get_forecast returns active forecast with line items
- get_fx_exposure returns exposure summary by currency
- MCP server enforces tenant isolation (cannot access other tenant data)
- MCP server requires authentication via API key
- Tool responses formatted correctly for LLM consumption

### Task 7.2: Natural Language Treasury Queries

**What:** LLM-powered query interface that answers treasury questions in natural language by calling MCP tools and synthesizing responses.

**Design:**

```typescript
// apps/api/src/modules/mcp/query-handler.ts
// Uses Claude API to process natural language queries
// LLM selects appropriate MCP tools to gather data
// Synthesizes response in natural language with supporting data
//
// Example queries:
// "How much cash do we have in Europe today?"
// "What is our FX exposure to GBP this quarter?"
// "Are we compliant with all debt covenants?"
// "Show me the forecast variance for last week"
```

**Testing:**
- "How much cash do we have?" returns total cash in base currency
- "What is our EUR exposure?" returns EUR FX exposure with hedge ratio
- "Show me pending payments" returns list of payments awaiting approval
- Response includes both natural language summary and structured data
- Query with insufficient context prompts for clarification
- Query about unauthorized data returns appropriate error

### Task 7.3: Anomaly Detection & Intelligent Alerts

**What:** ML-based anomaly detection on daily cash flows. Detect unusual patterns (unexpected large transactions, missing expected receipts, timing anomalies) and surface via alert system.

**Design:**

```python
# apps/ml/src/models/anomaly_detection/model.py
# Isolation Forest trained on historical daily cash flow patterns
# Features: amount, day_of_week, category, entity, deviation from rolling mean
# Anomaly score threshold configurable per tenant
# Detected anomalies published to alert system
```

**Testing:**
- Normal transaction pattern: no anomaly flagged
- Transaction 3x larger than rolling average for category: anomaly flagged
- Expected payroll not received within expected window: anomaly flagged
- Anomaly alert includes context (what was expected vs. what happened)
- False positive rate < 5% on historical test data

### Definition of Done -- Phase 7

- [ ] MCP server exposing at least 5 treasury tools
- [ ] Natural language query endpoint answering basic treasury questions
- [ ] Anomaly detection model trained and running on daily schedule
- [ ] Anomaly alerts integrated with alert system
- [ ] MCP server authenticated and tenant-isolated
- [ ] Documentation: MCP tool catalog with example queries
- [ ] Test coverage >80% for MCP module

---

## Phase 8: Advanced Treasury (Hedge Accounting, Graph Layer)

**Goal:** Hedge accounting under ASC 815 / IFRS 9, hedge effectiveness testing, and optional graph layer for multi-entity hierarchy analytics.

**Duration:** 6--8 weeks

**Dependencies:** Phase 5 (FX trades), Phase 6 (debt instruments)

### Task 8.1: Hedge Designation & Documentation

**What:** Formal hedge designation workflow under ASC 815 or IFRS 9. Document hedged item, hedging instrument, risk management objective, and assessment methodology.

**Design:**

```typescript
// Following Data Model Suggestion 3's hedge_designation table
// documentation JSONB stores hedged_item, hedging_instrument, risk_management_objective
// Designation links FX trade to FX exposure
// Status lifecycle: active -> dedesignated | expired | matured
```

**Testing:**
- Cash flow hedge designation created linking EUR forward to forecasted EUR revenue
- Documentation JSONB contains all required ASC 815 fields
- Hedge designation links to specific FX trade and exposure
- De-designation records reason and effective date
- Audit trail complete for hedge lifecycle

### Task 8.2: Hedge Effectiveness Testing

**What:** Periodic hedge effectiveness testing (dollar offset, regression, critical terms match). Store test results for auditor review.

**Testing:**
- Critical terms match test passes when hedge and exposure have matching terms
- Dollar offset test calculates ratio of hedged item FV change vs. instrument FV change
- Effective hedge (ratio 80-125% under ASC 815): marked effective
- Ineffective hedge: generates alert and flags for de-designation review
- Test results appended to effectiveness_tests JSONB array

### Task 8.3: Graph Layer (Optional)

**What:** Implement the graph_node and graph_edge tables from Data Model Suggestion 4 to enable entity hierarchy traversal, ownership-weighted position aggregation, and counterparty concentration analysis.

**Testing:**
- Entity hierarchy query returns correct ownership tree
- Ownership-weighted cash position aggregation matches manual calculation
- Intercompany flow analysis identifies netting opportunities
- Counterparty concentration analysis identifies top 5 counterparties by exposure

### Definition of Done -- Phase 8

- [ ] Hedge designation workflow functional under ASC 815 and IFRS 9
- [ ] Effectiveness testing with at least 2 methods
- [ ] Test results stored and queryable for audit
- [ ] Graph layer operational for entity hierarchy queries (if implemented)
- [ ] Hedge accounting dashboard showing active designations and test results
- [ ] Test coverage >80% for hedge accounting module

---

## Phase 9: ISO 20022 & SWIFT Integration

**Goal:** Parse and generate ISO 20022 CAMT/PAIN messages. Support SWIFT MT940/MT942 bank statement import and MT101 payment initiation as fallback for banks without API connectivity.

**Duration:** 4--6 weeks

**Dependencies:** Phase 2 (bank statements), Phase 3 (payments)

### Task 9.1: ISO 20022 CAMT Parser

**What:** Parse camt.053 (end-of-day statement), camt.052 (intraday report), and camt.054 (debit/credit notification) XML messages into bank_statement and bank_transaction records.

**Design:**

```typescript
// packages/iso20022/src/parsers/camt053.ts
import { XMLParser } from 'fast-xml-parser';

export class Camt053Parser {
  parse(xml: string): ParsedBankStatement {
    const parser = new XMLParser({ ignoreAttributes: false });
    const doc = parser.parse(xml);
    const stmt = doc.Document.BkToCstmrStmt.Stmt;

    return {
      statementId: stmt.Id,
      accountIban: stmt.Acct.Id.IBAN,
      openingBalance: this.parseBalance(stmt.Bal, 'OPBD'),
      closingBalance: this.parseBalance(stmt.Bal, 'CLBD'),
      entries: (stmt.Ntry || []).map(entry => this.parseEntry(entry)),
    };
  }

  private parseEntry(ntry: any): ParsedTransaction {
    return {
      amount: parseFloat(ntry.Amt['#text']),
      currency: ntry.Amt['@_Ccy'],
      creditDebit: ntry.CdtDbtInd === 'CRDT' ? 'C' : 'D',
      bookingDate: ntry.BookgDt.Dt,
      valueDate: ntry.ValDt.Dt,
      entryReference: ntry.NtryRef,
      bankTransactionCode: {
        domain: ntry.BkTxCd?.Domn?.Cd,
        family: ntry.BkTxCd?.Domn?.Fmly?.Cd,
        subFamily: ntry.BkTxCd?.Domn?.Fmly?.SubFmlyCd,
      },
      remittanceInfo: ntry.NtryDtls?.TxDtls?.RmtInf?.Ustrd,
      counterpartyName: ntry.NtryDtls?.TxDtls?.RltdPties?.Dbtr?.Nm
        || ntry.NtryDtls?.TxDtls?.RltdPties?.Cdtr?.Nm,
    };
  }
}
```

**Testing:**
- Parse sample camt.053 XML with 10 entries: all entries extracted correctly
- Opening and closing balances extracted correctly
- Credit/debit indicator mapped correctly
- Counterparty name and remittance info extracted
- BankTransactionCode mapped to domain/family/subfamily
- Invalid XML returns descriptive parse error
- camt.052 (intraday) parsed with intraday balance type
- camt.054 (notification) parsed as individual transaction

### Task 9.2: ISO 20022 pain.001 Generator

**What:** Generate pain.001 (CustomerCreditTransferInitiation) XML for payment orders that cannot be submitted via API.

**Testing:**
- Generated pain.001 validates against ISO 20022 XSD schema
- SEPA payment generates correct pain.001 with EU-specific fields
- Wire transfer generates correct pain.001 with intermediary bank
- Batch of 10 payments generates single pain.001 with 10 CdtTrfTxInf elements
- Generated XML downloadable via API endpoint

### Task 9.3: SWIFT MT940 Parser (Legacy Fallback)

**What:** Parse SWIFT MT940 bank statement messages for banks still using legacy format.

**Testing:**
- Parse sample MT940 message with 5 transactions
- Field 60F (opening balance) extracted correctly
- Field 61 (statement line) entries parsed with correct amounts and dates
- Field 62F (closing balance) extracted correctly
- MT942 (interim statement) parsed for intraday balances

### Definition of Done -- Phase 9

- [ ] camt.053, camt.052, camt.054 parsing functional
- [ ] pain.001 generation functional and XSD-validated
- [ ] MT940/MT942 parsing functional (legacy fallback)
- [ ] Parsed statements create bank_statement and bank_transaction records
- [ ] ISO 20022 message upload endpoint working
- [ ] Test coverage >90% for parsers (financial data parsing requires high coverage)

---

## Phase 10: Production Hardening & Deployment

**Goal:** Security hardening, performance optimization, comprehensive documentation, deployment automation, and production readiness.

**Duration:** 4--6 weeks

**Dependencies:** All previous phases

### Task 10.1: Security Hardening

**What:** PCI DSS alignment for payment data handling, secret encryption at rest, rate limiting, CSRF protection, security headers, penetration test preparation.

**Testing:**
- All bank credentials encrypted at rest with AES-256
- API rate limiting: 100 req/min per user, 1000 req/min per tenant
- CSRF tokens required on all mutating endpoints
- Security headers (CSP, HSTS, X-Frame-Options) present on all responses
- SQL injection: parameterized queries verified on all database calls
- XSS: all user input sanitized in frontend
- Dependency audit: no known high/critical CVEs in dependencies

### Task 10.2: Performance Optimization

**What:** Database query optimization, connection pooling, caching strategy, index tuning, and load testing.

**Testing:**
- Cash position dashboard loads in <2 seconds with 100 accounts
- Balance sync for 100 accounts completes in <60 seconds
- Forecast generation completes in <30 seconds
- API response time p95 < 500ms under 50 concurrent users
- Database connection pool sized correctly (no pool exhaustion under load)
- Redis cache hit rate >80% for cash position queries
- PostgreSQL query plans reviewed: no sequential scans on indexed columns

### Task 10.3: Deployment Automation

**What:** Docker Compose production configuration, Kubernetes Helm chart, backup/restore procedures, monitoring (Prometheus/Grafana), and health check endpoints.

**Testing:**
- Docker Compose deployment starts all services with production configuration
- Kubernetes deployment with Helm chart creates all resources
- Health check endpoints return 200 for all services
- PostgreSQL backup and restore tested successfully
- Grafana dashboard shows API latency, error rate, and resource utilization
- Auto-scaling rules trigger under load (Kubernetes)
- Zero-downtime deployment via rolling update

### Task 10.4: Documentation & OpenAPI Spec

**What:** Complete OpenAPI 3.1 specification, deployment guide, user guide, and contributor documentation.

**Testing:**
- OpenAPI spec validates against OAS 3.1 schema
- All API endpoints documented with request/response examples
- Deployment guide: follow instructions and achieve running system
- SDK generated from OpenAPI spec compiles and calls API successfully

### Definition of Done -- Phase 10

- [ ] Security hardening checklist complete
- [ ] Performance targets met (dashboard <2s, API p95 <500ms)
- [ ] Docker Compose production deployment tested
- [ ] Kubernetes Helm chart tested
- [ ] Monitoring and alerting operational
- [ ] OpenAPI 3.1 spec published and validated
- [ ] Deployment guide tested by someone other than the author
- [ ] Backup/restore procedure tested
- [ ] Overall test coverage >75% across all modules
- [ ] README updated with final feature list and deployment instructions
- [ ] v1.0.0 release tagged

---

## Summary

| Phase | Name | Duration | Key Deliverables |
|-------|------|----------|-----------------|
| 1 | Foundation & Core Infrastructure | 4--6 weeks | Database, auth, RBAC, entity management, audit logging |
| 2 | Bank Connectivity & Cash Positioning | 5--7 weeks | Plaid/TrueLayer connectivity, balance sync, cash dashboard |
| 3 | Payments & Reconciliation | 5--7 weeks | Payment orders, approval workflow, bank submission, reconciliation |
| 4 | Cash Flow Forecasting (ML) | 6--8 weeks | RF + LSTM forecasting, transaction categorization, scenarios |
| 5 | FX Risk & Exposure Management | 4--6 weeks | FX rates, exposure calculation, FX trades, monitoring |
| 6 | Debt, Covenants & Alerts | 4--5 weeks | Debt tracking, covenant testing, notification system |
| 7 | AI/LLM Integration & MCP Server | 4--6 weeks | MCP server, NL queries, anomaly detection |
| 8 | Advanced Treasury | 6--8 weeks | Hedge accounting, effectiveness testing, graph layer |
| 9 | ISO 20022 & SWIFT Integration | 4--6 weeks | CAMT/PAIN parsers, MT940 parser, message generation |
| 10 | Production Hardening | 4--6 weeks | Security, performance, deployment, documentation |
| **Total** | | **46--63 weeks** | **Full-featured OSS TMS with AI forecasting** |

**MVP (Phases 1--4):** 20--28 weeks. Delivers cash positioning, payment execution, and ML forecasting -- the core value proposition for mid-market companies currently using spreadsheets.

**Full v1.0 (Phases 1--10):** 46--63 weeks. Delivers a feature-complete open-source TMS competitive with commercial mid-market offerings, differentiated by transparent ML forecasting, MCP/AI integration, and Open Banking-first connectivity.
