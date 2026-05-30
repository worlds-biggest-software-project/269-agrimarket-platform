# AgriMarket Platform — Phased Development Plan

> Project: 269-agrimarket-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

AgriMarket Platform is an AI-native, open-source agricultural commodity marketplace that connects crop producers, agri-buyers, traders, and lenders in a single transparent venue. It unifies spot trading, real-time price discovery, quality verification, logistics coordination, and embedded trade finance — closing the gap left by derivatives-only exchanges (CME), region-locked marketplaces (agribazaar, FBN), and produce-only finance platforms (Produce Pay). The AI layer powers dynamic pricing, buyer-seller matching, trade-finance underwriting, and fraud/quality dispute detection.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the canonical schema — strongly-typed columns for the transactional core (organizations, listings, orders, payments) plus JSONB extension columns for jurisdiction- and commodity-specific attributes that vary by market. This balances referential integrity and standards mapping (from Suggestion 1) with the speed-to-market and schema-evolution needs of a multi-region MVP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | The platform is AI-heavy (pricing, matching, fraud, underwriting, MCP tooling). Python has the strongest LLM/ML ecosystem (openai SDK, scikit-learn, pandas) and first-class async for I/O-bound integrations with external price feeds. |
| API framework | FastAPI | Native async, Pydantic v2 request/response validation, and automatic OpenAPI 3.1 generation — directly satisfies the `standards.md` requirement to publish the marketplace API in OAS3 for SDK generation and partner integration. |
| ASGI server | Uvicorn (dev) / Gunicorn+Uvicorn workers (prod) | Standard FastAPI deployment; worker model handles concurrent commodity feed subscriptions and webhook traffic. |
| Database | PostgreSQL 16 | The hybrid model needs strong relational constraints AND rich JSONB querying (GIN indexes, JSON path ops). Postgres also provides range partitioning for `price_feed`/`audit_log` time series and PostGIS for logistics-proximity matching. |
| Spatial extension | PostGIS | AI match engine ranks by logistics proximity; PostGIS `geography` columns + KNN distance operators make "nearest verified buyer/seller" queries fast. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Async ORM matches FastAPI; Alembic gives reproducible DDL migrations required by a compliance-first financial platform. |
| Task queue | Celery + Redis | Async workloads: external price-feed polling, AI inference jobs, webhook delivery, settlement message generation, alert evaluation. Redis doubles as cache and Celery broker/result backend. |
| Cache / pub-sub | Redis 7 | Caches latest price quotes, rate-limits APIs, and powers WebSocket fan-out for real-time price updates. |
| Real-time transport | FastAPI WebSockets + Redis pub/sub | Real-time price discovery and live bid updates pushed to connected clients. |
| LLM / AI provider | Pluggable provider abstraction (OpenAI + local fallback) | Pricing rationale, match explanations, fraud triage, and finance underwriting summaries use an LLM behind a provider-agnostic interface so self-hosters can swap models. Deterministic numeric models (pricing, scoring) use scikit-learn, not the LLM. |
| AI tool exposure | MCP server (Python MCP SDK) | `standards.md` flags MCP (2025-11-25 spec) as highly relevant. A bundled MCP server exposes price feeds, listings, supply data, and analytics as callable tools for AI agents. |
| Auth | OAuth 2.0 + OIDC (Authlib) + JWT (RFC 7519) | `standards.md` mandates OAuth 2.0 (RFC 6749), OIDC federated login, and JWT access tokens. Finance endpoints layer FAPI 2.0-style controls. |
| Frontend | Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui | Buyer/seller dashboard, listing discovery, live price charts, order management. Server components for SEO-friendly public listings; client components for real-time feeds. |
| Payments | Provider abstraction (Stripe Connect dev adapter) + ISO 20022 pacs.008 message generation | Escrow + settlement via a pluggable gateway; cross-border settlement emits ISO 20022 messages per `standards.md`. |
| Containerisation | Docker + docker-compose | Self-hostable open-source deployment; compose wires Postgres, Redis, API, worker, frontend, MCP server. |
| Testing | pytest + pytest-asyncio + httpx + testcontainers | Unit, mocked-integration, and real-dependency (Postgres/Redis via testcontainers) tiers. |
| Frontend testing | Vitest + Playwright | Component tests and E2E browser flows. |
| Code quality | Ruff (lint+format) + mypy (strict) | Fast linting/formatting and static typing on a financial codebase. |
| Package manager | uv (Python) / pnpm (frontend) | Fast, reproducible installs. |
| Key libraries | `pydantic`, `sqlalchemy`, `alembic`, `celery`, `redis`, `authlib`, `httpx`, `scikit-learn`, `pandas`, `geoalchemy2`, `python-jose`, `mcp` | Domain- and infra-specific support. |
| Standards outputs | OpenAPI 3.1, ISO 20022 pacs.008, GS1 EPCIS 2.0 events, W3C Verifiable Credentials | Wire formats the platform must produce/consume per `standards.md`. |

### Project Structure

```
agrimarket-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── openapi/                         # generated + committed OAS3 spec snapshot
│   └── agrimarket.openapi.json
├── migrations/                      # Alembic versions
│   └── versions/
├── src/
│   └── agrimarket/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory, router wiring
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py           # async engine + session
│       │   ├── base.py              # declarative base, mixins (TimestampMixin)
│       │   └── models/              # SQLAlchemy models grouped by domain
│       │       ├── identity.py      # organization, user_account, org_member
│       │       ├── catalogue.py     # commodity, commodity_grade
│       │       ├── marketplace.py   # listing, location, bid, purchase_order
│       │       ├── quality.py       # inspection, certification, lot, supply_chain_event
│       │       ├── payments.py      # payment, escrow, trade_finance, finance_drawdown
│       │       ├── reputation.py    # review, dispute
│       │       ├── market.py        # price_feed, ai_model_output, market_alert
│       │       └── audit.py         # audit_log
│       ├── schemas/                 # Pydantic request/response DTOs (per domain)
│       ├── api/
│       │   ├── deps.py              # auth, db session, pagination deps
│       │   └── routes/             # FastAPI routers (per domain)
│       ├── services/                # business logic (command handlers)
│       │   ├── listings.py
│       │   ├── orders.py
│       │   ├── matching.py
│       │   ├── pricing.py
│       │   ├── payments.py
│       │   ├── finance.py
│       │   ├── quality.py
│       │   └── disputes.py
│       ├── ai/
│       │   ├── provider.py          # LLM provider abstraction
│       │   ├── pricing_engine.py    # dynamic pricing model
│       │   ├── match_engine.py      # supply↔demand scoring
│       │   ├── underwriter.py       # trade-finance risk scoring
│       │   └── fraud.py             # anomaly / dispute triage
│       ├── integrations/
│       │   ├── pricefeeds/          # CME, USDA AMS, DTN, Agmarknet, Commodities-API adapters
│       │   ├── payments/            # gateway adapters + ISO 20022 emitter
│       │   ├── epcis.py             # GS1 EPCIS 2.0 event mapping
│       │   └── vc.py                # W3C Verifiable Credentials issue/verify
│       ├── auth/                    # OAuth2/OIDC/JWT, RBAC
│       ├── realtime/                # WebSocket hub + Redis pub/sub
│       ├── tasks/                   # Celery app + task modules
│       └── mcp/
│           └── server.py            # MCP server exposing platform tools
├── tests/
│   ├── conftest.py                  # fixtures: db, redis, client, factories
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── seeds/                           # reference data (commodities, grades, jurisdictions)
└── frontend/
    ├── package.json
    ├── app/                         # Next.js App Router
    ├── components/
    ├── lib/api-client.ts            # generated from OpenAPI
    └── tests/
```

