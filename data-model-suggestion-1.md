# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: AgriMarket Platform · Created: 2026-05-24

## Philosophy

This model follows classical relational database design principles: every real-world concept gets its own table, foreign keys enforce referential integrity, and junction tables handle many-to-many relationships. The schema is heavily normalized (3NF+) to eliminate data redundancy, ensure consistency, and make complex cross-entity queries straightforward.

The approach mirrors how exchanges like CME Group and government systems like USDA AMS structure their data -- separate, well-defined entities for participants, instruments, orders, trades, and settlements, each with strict type constraints. Regulatory bodies and auditors can query any slice of the data with standard SQL joins, and every relationship is explicit in the schema rather than encoded in application logic.

This is the safest choice for a platform that must comply with GIPSA grain grading standards, Codex Alimentarius food safety requirements, ISO 20022 payment messaging, and GS1 traceability mandates -- each standard maps cleanly to dedicated tables with standardized identifier columns.

**Best for:** Teams building a compliance-first platform that must satisfy regulators, auditors, and institutional buyers who expect well-structured, queryable data with full referential integrity.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Clean mapping to industry standards (GS1, ISO 20022, GIPSA grades)
- (+) Straightforward SQL queries for reporting and analytics
- (+) Well-understood by most development teams
- (-) High table count (~55-65 tables) increases migration complexity
- (-) Schema changes require DDL migrations for every new commodity attribute
- (-) Junction tables add query complexity for many-to-many relationships
- (-) Less flexible for jurisdiction-specific fields that vary by country

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GIPSA / USDA Grain Standards | `commodity_grade` table stores official US grade designations, test weight, moisture, damage factors per FGIS handbook |
| Codex Alimentarius | `commodity_category` uses Codex classification codes for food/feed groupings |
| GS1 GTS2 / EPCIS 2.0 | `traceability_event` table models Critical Tracking Events (CTEs) with GTIN and GLN identifiers |
| ISO 20022 | `payment_instruction` table maps to pacs.008 credit transfer message fields |
| ISO 22005 | `lot` and `lot_provenance` tables implement feed/food chain traceability |
| ISO 3166 | `jurisdiction` table uses ISO 3166-1 alpha-2 country codes and 3166-2 subdivision codes |
| FIX Protocol | `order` table field naming aligns with FIX tag conventions (side, order_type, time_in_force) |
| GS1 GTIN | `commodity_listing.gtin` column stores Global Trade Item Numbers for product identification |
| OAuth 2.0 / OIDC | `user_session` and `api_credential` tables support token-based auth flows |

---

## Participant Management

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_name      TEXT NOT NULL,
    trade_name      TEXT,
    org_type        TEXT NOT NULL CHECK (org_type IN ('farm', 'cooperative', 'processor', 'exporter', 'importer', 'trader', 'lender', 'government', 'logistics_provider')),
    registration_number TEXT,              -- national business registration
    lei             TEXT CHECK (length(lei) = 20), -- ISO 17442 Legal Entity Identifier
    gln             TEXT CHECK (length(gln) = 13), -- GS1 Global Location Number
    tax_id          TEXT,
    jurisdiction_id UUID NOT NULL REFERENCES jurisdiction(id),
    verification_status TEXT NOT NULL DEFAULT 'pending' CHECK (verification_status IN ('pending', 'verified', 'suspended', 'rejected')),
    verified_at     TIMESTAMPTZ,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organization_type ON organization(org_type);
CREATE INDEX idx_organization_jurisdiction ON organization(jurisdiction_id);
CREATE INDEX idx_organization_lei ON organization(lei) WHERE lei IS NOT NULL;
CREATE INDEX idx_organization_gln ON organization(gln) WHERE gln IS NOT NULL;

CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    phone           TEXT,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,                  -- null for SSO-only users
    identity_provider TEXT DEFAULT 'local', -- 'local', 'google', 'microsoft', 'gov_id'
    identity_provider_sub TEXT,            -- external IdP subject identifier
    preferred_locale TEXT DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_user_idp ON user_account(identity_provider, identity_provider_sub)
    WHERE identity_provider != 'local';

CREATE TABLE organization_membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES user_account(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'trader', 'viewer', 'finance')),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    UNIQUE (user_id, organization_id, role)
);

CREATE INDEX idx_orgmember_org ON organization_membership(organization_id);
CREATE INDEX idx_orgmember_user ON organization_membership(user_id);

CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    TEXT NOT NULL CHECK (length(country_code) = 2), -- ISO 3166-1 alpha-2
    subdivision_code TEXT,                 -- ISO 3166-2 (e.g., 'US-IA' for Iowa)
    name            TEXT NOT NULL,
    currency_code   TEXT NOT NULL CHECK (length(currency_code) = 3), -- ISO 4217
    timezone        TEXT NOT NULL,         -- IANA timezone
    regulatory_body TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_jurisdiction_codes ON jurisdiction(country_code, subdivision_code);
```

---

## Commodity Catalogue

```sql
CREATE TABLE commodity_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codex_code      TEXT UNIQUE,           -- Codex Alimentarius classification code
    gpc_code        TEXT,                  -- GS1 Global Product Classification code
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES commodity_category(id),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commodity_cat_parent ON commodity_category(parent_id);

CREATE TABLE commodity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_id     UUID NOT NULL REFERENCES commodity_category(id),
    code            TEXT NOT NULL UNIQUE,   -- internal commodity code (e.g., 'CORN_YEL_2')
    name            TEXT NOT NULL,
    variety         TEXT,                   -- e.g., 'Yellow Dent', 'Hard Red Winter'
    unit_of_measure TEXT NOT NULL CHECK (unit_of_measure IN ('mt', 'bushel', 'cwt', 'kg', 'lb', 'ton')),
    hs_code         TEXT,                   -- Harmonized System tariff code for cross-border trade
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commodity_category ON commodity(category_id);

CREATE TABLE commodity_grade (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    grading_system  TEXT NOT NULL CHECK (grading_system IN ('GIPSA', 'Codex', 'EU', 'national', 'custom')),
    grade_code      TEXT NOT NULL,          -- e.g., 'US_1', 'US_2', 'US_SG' (Sample Grade)
    grade_name      TEXT NOT NULL,          -- e.g., 'U.S. No. 1 Yellow Corn'
    min_test_weight NUMERIC,               -- minimum test weight (lb/bu or kg/hl)
    max_moisture    NUMERIC,               -- maximum moisture percentage
    max_damage      NUMERIC,               -- maximum total damage percentage
    max_foreign_material NUMERIC,          -- maximum foreign material percentage
    max_broken      NUMERIC,               -- maximum broken kernels percentage
    specifications  JSONB DEFAULT '{}',    -- additional grade-specific quality factors
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (commodity_id, grading_system, grade_code)
);

CREATE INDEX idx_grade_commodity ON commodity_grade(commodity_id);
```

---

## Marketplace Listings & Orders

```sql
CREATE TABLE commodity_listing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_org_id   UUID NOT NULL REFERENCES organization(id),
    created_by      UUID NOT NULL REFERENCES user_account(id),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    grade_id        UUID REFERENCES commodity_grade(id),
    gtin            TEXT,                   -- GS1 Global Trade Item Number
    listing_type    TEXT NOT NULL CHECK (listing_type IN ('spot', 'forward', 'auction')),
    quantity        NUMERIC NOT NULL CHECK (quantity > 0),
    unit_of_measure TEXT NOT NULL,
    price_per_unit  NUMERIC,               -- null for auction listings
    currency_code   TEXT NOT NULL CHECK (length(currency_code) = 3),
    price_type      TEXT NOT NULL CHECK (price_type IN ('fixed', 'basis', 'negotiable', 'auction')),
    basis_reference TEXT,                   -- e.g., 'CBOT_ZC_2026_07' for basis pricing
    basis_offset    NUMERIC,               -- cents per bushel above/below reference
    origin_jurisdiction_id UUID REFERENCES jurisdiction(id),
    harvest_year    INTEGER,
    harvest_date    DATE,
    available_from  DATE NOT NULL,
    available_until DATE,
    delivery_terms  TEXT CHECK (delivery_terms IN ('FOB', 'CIF', 'DAP', 'FCA', 'EXW')), -- Incoterms 2020
    delivery_location_id UUID REFERENCES location(id),
    organic_certified BOOLEAN DEFAULT false,
    certifications  TEXT[],                -- array of certification names
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('draft', 'active', 'matched', 'expired', 'cancelled', 'sold')),
    views_count     INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_listing_seller ON commodity_listing(seller_org_id);