The structure groups by concern (models, schemas, services, integrations, ai). Each phase adds modules without restructuring.

---

## Phase 1: Foundation & Project Skeleton

### Purpose
Establish the runnable application shell, configuration, database connectivity, migrations, containerisation, and the test harness. After this phase a developer can start the stack with `docker compose up`, hit a health endpoint, run migrations, and execute the test suite — the platform every later phase builds on.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Create the Python package, dependency manifest, linting/typing config, and CI-ready test setup.

**Design**:
- `pyproject.toml` with dependencies from the tech table; tool config for Ruff and mypy (strict mode, `disallow_untyped_defs = true`).
- `src/agrimarket/config.py` using Pydantic `BaseSettings`:

```python
class Settings(BaseSettings):
    environment: Literal["dev", "test", "prod"] = "dev"
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_ttl_seconds: int = 900
    refresh_token_ttl_seconds: int = 1_209_600
    llm_provider: Literal["openai", "null"] = "null"
    openai_api_key: str | None = None
    default_currency: str = "USD"
    model_config = SettingsConfigDict(env_file=".env", env_prefix="AGRIMARKET_")
```

- `.env.example` documenting every variable.

**Testing**:
- `Unit: Settings loads from env vars → correct typed values, defaults applied`
- `Unit: missing required AGRIMARKET_DATABASE_URL → ValidationError naming the field`
- `Unit: invalid environment value 'staging' → ValidationError`

#### 1.2 — Database session, base model, and Alembic

**What**: Async SQLAlchemy engine/session and Alembic migration environment.

**Design**:
- `db/session.py`: `create_async_engine(settings.database_url)`, `async_sessionmaker`, `get_session()` async generator dependency.
- `db/base.py`: declarative `Base`; `TimestampMixin` providing `created_at`/`updated_at` (`server_default=func.now()`, `onupdate=func.now()`).
- Enable extensions in the first migration: `CREATE EXTENSION IF NOT EXISTS pgcrypto;` (for `gen_random_uuid()`) and `CREATE EXTENSION IF NOT EXISTS postgis;`.
- Alembic configured for async (`run_async_migrations`).

**Testing**:
- `Integration (real, testcontainers Postgres): run alembic upgrade head → pgcrypto + postgis extensions present`
- `Integration: open session, SELECT 1 → returns 1`
- `Integration: alembic downgrade base then upgrade head → succeeds idempotently`

#### 1.3 — FastAPI app factory & health endpoint

**What**: App factory wiring middleware, exception handlers, and a health route.

**Design**:
- `main.py`: `create_app() -> FastAPI` with CORS, request-ID middleware, a global exception handler returning RFC 7807 problem+json bodies.
- `GET /healthz` → `{"status": "ok", "db": "ok"|"down", "redis": "ok"|"down"}` (pings dependencies).
- `GET /version` → build/version metadata.

**Testing**:
- `Integration (mocked deps): GET /healthz with healthy deps → 200, status ok`
- `Integration: GET /healthz with db down → 503, db: "down"`
- `Unit: unhandled exception → 500 problem+json with type, title, detail`

#### 1.4 — Docker & compose

**What**: Containerise API, worker, and dependencies.

**Design**:
- Multi-stage `Dockerfile` (builder installs with uv; runtime is slim).
- `docker-compose.yml` services: `db` (postgis/postgis:16), `redis`, `api`, `worker`, `mcp`, `frontend`. Healthchecks; `api` waits on `db`/`redis` healthy.

**Testing**:
- `E2E (CI): docker compose up; poll GET /healthz until 200 → all services healthy within timeout`
- `Manual checklist: docker build succeeds for api image`

### Definition of Done
Skeleton runs via compose, `/healthz` green, migrations apply, Ruff/mypy/pytest pass in CI.

---

## Phase 2: Identity, Organizations & Authentication

### Purpose
Model the participant graph (users, organizations, memberships) and secure the API with OAuth 2.0 / OIDC / JWT and role-based access control. Every subsequent action is attributable to a user acting on behalf of an organization, satisfying the audit and authorisation requirements of a financial marketplace (OWASP API Security Top 10 — BOLA/BFLA).

### Tasks

#### 2.1 — Identity data model

**What**: `organization`, `user_account`, `org_member` tables (Suggestion 3, augmented with Suggestion 1's standards columns).

**Design** (SQL DDL):

```sql
CREATE TABLE organization (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_name TEXT NOT NULL,
    trade_name TEXT,
    org_type TEXT NOT NULL CHECK (org_type IN
        ('farm','cooperative','processor','exporter','importer','trader','lender','government','logistics_provider')),
    country_code TEXT NOT NULL CHECK (length(country_code)=2),   -- ISO 3166-1
    subdivision_code TEXT,                                       -- ISO 3166-2
    gln TEXT CHECK (gln IS NULL OR length(gln)=13),              -- GS1 GLN
    lei TEXT CHECK (lei IS NULL OR length(lei)=20),              -- ISO 17442 LEI
    tax_id TEXT,
    verification_status TEXT NOT NULL DEFAULT 'pending'
        CHECK (verification_status IN ('pending','verified','suspended','rejected')),
    attributes JSONB NOT NULL DEFAULT '{}',                      -- jurisdiction-specific fields
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_type ON organization(org_type);
CREATE INDEX idx_org_attrs ON organization USING gin (attributes);

CREATE TABLE user_account (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    phone TEXT,
    display_name TEXT NOT NULL,
    password_hash TEXT,                  -- null for SSO-only
    identity_provider TEXT NOT NULL DEFAULT 'local',
    identity_provider_sub TEXT,
    preferred_locale TEXT NOT NULL DEFAULT 'en',
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_user_idp ON user_account(identity_provider, identity_provider_sub)
    WHERE identity_provider <> 'local';

CREATE TABLE org_member (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES user_account(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    role TEXT NOT NULL CHECK (role IN ('owner','admin','trader','finance','viewer')),
    is_primary BOOLEAN NOT NULL DEFAULT false,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at TIMESTAMPTZ,
    UNIQUE (user_id, organization_id, role)
);
CREATE INDEX idx_member_org ON org_member(organization_id);
CREATE INDEX idx_member_user ON org_member(user_id);
```

**Testing**:
- `Unit (model): create org with invalid org_type → IntegrityError (check constraint)`
- `Unit: gln of wrong length → IntegrityError`
- `Integration: insert user + org + membership → relationships traversable`
- `Integration: duplicate email → unique violation`

#### 2.2 — Password & local auth

**What**: Registration, login, JWT issuance/refresh.

**Design**:
- Argon2 hashing (`argon2-cffi`).
- Endpoints:
  - `POST /auth/register` → `{email, password, display_name}` → 201 `{user_id}`
  - `POST /auth/login` → `{email, password}` → 200 `{access_token, refresh_token, token_type:"bearer", expires_in}`
  - `POST /auth/refresh` → `{refresh_token}` → new access token
  - `POST /auth/logout` → revokes refresh token (Redis denylist)
- Access token claims (RFC 7519): `sub` (user_id), `orgs` (list of `{org_id, role}`), `iat`, `exp`, `jti`.

**Testing**:
- `Integration: register then login → valid JWT decodes with correct sub`
- `Integration: login wrong password → 401, generic message (no user enumeration)`
- `Unit: expired token → 401`
- `Integration: refresh after logout → 401 (denylisted)`

#### 2.3 — OAuth 2.0 / OIDC federated login

**What**: Authorization-code login via external IdPs (Google, Microsoft, gov digital ID) using Authlib.

**Design**:
- `GET /auth/oidc/{provider}/start` → 302 to IdP authorize URL (PKCE).
- `GET /auth/oidc/{provider}/callback` → exchanges code, validates ID token, upserts `user_account` keyed on `(identity_provider, identity_provider_sub)`, issues platform JWT.
- Provider config (issuer, client id/secret, scopes) in settings.

**Testing**:
- `Integration (mocked IdP): valid callback → user upserted, platform JWT returned`
- `Integration (mocked): tampered state → 400`
- `Integration (mocked): invalid ID token signature → 401`

#### 2.4 — RBAC & request context

**What**: Dependencies that resolve the current user, active org, and enforce roles.

**Design**:
- `deps.get_current_user()` parses/validates JWT.
- `deps.require_org_role(*roles)` — checks the caller's membership/role for the `X-Org-Id` header (or path org). Enforces object-level authorisation (OWASP API1) so a user cannot act for an org they do not belong to.
- Roles: `owner`>`admin`>`trader`>`finance`>`viewer` (capability matrix documented).

**Testing**:
- `Unit: user with trader role calling finance-only endpoint → 403`
- `Integration: user A requesting org B resource without membership → 403`
- `Unit: missing/invalid token → 401`

### Definition of Done
Users register, log in (local + OIDC), receive JWTs; RBAC enforced; auth flows covered by tests; endpoints appear in the generated OpenAPI spec.

---

## Phase 3: Commodity Catalogue & Marketplace Listings

### Purpose
Deliver the first half of the core value proposition: a structured commodity catalogue with standards-aligned grades, and the ability for sellers to create, search, and manage listings. This is the marketplace's foundational inventory layer.

### Tasks

#### 3.1 — Commodity & grade catalogue

**What**: Reference catalogue of commodities and their quality grades.

**Design** (SQL DDL, GIPSA/Codex-aligned):

```sql
CREATE TABLE commodity (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code TEXT NOT NULL UNIQUE,            -- e.g. 'CORN_YEL_2'
    name TEXT NOT NULL,
    category TEXT NOT NULL,               -- 'grain','pulse','oilseed','produce', ...
    codex_code TEXT,                      -- Codex Alimentarius classification
    gpc_code TEXT,                        -- GS1 Global Product Classification
    variety TEXT,
    default_uom TEXT NOT NULL CHECK (default_uom IN ('mt','bushel','cwt','kg','lb','ton')),
    hs_code TEXT,                         -- Harmonized System tariff code
    is_active BOOLEAN NOT NULL DEFAULT true,
    attributes JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE commodity_grade (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commodity_id UUID NOT NULL REFERENCES commodity(id),
    grading_system TEXT NOT NULL CHECK (grading_system IN ('GIPSA','Codex','EU','national','custom')),
    grade_code TEXT NOT NULL,             -- e.g. 'US_2'
    grade_name TEXT NOT NULL,             -- 'U.S. No. 2 Yellow Corn'
    spec JSONB NOT NULL DEFAULT '{}',     -- {max_moisture, min_test_weight, max_damage, ...}
    is_active BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (commodity_id, grading_system, grade_code)
);
```

- Seed file `seeds/commodities.json` covering major grains/pulses/oilseeds + GIPSA grades.
- Endpoints: `GET /commodities` (filter by category/active), `GET /commodities/{id}/grades`, admin `POST/PATCH` (role `admin`).

**Testing**:
- `Integration: seed load → expected commodity count, GIPSA corn grades present`
- `Integration: GET /commodities?category=grain → only grains`
- `Unit: duplicate (commodity, grading_system, grade_code) → unique violation`

#### 3.2 — Locations (PostGIS)

**What**: Geocoded locations for farms, warehouses, elevators, ports.

**Design**:

```sql
CREATE TABLE location (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organization(id),
    gln TEXT,
    name TEXT NOT NULL,
    location_type TEXT NOT NULL CHECK (location_type IN
        ('farm','warehouse','elevator','port','mill','processing_plant','delivery_point')),
    address JSONB NOT NULL DEFAULT '{}',
    country_code TEXT NOT NULL CHECK (length(country_code)=2),
    geo geography(Point,4326),            -- PostGIS point
    capacity_mt NUMERIC,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_location_geo ON location USING gist (geo);
CREATE INDEX idx_location_org ON location(organization_id);
```

- `POST /orgs/{org_id}/locations` (role `admin`/`trader`), `GET /locations?near=lat,lon&radius_km=`.

**Testing**:
- `Integration: create location with coords → geo populated; KNN query returns it within radius`
- `Unit: missing required name → 422`

#### 3.3 — Listing CRUD

**What**: Sellers create and manage commodity offers.

**Design**:

```sql
CREATE TABLE listing (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_org_id UUID NOT NULL REFERENCES organization(id),
    created_by UUID NOT NULL REFERENCES user_account(id),
    commodity_id UUID NOT NULL REFERENCES commodity(id),
    grade_id UUID REFERENCES commodity_grade(id),
    gtin TEXT,
    listing_type TEXT NOT NULL CHECK (listing_type IN ('spot','forward','auction')),
    quantity NUMERIC NOT NULL CHECK (quantity > 0),
    remaining_qty NUMERIC NOT NULL CHECK (remaining_qty >= 0),
    uom TEXT NOT NULL,
    price_per_unit NUMERIC,              -- null for auction
    currency_code TEXT NOT NULL CHECK (length(currency_code)=3),
    price_type TEXT NOT NULL CHECK (price_type IN ('fixed','basis','negotiable','auction')),
    basis_reference TEXT,
    basis_offset NUMERIC,
    origin_location_id UUID REFERENCES location(id),
    delivery_location_id UUID REFERENCES location(id),
    delivery_terms TEXT CHECK (delivery_terms IN ('FOB','CIF','DAP','FCA','EXW')), -- Incoterms 2020
    harvest_year INTEGER,
    available_from DATE NOT NULL,
    available_until DATE,
    organic_certified BOOLEAN NOT NULL DEFAULT false,
    attributes JSONB NOT NULL DEFAULT '{}',
    status TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft','active','matched','partially_sold','sold','expired','cancelled')),
    views_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listing_active ON listing(commodity_id, status, price_per_unit) WHERE status='active';
CREATE INDEX idx_listing_seller ON listing(seller_org_id);
CREATE INDEX idx_listing_attrs ON listing USING gin (attributes);
```

- Lifecycle state machine: `draft → active → (partially_sold) → sold | matched | expired | cancelled`. Service rejects illegal transitions.
- Endpoints: `POST /listings`, `PATCH /listings/{id}`, `POST /listings/{id}/publish`, `POST /listings/{id}/cancel`, `GET /listings/{id}`.
- Pydantic `ListingCreate` validates: `price_per_unit` required unless `listing_type='auction'`; `basis_offset` required when `price_type='basis'`.

**Testing**:
- `Unit: ListingCreate fixed price without price_per_unit → 422`
- `Integration: create draft then publish → status active`
- `Integration: cancel a sold listing → 409 illegal transition`
- `Integration: non-member of seller_org creating listing → 403`

#### 3.4 — Listing search & discovery

**What**: Filtered, paginated, geo-aware discovery feed.

**Design**:
- `GET /marketplace/listings` query params: `commodity_code`, `grade_code`, `min_price`, `max_price`, `country`, `near` (lat,lon), `radius_km`, `organic`, `delivery_terms`, `sort` (`price_asc|price_desc|newest|nearest`), `cursor`, `limit` (default 25, max 100).
- Cursor pagination; `Link` header per RFC 8288 (`rel="next"`).
- Only `status='active'` listings; increments `views_count` on detail fetch (async via Celery to avoid write contention).

**Testing**:
- `Integration: filter by commodity + price range → only matching active listings`
- `Integration: sort=nearest with near coords → ordered by distance`
- `Integration: pagination → Link: next header present; following cursor yields next page, no overlap`
- `Integration: draft listing excluded from search`

### Definition of Done
Catalogue seeded; sellers create/publish/manage listings with enforced state machine; buyers search with filters, geo, and pagination; OpenAPI updated; migrations created.

---

## Phase 4: Bidding, Orders & Reviews

### Purpose
Complete the transactional core: buyers bid or buy, sellers accept, orders progress through their lifecycle, and both parties build reputation via reviews. This makes the platform a functioning spot marketplace.

### Tasks

#### 4.1 — Bids & negotiation

**What**: Buyers place bids; sellers accept/reject/counter.

**Design**:

```sql
CREATE TABLE bid (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id UUID NOT NULL REFERENCES listing(id),
    buyer_org_id UUID NOT NULL REFERENCES organization(id),
    created_by UUID NOT NULL REFERENCES user_account(id),
    bid_price NUMERIC NOT NULL CHECK (bid_price > 0),
    quantity NUMERIC NOT NULL CHECK (quantity > 0),
    currency_code TEXT NOT NULL,
    message TEXT,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','accepted','rejected','countered','expired','withdrawn')),
    counter_of UUID REFERENCES bid(id),
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bid_listing ON bid(listing_id, status);
```

- `POST /listings/{id}/bids`, `POST /bids/{id}/accept|reject|counter|withdraw`.
- Accepting a bid creates a `purchase_order` (4.2) and decrements `listing.remaining_qty`; broadcasts via WebSocket (Phase 6).
- Validation: bid quantity ≤ `remaining_qty`; cannot bid on own listing.

**Testing**:
- `Integration: place bid → pending; seller accepts → order created, remaining_qty reduced`
- `Unit: bid on own org listing → 403`
- `Integration: bid quantity > remaining → 422`
- `Integration: counter creates linked bid with counter_of set`

#### 4.2 — Purchase orders & lifecycle

**What**: Order entity tracking a confirmed trade from creation to completion.

**Design**:

```sql
CREATE TABLE purchase_order (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number TEXT NOT NULL UNIQUE,     -- AGM-YYYY-NNNNN
    listing_id UUID NOT NULL REFERENCES listing(id),
    bid_id UUID REFERENCES bid(id),
    buyer_org_id UUID NOT NULL REFERENCES organization(id),
    seller_org_id UUID NOT NULL REFERENCES organization(id),
    commodity_id UUID NOT NULL REFERENCES commodity(id),
    grade_id UUID REFERENCES commodity_grade(id),
    quantity NUMERIC NOT NULL CHECK (quantity > 0),
    uom TEXT NOT NULL,
    agreed_price NUMERIC NOT NULL,
    total_value NUMERIC NOT NULL,
    currency_code TEXT NOT NULL,
    delivery_terms TEXT,
    delivery_date DATE,
    delivery_location_id UUID REFERENCES location(id),
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','accepted','awaiting_payment','paid','in_transit','delivered','completed','disputed','cancelled')),
    accepted_at TIMESTAMPTZ, delivered_at TIMESTAMPTZ, completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_order_buyer ON purchase_order(buyer_org_id, status);
CREATE INDEX idx_order_seller ON purchase_order(seller_org_id, status);
```

- `order_number` generated `AGM-{year}-{zero-padded sequence}`.
- State machine enforced in `services/orders.py`; transitions audited (Phase 9).
- Endpoints: `GET /orders` (scoped to caller's orgs), `GET /orders/{id}`, `POST /orders/{id}/confirm-delivery`, `POST /orders/{id}/cancel`.

**Testing**:
- `Integration: accepted bid → order with computed total_value = price*qty`
- `Integration: confirm-delivery on in_transit order → delivered`
- `Integration: cancel completed order → 409`
- `Integration: buyer from unrelated org → 403 on GET /orders/{id}`

#### 4.3 — Ratings & reviews

**What**: Post-trade reputation between counterparties.

**Design**:

```sql
CREATE TABLE review (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES purchase_order(id),
    reviewer_org_id UUID NOT NULL REFERENCES organization(id),
    reviewee_org_id UUID NOT NULL REFERENCES organization(id),
    reviewer_role TEXT NOT NULL CHECK (reviewer_role IN ('buyer','seller')),
    overall_rating SMALLINT NOT NULL CHECK (overall_rating BETWEEN 1 AND 5),
    quality_rating SMALLINT CHECK (quality_rating BETWEEN 1 AND 5),
    reliability_rating SMALLINT CHECK (reliability_rating BETWEEN 1 AND 5),
    communication_rating SMALLINT CHECK (communication_rating BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (order_id, reviewer_org_id)
);
```

- `POST /orders/{id}/review` allowed only when order `completed` and reviewer is a party.
- `GET /orgs/{id}/reputation` → aggregate `{avg_overall, count, on_time_pct, dispute_pct}` (computed; cached in Redis).

**Testing**:
- `Integration: review on completed order by buyer → 201; second review by same org → 409`
- `Integration: review on non-completed order → 409`
- `Integration: reputation aggregate reflects new review`

### Definition of Done
Full bid→order→delivery→review flow works end-to-end; state machines enforced; reputation computed; tests cover happy and illegal paths; OpenAPI updated.

---

## Phase 5: Payments, Escrow & Settlement

### Purpose
Add money movement: escrow to de-risk first-time counterparties, settlement against delivery, and ISO 20022-compliant cross-border payment messaging. Payment endpoints adopt elevated (FAPI-style) controls per `standards.md`.

### Tasks

#### 5.1 — Payment & escrow data model

**What**: Payment instructions and escrow accounts tied to orders.

**Design**:

```sql
CREATE TABLE payment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES purchase_order(id),
    payer_org_id UUID NOT NULL REFERENCES organization(id),
    payee_org_id UUID NOT NULL REFERENCES organization(id),
    amount NUMERIC NOT NULL CHECK (amount > 0),
    currency_code TEXT NOT NULL CHECK (length(currency_code)=3),
    method TEXT NOT NULL CHECK (method IN ('bank_transfer','escrow','letter_of_credit','mobile_money','platform_wallet')),
    instruction_id TEXT UNIQUE,        -- ISO 20022 InstrId
    end_to_end_id TEXT,                -- ISO 20022 EndToEndId
    payer_bic TEXT, payee_bic TEXT,    -- SWIFT BIC
    payer_iban TEXT, payee_iban TEXT,
    iso20022_message JSONB,            -- generated pacs.008 (stored for audit)
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','processing','completed','failed','refunded')),
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE escrow_account (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL UNIQUE REFERENCES purchase_order(id),
    buyer_org_id UUID NOT NULL REFERENCES organization(id),
    seller_org_id UUID NOT NULL REFERENCES organization(id),
    amount NUMERIC NOT NULL CHECK (amount > 0),
    currency_code TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','funded','released','refunded','disputed')),
    funded_at TIMESTAMPTZ, released_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing**:
- `Unit: payment amount <= 0 → IntegrityError`
- `Integration: one escrow per order (unique order_id) enforced`

#### 5.2 — Payment gateway abstraction

**What**: Provider-agnostic gateway interface with a Stripe Connect dev adapter and a deterministic `FakeGateway` for tests.

**Design**:

```python
class PaymentGateway(Protocol):
    async def create_escrow(self, order_id: UUID, amount: Decimal, currency: str) -> EscrowRef: ...
    async def capture(self, escrow_ref: str) -> CaptureResult: ...   # buyer funds escrow
    async def release(self, escrow_ref: str, to_org: UUID) -> PayoutResult: ...  # to seller on delivery
    async def refund(self, escrow_ref: str) -> RefundResult: ...      # to buyer on dispute
```

- Webhook receiver `POST /webhooks/payments/{provider}` verifies signature, enqueues a Celery task to advance escrow/payment/order state idempotently (dedupe on provider event id).
- Settlement flow: order `accepted` → create+fund escrow → order `paid` → delivery confirmed → release escrow → order `completed`.

**Testing**:
- `Integration (FakeGateway): fund escrow → escrow funded, order paid`
- `Integration: confirm delivery → escrow released, order completed`
- `Integration (mocked): webhook with bad signature → 401, no state change`
- `Integration: duplicate webhook event id → processed once (idempotent)`

#### 5.3 — ISO 20022 settlement message generation

**What**: Emit pacs.008.001 (FI-to-FI Customer Credit Transfer) for cross-border bank-transfer settlements.

**Design**:
- `integrations/payments/iso20022.py`: `build_pacs008(payment: Payment) -> dict` producing the ISO 20022 message structure (Group Header, Credit Transfer Transaction Info with `InstrId`, `EndToEndId`, debtor/creditor agents by BIC, amount/currency). Validate against the pacs.008 XSD/schema; store JSON form in `payment.iso20022_message`.
- `GET /payments/{id}/iso20022` (role `finance`) returns the message (XML or JSON).

**Testing**:
- `Unit: build_pacs008 → required fields present, amount/currency correct, IDs propagated`
- `Unit: missing payee BIC for bank_transfer → ValidationError`
- `Fixture: generated XML validates against pacs.008 schema`

#### 5.4 — Finance-grade API controls

**What**: Elevated security on payment/finance endpoints (FAPI 2.0-inspired).

**Design**:
- Require fresh, narrowly-scoped tokens for payment mutations; enforce sender-constrained tokens (DPoP-style nonce) and short TTL.
- Rate-limit and anomaly-flag (hooks into Phase 8 fraud).

**Testing**:
- `Integration: payment mutation with stale/over-scoped token → 401/403`
- `Integration: replayed DPoP nonce → 401`

### Definition of Done
Escrow-backed settlement works end-to-end with the fake gateway; ISO 20022 messages generated and schema-valid; finance endpoints carry elevated controls; webhooks idempotent; tests pass.

---

## Phase 6: Real-Time Price Discovery & External Feeds

### Purpose
Deliver the second half of the core value proposition — transparent, real-time price discovery — by ingesting external market data (CME, USDA AMS, DTN, Agmarknet, Commodities-API), aggregating platform trade prices, and streaming live updates to clients.

### Tasks

#### 6.1 — Price feed data model & partitioning

**What**: Time-series store of price observations from internal and external sources.

**Design**:

```sql
CREATE TABLE price_feed (
    id UUID DEFAULT gen_random_uuid(),
    commodity_id UUID NOT NULL REFERENCES commodity(id),
    source TEXT NOT NULL CHECK (source IN ('platform','cme','usda_ams','agmarknet','dtn','commodities_api','twelve_data','manual')),
    quote_type TEXT NOT NULL CHECK (quote_type IN ('spot','bid','ask','settlement','cash_bid')),
    price NUMERIC NOT NULL,
    currency_code TEXT NOT NULL,
    uom TEXT NOT NULL,
    location_id UUID REFERENCES location(id),
    grade_id UUID REFERENCES commodity_grade(id),
    contract_month TEXT,
    raw JSONB NOT NULL DEFAULT '{}',     -- original provider payload
    quoted_at TIMESTAMPTZ NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, quoted_at)
) PARTITION BY RANGE (quoted_at);
CREATE INDEX idx_price_commodity_time ON price_feed(commodity_id, quoted_at DESC);
CREATE INDEX idx_price_source ON price_feed(source, commodity_id, quoted_at DESC);
```

- Monthly partitions auto-created by a Celery beat job.

**Testing**:
- `Integration: insert into current month → routed to correct partition`
- `Integration: query last 24h for commodity → returns ordered series`

#### 6.2 — External feed adapter framework

**What**: Pluggable adapters normalising each provider into `price_feed` rows.

**Design**:

```python
class PriceFeedAdapter(Protocol):
    source: str
    async def fetch(self, commodities: list[str], since: datetime) -> list[NormalizedQuote]: ...
```

- `NormalizedQuote`: `{commodity_code, quote_type, price, currency, uom, location_gln?, contract_month?, quoted_at, raw}`.
- Adapters: `CMEAdapter` (REST/WebSocket, API key), `USDAAMSAdapter` (MARS API), `DTNAdapter` (grain cash bids, `apikey` header), `AgmarknetAdapter` (data.gov.in; tolerant of gaps per `standards.md` note), `CommoditiesAPIAdapter`, `TwelveDataAdapter`.
- Celery beat schedules polling per source; HTTP cached + rate-limited; failures logged, not fatal.
- Config: API keys per source in settings; per-source enable flags.

**Testing**:
- `Unit (per adapter, recorded fixtures): provider JSON → correct NormalizedQuote list`
- `Unit: Agmarknet partial/missing fields → skipped gracefully, no crash`
- `Integration (mocked HTTP): poll task → rows inserted with correct source`

#### 6.3 — Price aggregation & reference index

**What**: Compute platform price index per commodity/region from listings, accepted orders, and external feeds.

**Design**:
- `services/pricing.py: compute_reference_price(commodity_id, region, window)` → volume-weighted average of recent platform trades blended with latest external settlement; returns `{price, currency, confidence, components}`.
- `GET /prices/{commodity_code}` → latest reference + per-source breakdown.
- `GET /prices/{commodity_code}/history?from=&to=&source=` → series for charts.

**Testing**:
- `Unit: reference price = volume-weighted blend of fixture trades + feed`
- `Unit: no data → confidence 0, price null`
- `Integration: history endpoint returns chronologically ordered points`

#### 6.4 — WebSocket live price & bid stream

**What**: Push price/bid updates to subscribed clients via Redis pub/sub.

**Design**:
- `WS /ws/prices?commodities=CORN_YEL_2,WHEAT_HRW` — server subscribes the socket to Redis channels `price:{commodity}`; new `price_feed` inserts and reference recomputes publish updates.
- `WS /ws/listings/{id}` — bid/price changes for a listing (ties to Phase 4 bids).
- Auth via token query param or subprotocol header.

**Testing**:
- `Integration: subscribe to commodity, publish price event → client receives matching message`
- `Integration: new bid on listing → subscribers receive bid update`
- `Integration: unauthenticated WS connect → closed with policy-violation code`

### Definition of Done
External feeds ingest into partitioned price store; reference index computed; REST history + live WebSocket streams operational; adapters fault-tolerant; tests with recorded fixtures pass.

---

## Phase 7: AI Pricing & Matching Engines

### Purpose
Activate the first AI-native differentiators: dynamic pricing recommendations and intelligent buyer-seller matching. These turn the marketplace from a passive venue into an advisory one, the core of the project's positioning.

### Tasks

#### 7.1 — LLM provider abstraction & AI output store

**What**: Provider-agnostic LLM interface and a table recording AI outputs for auditability and training.

**Design**:

```python
class LLMProvider(Protocol):
    async def complete(self, system: str, user: str, *, json_schema: dict | None = None) -> LLMResult: ...
```

```sql
CREATE TABLE ai_model_output (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_kind TEXT NOT NULL CHECK (model_kind IN ('pricing','matching','underwriting','fraud','alert')),
    subject_type TEXT NOT NULL,         -- 'listing','order','org','facility'
    subject_id UUID NOT NULL,
    inputs JSONB NOT NULL,
    output JSONB NOT NULL,              -- {recommendation, score, rationale}
    model_version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_subject ON ai_model_output(subject_type, subject_id);
```

- `OpenAIProvider` and a deterministic `NullProvider` (template responses) for tests/self-host.

**Testing**:
- `Unit (NullProvider): complete with json_schema → schema-valid JSON`
- `Integration: ai_model_output persisted with inputs+output+version`

#### 7.2 — Dynamic pricing engine

**What**: Recommend a price/price-band for a listing using supply, demand, logistics, and external indices.

**Design**:
- Numeric model (`scikit-learn` gradient-boosted regressor) trained offline on historical `price_feed` + completed orders; features: commodity, grade, region, season/harvest_year, recent reference price, local supply (active listing volume), demand (recent bid depth), logistics proximity score.
- `ai/pricing_engine.py: recommend_price(listing) -> PriceRecommendation{suggested, low, high, confidence, drivers}`.
- LLM generates a human-readable rationale from `drivers` (advisory text only — never the number).
- `GET /listings/{id}/price-recommendation` (seller-only); also surfaced at listing creation.

**Testing**:
- `Unit (fixed model fixture): known features → expected band; high supply lowers suggestion`
- `Unit: missing reference price → falls back to last platform trade, confidence reduced`
- `Integration: endpoint returns recommendation + rationale; output stored in ai_model_output`

#### 7.3 — AI match engine

**What**: Rank buyer demand against a listing (and vice versa) by quality fit, logistics proximity, and counterparty reliability.

**Design**:
- `ai/match_engine.py: rank_matches(listing, candidates) -> list[MatchScore{org_id, score, factors}]`.
- Score = weighted blend: grade/spec compatibility, distance (PostGIS), reputation (Phase 4), historical trade reliability, price expectation gap. Weights configurable.
- Demand profiles: `buyer_demand_profile` derived from a buyer org's past orders + standing requirements (stored in `organization.attributes`).
- `GET /listings/{id}/matches` (seller), `GET /matches/recommendations` (buyer: best listings for me).

**Testing**:
- `Unit: closer + higher-reputation buyer ranks above distant low-reputation one`
- `Unit: grade mismatch zeroes compatibility factor`
- `Integration: endpoint returns ranked matches with explainable factors`

#### 7.4 — Market intelligence alerts

**What**: Notify users of price thresholds, demand signals, and optimal selling windows.

**Design**:

```sql
CREATE TABLE market_alert (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES user_account(id),
    commodity_id UUID NOT NULL REFERENCES commodity(id),
    alert_type TEXT NOT NULL CHECK (alert_type IN ('price_above','price_below','new_listing','demand_signal','optimal_window')),
    threshold_value NUMERIC,
    currency_code TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_triggered TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- Celery beat evaluates active alerts against latest prices/signals; delivers via in-app notification + WebSocket; `optimal_window` uses pricing engine forecast.

**Testing**:
- `Unit: price crosses threshold → alert fires once until reset`
- `Integration: alert evaluation task triggers notification for matching alert`

### Definition of Done
Pricing recommendations and match rankings available via API with stored, auditable outputs and explainable factors; alerts fire correctly; LLM abstraction swappable; tests pass with deterministic providers.

---

## Phase 8: Quality Verification, Traceability & AI Fraud Detection

### Purpose
Reduce transaction friction and risk: digital grading certificates, GS1 EPCIS 2.0 supply-chain traceability, W3C Verifiable Credentials, plus AI-driven fraud and dispute triage. This builds the trust layer institutional buyers require.

### Tasks

#### 8.1 — Inspections, lots & certifications

**What**: Record quality inspections, production lots, and org certifications.

**Design** (consolidating Suggestion 1 inspection/cert/lot into the hybrid model):

```sql
CREATE TABLE lot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_number TEXT NOT NULL,
    commodity_id UUID NOT NULL REFERENCES commodity(id),
    producer_org_id UUID NOT NULL REFERENCES organization(id),
    origin_location_id UUID REFERENCES location(id),
    harvest_date DATE, harvest_year INTEGER,
    quantity NUMERIC NOT NULL, uom TEXT NOT NULL,
    gtin TEXT, sscc TEXT,
    status TEXT NOT NULL DEFAULT 'available'
        CHECK (status IN ('available','reserved','in_transit','delivered','consumed')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (producer_org_id, lot_number)
);

CREATE TABLE inspection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id UUID REFERENCES listing(id),
    order_id UUID REFERENCES purchase_order(id),
    lot_id UUID REFERENCES lot(id),
    inspector_org_id UUID REFERENCES organization(id),
    inspection_type TEXT NOT NULL CHECK (inspection_type IN ('pre_sale','loading','discharge','delivery')),
    grading_system TEXT NOT NULL,
    assigned_grade TEXT NOT NULL,
    measurements JSONB NOT NULL DEFAULT '{}',   -- {test_weight, moisture_pct, damage_pct, aflatoxin_ppb,...}
    certificate_number TEXT UNIQUE,
    certificate_url TEXT,
    inspected_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE certification (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    cert_type TEXT NOT NULL CHECK (cert_type IN
        ('organic','fair_trade','iso_22000','haccp','global_gap','sps_export','phytosanitary','halal','kosher')),
    cert_number TEXT, issuing_body TEXT NOT NULL,
    issued_date DATE NOT NULL, expiry_date DATE,
    status TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','expired','revoked','suspended')),
    document_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- Endpoints to create inspections/lots/certs; listings can reference a `lot_id` and surface inspection grade.

**Testing**:
- `Integration: attach inspection to listing → grade visible on listing detail`
- `Unit: duplicate lot_number per producer → unique violation`
- `Integration: expired cert excluded from active-cert listing badges`

#### 8.2 — GS1 EPCIS 2.0 supply-chain events

**What**: Capture critical tracking events (receiving, packing, shipping, delivery) per GS1 EPCIS 2.0.

**Design**:

```sql
CREATE TABLE supply_chain_event (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_id UUID NOT NULL REFERENCES lot(id),
    order_id UUID REFERENCES purchase_order(id),
    epcis_event_type TEXT NOT NULL CHECK (epcis_event_type IN ('object','aggregation','transformation','transaction','association')),
    action TEXT NOT NULL CHECK (action IN ('ADD','OBSERVE','DELETE')),
    biz_step TEXT NOT NULL,            -- urn:epcglobal:cbv:bizstep:shipping
    disposition TEXT,
    read_point_gln TEXT, biz_location_gln TEXT,
    source_org_id UUID REFERENCES organization(id),
    dest_org_id UUID REFERENCES organization(id),
    quantity NUMERIC, uom TEXT,
    sensor_data JSONB,                 -- {temperature_c, humidity_pct, device_id}
    event_time TIMESTAMPTZ NOT NULL,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sce_lot ON supply_chain_event(lot_id, event_time);
```

- `integrations/epcis.py`: import/export EPCIS 2.0 JSON-LD; `POST /epcis/events` (ingest from logistics partners), `GET /lots/{id}/trace` → ordered chain.

**Testing**:
- `Unit: EPCIS 2.0 JSON-LD payload → supply_chain_event rows with correct biz_step/GLN`
- `Integration: GET /lots/{id}/trace → chronological CTE chain`
- `Fixture: exported events validate against EPCIS 2.0 schema`

#### 8.3 — W3C Verifiable Credentials for certificates

**What**: Issue and verify tamper-evident digital quality/organic/export certificates.

**Design**:
- `integrations/vc.py`: `issue_credential(subject, claims) -> VerifiableCredential` (VC Data Model 2.0, signed with platform key), `verify_credential(vc) -> VerificationResult`.
- Inspection certificates and org certifications can be issued as VCs; `GET /certs/{id}/vc` returns the credential; `POST /vc/verify` checks any presented VC.

**Testing**:
- `Unit: issue then verify → valid; tampered claim → invalid`
- `Integration: issue VC from inspection → verifiable, subject claims match`

#### 8.4 — AI fraud & dispute triage

**What**: Flag anomalous listings/orders and triage disputes.

**Design**:
- `ai/fraud.py: score_transaction(order|listing) -> FraudSignal{score, reasons}` using rules + anomaly model: rapid price changes (from `price_feed`/event history), grade/price mismatch vs reference, new counterparty with high value, certificate inconsistencies, abnormal bid patterns.
- Dispute model:

```sql
CREATE TABLE dispute (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES purchase_order(id),
    raised_by_org_id UUID NOT NULL REFERENCES organization(id),
    dispute_type TEXT NOT NULL CHECK (dispute_type IN ('quality','quantity','delivery','payment','fraud','other')),
    description TEXT NOT NULL,
    evidence_urls TEXT[],
    ai_triage JSONB,                   -- {suggested_resolution, confidence, rationale}
    status TEXT NOT NULL DEFAULT 'open'
        CHECK (status IN ('open','under_review','mediation','resolved_buyer','resolved_seller','escalated','closed')),
    resolved_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- Raising a dispute sets order `disputed`, freezes escrow; AI produces a triage suggestion (LLM over evidence + inspection + trace).
- High fraud score auto-flags a listing/order for review and notifies admins.

**Testing**:
- `Unit: listing priced far below reference by new org → high fraud score with reasons`
- `Integration: raise dispute → order disputed, escrow frozen, ai_triage populated`
- `Integration: resolve dispute (resolved_buyer) → escrow refunded`

### Definition of Done
Inspections/lots/certs recorded; EPCIS trace import/export schema-valid; VCs issue/verify; fraud scoring and dispute triage operational with escrow interactions; tests pass.

---

## Phase 9: Embedded Trade Finance & AI Underwriting

### Purpose
Add the embedded supply-chain finance differentiator: lenders extend short-term facilities (invoice factoring, crop advances) underwritten by AI from production and marketplace transaction history. This unlocks liquidity for farmers awaiting payment.

### Tasks

#### 9.1 — Finance facility & drawdown model

**What**: Credit facilities and drawdowns against orders/invoices.

**Design**:

```sql
CREATE TABLE trade_finance (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    borrower_org_id UUID NOT NULL REFERENCES organization(id),
    lender_org_id UUID NOT NULL REFERENCES organization(id),
    facility_type TEXT NOT NULL CHECK (facility_type IN ('invoice_factoring','reverse_factoring','crop_advance','warehouse_receipt')),
    credit_limit NUMERIC NOT NULL CHECK (credit_limit > 0),
    currency_code TEXT NOT NULL,
    interest_rate_bps INTEGER,
    term_days INTEGER,
    collateral JSONB NOT NULL DEFAULT '{}',
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','active','exhausted','suspended','closed')),
    approved_at TIMESTAMPTZ, expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE finance_drawdown (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id UUID NOT NULL REFERENCES trade_finance(id),
    order_id UUID REFERENCES purchase_order(id),
    invoice_number TEXT,
    amount NUMERIC NOT NULL CHECK (amount > 0),
    currency_code TEXT NOT NULL,
    disbursed_at TIMESTAMPTZ, repayment_due DATE, repaid_at TIMESTAMPTZ,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','disbursed','repaid','defaulted','written_off')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- Drawdown total per facility cannot exceed `credit_limit` (enforced in service).

**Testing**:
- `Integration: drawdowns exceeding credit_limit → 422`
- `Integration: full repayment → drawdown repaid; facility usage restored`

#### 9.2 — AI underwriting

**What**: Score a borrower's creditworthiness and recommend facility terms.

**Design**:
- `ai/underwriter.py: assess(borrower_org, requested) -> CreditDecision{approved, limit, rate_bps, term_days, score, rationale}`.
- Features: completed-order history/value, on-time delivery %, dispute rate, reputation, production capacity (lots), commodity price volatility, seasonality.
- LLM summarises rationale; numeric model produces the decision. Output persisted to `ai_model_output` (model_kind `underwriting`).
- `POST /finance/applications` → triggers assessment; lender sees recommendation, accepts/overrides.

**Testing**:
- `Unit: strong history → higher limit, lower rate than thin-file org`
- `Unit: high dispute rate → declined or reduced limit`
- `Integration: application → CreditDecision stored, facility created on lender approval`

#### 9.3 — Finance lifecycle & repayment

**What**: Disburse against escrow/invoice and reconcile repayment on settlement.

**Design**:
- On qualifying order, borrower requests drawdown; funds disbursed via payment gateway; repayment auto-collected from buyer settlement at order completion (links to Phase 5 escrow release).
- Default detection (Celery beat) when `repayment_due` passes unpaid → status `defaulted`, lender notified, borrower reputation impacted.

**Testing**:
- `Integration: drawdown disbursed → borrower paid early; on settlement repayment recorded`
- `Integration: overdue drawdown → marked defaulted, notification sent`

### Definition of Done
Facilities and drawdowns enforce limits; AI underwriting produces auditable decisions; disbursement and repayment reconcile with settlement; defaults detected; tests pass.

---

## Phase 10: Frontend Application

### Purpose
Provide the buyer/seller web experience: discovery, listing management, live prices, order management, dashboards, and finance. This makes the platform usable by non-developers.

### Tasks

#### 10.1 — App shell, auth & API client

**What**: Next.js shell with auth and a typed API client generated from OpenAPI.

**Design**:
- `lib/api-client.ts` generated from `openapi/agrimarket.openapi.json`.
- Auth context handling login (local + OIDC redirect), token storage (httpOnly cookie via route handler), org switcher.
- Layout: nav, org switcher, notifications bell (WebSocket).

**Testing**:
- `Component (Vitest): login form submits and stores session`
- `E2E (Playwright): unauthenticated user redirected to login`

#### 10.2 — Marketplace discovery & listing detail

**What**: Search UI with filters, map/proximity, and listing detail with bid placement.

**Design**:
- Listing grid with filters (commodity, grade, price, country, organic, delivery terms), sort, cursor pagination.
- Detail page: specs, inspection grade/certs, seller reputation, live price chart (WebSocket), bid form.

**Testing**:
- `E2E: filter by commodity + price → results update`
- `E2E: place bid as buyer → bid appears, seller notified`

#### 10.3 — Seller & order management dashboards

**What**: Create/manage listings (with AI price recommendation) and track orders.

**Design**:
- Listing wizard surfacing `GET /listings/{id}/price-recommendation` with rationale.
- Order board grouped by status; actions: accept bid, confirm delivery, raise dispute, leave review.
- Buyer match recommendations panel.

**Testing**:
- `E2E: create listing with recommended price → published, visible in marketplace`
- `E2E: seller accepts bid → order created and shown on board`

#### 10.4 — Prices, finance & analytics

**What**: Price charts, alert management, finance applications, reputation/analytics.

**Design**:
- Commodity price pages (history + live), alert CRUD.
- Finance: apply for facility, view decision, drawdown/repayment status.
- Org analytics: trade volume, avg rating, on-time %, dispute rate.

**Testing**:
- `E2E: create price alert → fires and shows notification when threshold crossed (seeded)`
- `E2E: submit finance application → decision displayed`

### Definition of Done
End-to-end web flows (discover → bid → order → settle → review; create listing; prices/alerts; finance) work against the live API; Playwright suite green; built artifact served via compose.

---

## Phase 11: MCP Server, Public API Hardening & Compliance

### Purpose
Expose the platform to AI agents via MCP, finalise the public OpenAPI surface with rate limiting and pagination guarantees, and close compliance gaps (audit log, GDPR, OWASP API Top 10).

### Tasks

#### 11.1 — MCP server

**What**: MCP server exposing platform data/tools to AI agents per the 2025-11-25 spec.

**Design**:
- `mcp/server.py` tools: `search_listings`, `get_price(commodity, region)`, `get_price_history`, `get_org_reputation`, `recommend_price(listing_id)`, `find_matches(listing_id)`, `get_market_alerts`. Read-only by default; mutations gated behind scoped tokens.
- Auth via platform JWT/OAuth scope; each tool maps to existing service functions (no business logic duplication).

**Testing**:
- `Integration (MCP client): list tools → expected catalogue with schemas`
- `Integration: call search_listings with filters → results match REST endpoint`
- `Integration: unauthorised mutation tool → denied`

#### 11.2 — API hardening (OWASP API Top 10)

**What**: Rate limiting, consistent pagination, input hardening, security headers.

**Design**:
- Redis token-bucket rate limiting per user/IP/endpoint class (stricter on auth/payment/finance).
- Enforce object-level auth audit across all routes (BOLA); add automated test sweep asserting cross-org access is denied.
- Security headers; request size limits; strict CORS allowlist.

**Testing**:
- `Integration: exceed rate limit → 429 with Retry-After`
- `Integration (sweep): every resource-by-id route rejects cross-org caller → 403`
- `Integration: oversized payload → 413`

#### 11.3 — Audit log & compliance

**What**: Immutable audit trail and GDPR data-subject tooling.

**Design**:

```sql
CREATE TABLE audit_log (
    id UUID DEFAULT gen_random_uuid(),
    actor_user_id UUID REFERENCES user_account(id),
    actor_org_id UUID REFERENCES organization(id),
    action TEXT NOT NULL,              -- 'listing.created','order.accepted','payment.completed'
    resource_type TEXT NOT NULL,
    resource_id UUID NOT NULL,
    changes JSONB,
    ip_address INET, user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

- Middleware/service hooks emit audit entries on all state-changing actions.
- GDPR: `GET /me/export` (data portability), `POST /me/erasure-request` (right to erasure with legal-hold checks for financial records), explicit consent capture on registration.

**Testing**:
- `Integration: order acceptance → audit_log entry with actor, action, changes`
- `Integration: erasure request on org with open orders → blocked (legal hold), reason returned`
- `Integration: data export → machine-readable bundle of the user's data`

#### 11.4 — OpenAPI publication & SDK

**What**: Finalise and publish the OAS3 spec; generate a client SDK.

**Design**:
- Commit `openapi/agrimarket.openapi.json` (OpenAPI 3.1); CI fails if drift vs running app.
- Generate Python + TypeScript SDKs; document auth, pagination (RFC 8288 Link headers), and error format (RFC 7807).

**Testing**:
- `CI: generated spec matches app routes (no drift)`
- `Integration: generated SDK round-trips a listing create/read`

### Definition of Done
MCP server live and authorised; rate limiting + BOLA sweep pass; audit log captures all mutations; GDPR export/erasure work with legal holds; OpenAPI published and SDKs generated; full suite green; `docker compose up` brings up the complete platform.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Skeleton            ─── required by everything
    │
Phase 2: Identity & Auth                  ─── requires 1
    │
Phase 3: Catalogue & Listings             ─── requires 2
    │
Phase 4: Bidding, Orders & Reviews        ─── requires 3
    │
Phase 5: Payments, Escrow & Settlement    ─── requires 4
    │
    ├── Phase 6: Real-Time Prices & Feeds  ─── requires 3 (can parallel with 5)
    │       │
    │   Phase 7: AI Pricing & Matching     ─── requires 4 + 6
    │
    ├── Phase 8: Quality, Trace & Fraud    ─── requires 4 (+5 for escrow freeze); can parallel with 6/7
    │
    └── Phase 9: Trade Finance & Underwrite─── requires 5 + 4 (+8 reputation signals)

Phase 10: Frontend                        ─── requires 2–9 incrementally (per-feature)
Phase 11: MCP, Hardening & Compliance     ─── requires all backend phases (3–9)
```

**Parallelism opportunities:**
- After Phase 4: **Phase 5 (payments)** and **Phase 6 (price feeds)** can be built concurrently.
- After Phases 4–5: **Phase 8 (quality/fraud)** can be built concurrently with **Phase 6/7 (prices/AI)**.
- **Phase 10 (frontend)** can begin per-feature as each backend phase stabilises (discovery after Phase 3, orders after Phase 4, etc.).
- **Phase 7** depends on both the transactional data (Phase 4) and the price feeds (Phase 6).

---

## Definition of Done (per phase)

Every phase must satisfy this checklist before it is considered complete:

1. All tasks implemented.
2. All unit and mocked-integration tests pass; real-dependency (testcontainers) tests pass in CI.
3. Ruff lint + format pass; mypy (strict) passes.
4. Alembic migration(s) created and `upgrade head`/`downgrade base` round-trip cleanly.
5. Docker image builds; `docker compose up` brings the new capability online.
6. The feature works end-to-end (demonstrated by an integration or E2E test).
7. New configuration options documented in `.env.example` and README.
8. New/changed API endpoints appear in the generated OpenAPI 3.1 spec with request/response schemas.
9. State-changing actions emit audit-log entries (from Phase 11 onward; stubbed earlier).
10. Relevant standards honoured and referenced (GIPSA/Codex grades, Incoterms 2020, ISO 20022, GS1 EPCIS 2.0, W3C VC, OAuth2/OIDC/JWT, OWASP API Top 10).
```