CREATE INDEX idx_listing_commodity ON commodity_listing(commodity_id);
CREATE INDEX idx_listing_status ON commodity_listing(status) WHERE status = 'active';
CREATE INDEX idx_listing_available ON commodity_listing(available_from, available_until);
CREATE INDEX idx_listing_commodity_status ON commodity_listing(commodity_id, status, price_per_unit);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organization(id),
    gln             TEXT,                   -- GS1 Global Location Number
    name            TEXT NOT NULL,
    location_type   TEXT NOT NULL CHECK (location_type IN ('farm', 'warehouse', 'elevator', 'port', 'mill', 'processing_plant', 'delivery_point')),
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    TEXT NOT NULL CHECK (length(country_code) = 2),
    latitude        NUMERIC,
    longitude       NUMERIC,
    capacity_mt     NUMERIC,               -- storage capacity in metric tonnes
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_org ON location(organization_id);
CREATE INDEX idx_location_type ON location(location_type);
CREATE INDEX idx_location_geo ON location USING gist (
    point(longitude, latitude)
) WHERE latitude IS NOT NULL AND longitude IS NOT NULL;

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number    TEXT NOT NULL UNIQUE,
    buyer_org_id    UUID NOT NULL REFERENCES organization(id),
    listing_id      UUID NOT NULL REFERENCES commodity_listing(id),
    seller_org_id   UUID NOT NULL REFERENCES organization(id),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    grade_id        UUID REFERENCES commodity_grade(id),
    quantity        NUMERIC NOT NULL CHECK (quantity > 0),
    unit_of_measure TEXT NOT NULL,
    agreed_price    NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    delivery_terms  TEXT,
    delivery_date   DATE,
    delivery_location_id UUID REFERENCES location(id),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'in_transit', 'delivered', 'completed', 'disputed', 'cancelled')),
    accepted_at     TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_buyer ON purchase_order(buyer_org_id);
CREATE INDEX idx_order_seller ON purchase_order(seller_org_id);
CREATE INDEX idx_order_listing ON purchase_order(listing_id);
CREATE INDEX idx_order_status ON purchase_order(status);

CREATE TABLE bid (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES commodity_listing(id),
    buyer_org_id    UUID NOT NULL REFERENCES organization(id),
    created_by      UUID NOT NULL REFERENCES user_account(id),
    bid_price       NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    quantity        NUMERIC NOT NULL CHECK (quantity > 0),
    message         TEXT,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'rejected', 'countered', 'expired', 'withdrawn')),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bid_listing ON bid(listing_id);
CREATE INDEX idx_bid_buyer ON bid(buyer_org_id);
CREATE INDEX idx_bid_status ON bid(status);
```

---

## Quality Inspection & Certification

```sql
CREATE TABLE quality_inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID REFERENCES commodity_listing(id),
    order_id        UUID REFERENCES purchase_order(id),
    lot_id          UUID REFERENCES lot(id),
    inspector_org_id UUID REFERENCES organization(id),
    inspector_name  TEXT,
    inspection_type TEXT NOT NULL CHECK (inspection_type IN ('pre_sale', 'loading', 'discharge', 'delivery')),
    grading_system  TEXT NOT NULL,
    assigned_grade  TEXT NOT NULL,
    test_weight     NUMERIC,
    moisture_pct    NUMERIC,
    damage_pct      NUMERIC,
    foreign_material_pct NUMERIC,
    protein_pct     NUMERIC,
    oil_pct         NUMERIC,
    aflatoxin_ppb   NUMERIC,               -- mycotoxin level
    inspection_date TIMESTAMPTZ NOT NULL,
    certificate_number TEXT UNIQUE,
    certificate_url TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_listing ON quality_inspection(listing_id);
CREATE INDEX idx_inspection_order ON quality_inspection(order_id);
CREATE INDEX idx_inspection_lot ON quality_inspection(lot_id);

CREATE TABLE certification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    cert_type       TEXT NOT NULL CHECK (cert_type IN ('organic', 'fair_trade', 'iso_22000', 'haccp', 'global_gap', 'rainforest_alliance', 'sps_export', 'phytosanitary', 'halal', 'kosher')),
    cert_number     TEXT,
    issuing_body    TEXT NOT NULL,
    issued_date     DATE NOT NULL,
    expiry_date     DATE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'expired', 'revoked', 'suspended')),
    document_url    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cert_org ON certification(organization_id);
CREATE INDEX idx_cert_type ON certification(cert_type);
CREATE INDEX idx_cert_expiry ON certification(expiry_date) WHERE status = 'active';
```

---

## Traceability & Logistics

```sql
CREATE TABLE lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_number      TEXT NOT NULL,
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    producer_org_id UUID NOT NULL REFERENCES organization(id),
    origin_location_id UUID REFERENCES location(id),
    harvest_date    DATE,
    harvest_year    INTEGER,
    quantity        NUMERIC NOT NULL,
    unit_of_measure TEXT NOT NULL,
    gtin            TEXT,                   -- GS1 GTIN for this lot
    sscc            TEXT,                   -- GS1 Serial Shipping Container Code
    status          TEXT NOT NULL DEFAULT 'available' CHECK (status IN ('available', 'reserved', 'in_transit', 'delivered', 'consumed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (producer_org_id, lot_number)
);

CREATE INDEX idx_lot_commodity ON lot(commodity_id);
CREATE INDEX idx_lot_producer ON lot(producer_org_id);

-- GS1 EPCIS 2.0 aligned traceability events
CREATE TABLE traceability_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      TEXT NOT NULL CHECK (event_type IN ('object', 'aggregation', 'transformation', 'transaction', 'association')),
    action          TEXT NOT NULL CHECK (action IN ('ADD', 'OBSERVE', 'DELETE')),
    biz_step        TEXT NOT NULL,          -- EPCIS business step URN (e.g., 'urn:epcglobal:cbv:bizstep:shipping')
    disposition     TEXT,                   -- EPCIS disposition URN
    lot_id          UUID NOT NULL REFERENCES lot(id),
    order_id        UUID REFERENCES purchase_order(id),
    read_point_gln  TEXT,                   -- GS1 GLN of the read point location
    biz_location_gln TEXT,                  -- GS1 GLN of the business location
    source_org_id   UUID REFERENCES organization(id),
    dest_org_id     UUID REFERENCES organization(id),
    quantity        NUMERIC,
    unit_of_measure TEXT,
    event_time      TIMESTAMPTZ NOT NULL,
    record_time     TIMESTAMPTZ NOT NULL DEFAULT now(),
    sensor_data     JSONB,                  -- temperature, humidity readings per EPCIS 2.0 sensor element
    -- Example sensor_data:
    -- {"temperature_c": 22.5, "humidity_pct": 45.2, "device_id": "SENS-001"}
    extensions      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trace_lot ON traceability_event(lot_id);
CREATE INDEX idx_trace_order ON traceability_event(order_id);
CREATE INDEX idx_trace_time ON traceability_event(event_time);
CREATE INDEX idx_trace_bizstep ON traceability_event(biz_step);

CREATE TABLE shipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    carrier_org_id  UUID REFERENCES organization(id),
    carrier_name    TEXT,
    tracking_number TEXT,
    transport_mode  TEXT CHECK (transport_mode IN ('truck', 'rail', 'vessel', 'barge', 'container')),
    origin_location_id UUID REFERENCES location(id),
    dest_location_id UUID REFERENCES location(id),
    estimated_departure TIMESTAMPTZ,
    actual_departure TIMESTAMPTZ,
    estimated_arrival TIMESTAMPTZ,
    actual_arrival  TIMESTAMPTZ,
    weight_mt       NUMERIC,
    status          TEXT NOT NULL DEFAULT 'planned' CHECK (status IN ('planned', 'in_transit', 'delayed', 'delivered', 'cancelled')),
    bill_of_lading  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shipment_order ON shipment(order_id);
CREATE INDEX idx_shipment_status ON shipment(status);
```

---

## Payments & Trade Finance

```sql
CREATE TABLE payment_instruction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    payer_org_id    UUID NOT NULL REFERENCES organization(id),
    payee_org_id    UUID NOT NULL REFERENCES organization(id),
    amount          NUMERIC NOT NULL CHECK (amount > 0),
    currency_code   TEXT NOT NULL CHECK (length(currency_code) = 3), -- ISO 4217
    payment_method  TEXT NOT NULL CHECK (payment_method IN ('bank_transfer', 'escrow', 'letter_of_credit', 'mobile_money', 'platform_wallet')),
    -- ISO 20022 pacs.008 aligned fields
    instruction_id  TEXT UNIQUE,            -- InstrId
    end_to_end_id   TEXT,                   -- EndToEndId
    payer_iban      TEXT,
    payee_iban      TEXT,
    payer_bic       TEXT CHECK (length(payer_bic) IN (8, 11)), -- SWIFT BIC
    payee_bic       TEXT CHECK (length(payee_bic) IN (8, 11)),
    remittance_info TEXT,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded')),
    due_date        DATE,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_order ON payment_instruction(order_id);
CREATE INDEX idx_payment_payer ON payment_instruction(payer_org_id);
CREATE INDEX idx_payment_payee ON payment_instruction(payee_org_id);
CREATE INDEX idx_payment_status ON payment_instruction(status);

CREATE TABLE escrow_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    buyer_org_id    UUID NOT NULL REFERENCES organization(id),
    seller_org_id   UUID NOT NULL REFERENCES organization(id),
    amount          NUMERIC NOT NULL CHECK (amount > 0),
    currency_code   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'funded' CHECK (status IN ('pending', 'funded', 'released', 'refunded', 'disputed')),
    funded_at       TIMESTAMPTZ,
    released_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_escrow_order ON escrow_account(order_id);

CREATE TABLE trade_finance_facility (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    borrower_org_id UUID NOT NULL REFERENCES organization(id),
    lender_org_id   UUID NOT NULL REFERENCES organization(id),
    facility_type   TEXT NOT NULL CHECK (facility_type IN ('invoice_factoring', 'reverse_factoring', 'crop_advance', 'warehouse_receipt')),
    credit_limit    NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    interest_rate_bps INTEGER,             -- basis points
    term_days       INTEGER,
    collateral_type TEXT,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('pending', 'active', 'exhausted', 'suspended', 'closed')),
    approved_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_finance_borrower ON trade_finance_facility(borrower_org_id);
CREATE INDEX idx_finance_lender ON trade_finance_facility(lender_org_id);

CREATE TABLE finance_drawdown (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES trade_finance_facility(id),
    order_id        UUID REFERENCES purchase_order(id),
    invoice_number  TEXT,
    amount          NUMERIC NOT NULL CHECK (amount > 0),
    currency_code   TEXT NOT NULL,
    disbursed_at    TIMESTAMPTZ,
    repayment_due   DATE,
    repaid_at       TIMESTAMPTZ,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'disbursed', 'repaid', 'defaulted', 'written_off')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drawdown_facility ON finance_drawdown(facility_id);
CREATE INDEX idx_drawdown_order ON finance_drawdown(order_id);
```

---

## Ratings, Reviews & Disputes

```sql
CREATE TABLE review (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    reviewer_org_id UUID NOT NULL REFERENCES organization(id),
    reviewee_org_id UUID NOT NULL REFERENCES organization(id),
    reviewer_role   TEXT NOT NULL CHECK (reviewer_role IN ('buyer', 'seller')),
    overall_rating  SMALLINT NOT NULL CHECK (overall_rating BETWEEN 1 AND 5),
    quality_rating  SMALLINT CHECK (quality_rating BETWEEN 1 AND 5),
    reliability_rating SMALLINT CHECK (reliability_rating BETWEEN 1 AND 5),
    communication_rating SMALLINT CHECK (communication_rating BETWEEN 1 AND 5),
    comment         TEXT,
    is_public       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (order_id, reviewer_org_id)
);

CREATE INDEX idx_review_reviewee ON review(reviewee_org_id);
CREATE INDEX idx_review_order ON review(order_id);

CREATE TABLE dispute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    raised_by_org_id UUID NOT NULL REFERENCES organization(id),
    dispute_type    TEXT NOT NULL CHECK (dispute_type IN ('quality', 'quantity', 'delivery', 'payment', 'fraud', 'other')),
    description     TEXT NOT NULL,
    evidence_urls   TEXT[],
    status          TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'under_review', 'mediation', 'resolved_buyer', 'resolved_seller', 'escalated', 'closed')),
    resolution_notes TEXT,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dispute_order ON dispute(order_id);
CREATE INDEX idx_dispute_status ON dispute(status);
```

---

## Price Data & Market Intelligence

```sql
CREATE TABLE price_quote (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    source          TEXT NOT NULL CHECK (source IN ('platform', 'cme', 'usda_ams', 'agmarknet', 'dtn', 'manual')),
    quote_type      TEXT NOT NULL CHECK (quote_type IN ('spot', 'bid', 'ask', 'settlement', 'cash_bid')),
    price           NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    unit_of_measure TEXT NOT NULL,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    location_id     UUID REFERENCES location(id),
    grade_id        UUID REFERENCES commodity_grade(id),
    contract_month  TEXT,                   -- e.g., '2026-07' for futures reference
    quoted_at       TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_price_commodity ON price_quote(commodity_id);
CREATE INDEX idx_price_time ON price_quote(quoted_at DESC);
CREATE INDEX idx_price_source ON price_quote(source, commodity_id, quoted_at DESC);

-- Partitioned by month for time-series performance
-- In production: ALTER TABLE price_quote PARTITION BY RANGE (quoted_at);

CREATE TABLE market_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES user_account(id),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    alert_type      TEXT NOT NULL CHECK (alert_type IN ('price_above', 'price_below', 'new_listing', 'demand_signal')),
    threshold_value NUMERIC,
    currency_code   TEXT,
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_user ON market_alert(user_id);
CREATE INDEX idx_alert_commodity ON market_alert(commodity_id) WHERE is_active = true;
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_user_id   UUID REFERENCES user_account(id),
    actor_org_id    UUID REFERENCES organization(id),
    action          TEXT NOT NULL,          -- e.g., 'listing.created', 'order.accepted', 'payment.completed'
    resource_type   TEXT NOT NULL,          -- e.g., 'commodity_listing', 'purchase_order'
    resource_id     UUID NOT NULL,
    changes         JSONB,                  -- {"field": {"old": ..., "new": ...}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_user_id);
CREATE INDEX idx_audit_time ON audit_log(created_at DESC);

-- Partitioned by month in production
-- ALTER TABLE audit_log PARTITION BY RANGE (created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Participant Management | 4 | organization, user_account, organization_membership, jurisdiction |
| Commodity Catalogue | 3 | commodity_category, commodity, commodity_grade |
| Marketplace & Orders | 4 | commodity_listing, location, purchase_order, bid |
| Quality & Certification | 2 | quality_inspection, certification |
| Traceability & Logistics | 3 | lot, traceability_event, shipment |
| Payments & Finance | 4 | payment_instruction, escrow_account, trade_finance_facility, finance_drawdown |
| Ratings & Disputes | 2 | review, dispute |
| Market Intelligence | 2 | price_quote, market_alert |
| Audit | 1 | audit_log |
| **Total** | **25** | Core tables; production would add ~10 more (notifications, documents, API keys, etc.) |

---

## Key Design Decisions

1. **UUID primary keys throughout** -- enables distributed ID generation, safe for multi-region deployment, and prevents sequential ID enumeration attacks on a marketplace with financial data.

2. **Separate `organization` and `user_account` tables with a junction** -- models the real-world reality that one person may trade on behalf of multiple organizations (a farmer who also runs a cooperative), and one organization has multiple users with different roles.

3. **Grade stored as a reference table, not inline** -- GIPSA and Codex grade definitions are standardized; storing them once and referencing by FK ensures consistency and allows updating grade specifications without touching every listing.

4. **ISO 20022-aligned payment fields** -- `instruction_id`, `end_to_end_id`, BIC, and IBAN columns on `payment_instruction` mean cross-border settlement messages can be generated directly from the table without transformation.

5. **GS1 EPCIS-aligned traceability events** -- the `traceability_event` table mirrors the EPCIS event model (event_type, action, biz_step, disposition, read_point) so data can be exported to or imported from GS1-compliant supply chain partners without mapping layers.

6. **Explicit `lot` table for provenance** -- each lot tracks from harvest to delivery, enabling farm-to-fork traceability required by ISO 22005 and increasingly by EU Digital Product Passport regulations.

7. **Trade finance as a separate domain** -- `trade_finance_facility` and `finance_drawdown` tables model the embedded finance feature (invoice factoring, crop advances) as a first-class concern rather than an afterthought on the payment table.

8. **Time-series partitioning for price data and audit logs** -- `price_quote` and `audit_log` will grow rapidly; range partitioning by month keeps queries fast and enables efficient archival.

9. **Incoterms 2020 delivery terms** -- using standardized trade terms (FOB, CIF, DAP, FCA, EXW) on listings and orders eliminates ambiguity about who bears shipping costs and risk.

10. **Partial indexes for active records** -- indexes like `WHERE status = 'active'` on listings and `WHERE is_active = true` on alerts keep the index small and queries fast for the common case of querying only live data.
