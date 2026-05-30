# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: AgriMarket Platform · Created: 2026-05-24

## Philosophy

This model keeps the structural backbone relational -- participants, orders, payments, and core relationships are enforced by foreign keys and constraints -- but pushes variable, jurisdiction-specific, and rapidly evolving attributes into JSONB columns. The key insight is that an agricultural marketplace spanning multiple countries, commodity types, and regulatory regimes will encounter enormous variation in the details: Indian mandis report modal prices in INR per quintal; US grain elevators quote basis in cents per bushel against CBOT futures; EU shipments require Digital Product Passport metadata. Trying to normalise all of this into typed columns produces an explosion of nullable columns or an unmanageable EAV (Entity-Attribute-Value) pattern.

PostgreSQL's JSONB type provides the escape valve: core fields that every listing shares (commodity, quantity, price, seller) live in typed columns with constraints and indexes. Region-specific quality factors, certification metadata, AI model scores, and custom buyer/seller attributes live in JSONB columns with GIN indexes for containment queries. This hybrid approach is used in production by platforms like Shopify (product metafields), Stripe (metadata on every object), and modern e-commerce marketplaces that must support heterogeneous product catalogues.

The result is a schema that can ship an MVP quickly (fewer tables than a fully normalised model), adapt to new regions and commodity types without DDL migrations, and still provide the relational integrity needed for financial transactions and regulatory reporting.

**Best for:** Teams building a multi-region MVP where commodity attributes vary by jurisdiction and commodity type, regulatory requirements differ by country, and speed to market matters more than theoretical purity. Ideal for startups that expect the data model to evolve rapidly in the first 12-18 months.

**Trade-offs:**
- (+) Faster MVP -- fewer tables, less migration overhead
- (+) Multi-region flexibility without nullable column sprawl
- (+) New commodity attributes added without schema migration
- (+) GIN indexes on JSONB enable efficient queries on variable attributes
- (+) Relational integrity preserved for core financial/transactional data
- (-) JSONB fields lack compile-time type safety -- validation must happen in application code or CHECK constraints
- (-) Complex queries on deeply nested JSONB can be slower than typed column queries
- (-) Reporting tools may struggle with JSONB fields compared to flat columns
- (-) Risk of "JSONB junk drawer" if discipline around JSONB structure is not maintained
- (-) Schema documentation requires extra effort since JSONB structure is not visible in DDL

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GIPSA / USDA Grain Standards | Quality factors stored in `listing.quality_attributes` JSONB with standardised key names |
| Codex Alimentarius | `commodity.codex_code` column; Codex-specific attributes in `commodity.extensions` JSONB |
| GS1 EPCIS 2.0 | Traceability events in `supply_chain_event` with EPCIS fields as JSONB `event_data` |
| ISO 20022 | Payment `settlement_data` JSONB carries ISO 20022 message identifiers when applicable |
| ISO 3166 | `organization.country_code` and `listing.origin` JSONB use ISO 3166 codes |
| ISO 22005 | Lot provenance chain tracked via `supply_chain_event` linked to `lot` |
| FIX Protocol | Order fields align with FIX conventions; exchange-specific fields in `order.exchange_data` JSONB |
| GS1 GTIN/GLN | Stored as typed columns on `listing` and `location` where available |

---

## Core Identity & Participants

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_name      TEXT NOT NULL,
    trade_name      TEXT,
    org_type        TEXT NOT NULL CHECK (org_type IN (
        'farm', 'cooperative', 'processor', 'exporter',
        'importer', 'trader', 'lender', 'government', 'logistics'
    )),
    country_code    TEXT NOT NULL CHECK (length(country_code) = 2), -- ISO 3166-1
    subdivision     TEXT,                   -- ISO 3166-2
    currency_code   TEXT NOT NULL CHECK (length(currency_code) = 3), -- ISO 4217
    gln             TEXT,                   -- GS1 Global Location Number
    lei             TEXT,                   -- ISO 17442 Legal Entity Identifier
    tax_id          TEXT,
    verification_status TEXT NOT NULL DEFAULT 'pending',
    -- Jurisdiction-specific registration details vary by country
    registration    JSONB DEFAULT '{}',
    -- Example registration (India):
    -- {
    --   "pan": "ABCDE1234F",
    --   "gstin": "22AAAAA0000A1Z5",
    --   "mandi_license": "ML-2026-MH-042",
    --   "apmc_market_id": "MH-042"
    -- }
    -- Example registration (US):
    -- {
    --   "ein": "12-3456789",
    --   "usda_license": "USDA-WH-2026-001",
    --   "state_ag_license": "IA-AG-2026-0042"
    -- }
    preferences     JSONB DEFAULT '{}',     -- notification prefs, display settings
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_type ON organization(org_type);
CREATE INDEX idx_org_country ON organization(country_code);
CREATE INDEX idx_org_gln ON organization(gln) WHERE gln IS NOT NULL;
CREATE INDEX idx_org_registration ON organization USING gin(registration);

CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    phone           TEXT,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    auth_provider_id TEXT,
    locale          TEXT DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB DEFAULT '{}',     -- avatar_url, bio, language preferences
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES user_account(id),
    org_id          UUID NOT NULL REFERENCES organization(id),
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'trader', 'viewer', 'finance')),
    permissions     JSONB DEFAULT '[]',     -- fine-grained permission overrides
    -- Example: ["listing.create", "order.approve", "payment.view"]
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    UNIQUE (user_id, org_id, role)
);

CREATE INDEX idx_orgmember_org ON org_member(org_id);
```

---

## Commodity Catalogue

```sql
CREATE TABLE commodity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,    -- e.g., 'WHEAT_HRW', 'RICE_BASMATI', 'CORN_YEL'
    name            TEXT NOT NULL,
    category        TEXT NOT NULL,           -- e.g., 'grains', 'pulses', 'oilseeds', 'spices'
    codex_code      TEXT,                    -- Codex Alimentarius classification
    gpc_code        TEXT,                    -- GS1 Global Product Classification
    hs_code         TEXT,                    -- Harmonized System tariff code
    default_uom     TEXT NOT NULL,           -- 'mt', 'bushel', 'quintal', 'kg'
    -- Variable commodity-specific attributes
    extensions      JSONB DEFAULT '{}',
    -- Example extensions (wheat):
    -- {
    --   "protein_classes": ["HRW", "HRS", "SRW", "SRS"],
    --   "grading_systems": ["GIPSA", "EU", "Codex"],
    --   "futures_symbol": "ZW",
    --   "cbot_contract_months": ["H", "K", "N", "U", "Z"],
    --   "typical_moisture_range": {"min": 10.0, "max": 14.5}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_commodity_category ON commodity(category);
CREATE INDEX idx_commodity_extensions ON commodity USING gin(extensions);
```

---

## Marketplace Listings

```sql
CREATE TABLE listing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_org_id   UUID NOT NULL REFERENCES organization(id),
    created_by      UUID NOT NULL REFERENCES user_account(id),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),

    -- Core fields present on every listing
    listing_type    TEXT NOT NULL CHECK (listing_type IN ('spot', 'forward', 'auction')),
    quantity        NUMERIC NOT NULL CHECK (quantity > 0),
    remaining_qty   NUMERIC NOT NULL CHECK (remaining_qty >= 0),
    uom             TEXT NOT NULL,
    price           NUMERIC,                -- null for auctions
    currency_code   TEXT NOT NULL CHECK (length(currency_code) = 3),
    price_type      TEXT NOT NULL CHECK (price_type IN ('fixed', 'basis', 'negotiable', 'auction')),

    -- Origin and delivery
    origin          JSONB NOT NULL,
    -- Example origin:
    -- {
    --   "country": "US",
    --   "subdivision": "US-IA",
    --   "farm_name": "Peterson Family Farm",
    --   "field_id": "NW-40",
    --   "coordinates": {"lat": 41.878, "lng": -93.098}
    -- }
    delivery_terms  TEXT,                    -- Incoterms: FOB, CIF, DAP, etc.
    delivery_location_id UUID REFERENCES location(id),
    available_from  DATE NOT NULL,
    available_until DATE,

    -- Quality attributes: JSONB because fields vary by commodity and grading system
    quality_attributes JSONB DEFAULT '{}',
    -- Example quality_attributes (US corn, GIPSA grading):
    -- {
    --   "grading_system": "GIPSA",
    --   "grade": "US_2",
    --   "test_weight_lb_bu": 54.5,
    --   "moisture_pct": 14.0,
    --   "damage_total_pct": 3.0,
    --   "heat_damage_pct": 0.1,
    --   "broken_pct": 4.5,
    --   "foreign_material_pct": 1.2,
    --   "aflatoxin_ppb": 12
    -- }
    -- Example quality_attributes (Indian wheat, Agmarknet style):
    -- {
    --   "grading_system": "AGMARK",
    --   "grade": "FAQ",
    --   "moisture_pct": 12.0,
    --   "foreign_matter_pct": 0.5,
    --   "damaged_grains_pct": 2.0,
    --   "shrivelled_grains_pct": 3.0,
    --   "weevilled_grains_pct": 1.0,
    --   "mandi_id": "MH-042"
    -- }

    -- Basis pricing details (when price_type = 'basis')
    basis_data      JSONB,
    -- Example: {"reference": "CBOT_ZC_2026_07", "offset_cents": -15}

    -- Certifications attached to this listing
    certifications  JSONB DEFAULT '[]',
    -- Example: [
    --   {"type": "organic", "body": "USDA NOP", "cert_no": "NOP-2026-001", "expires": "2027-01-15"},
    --   {"type": "non_gmo", "body": "Non-GMO Project", "cert_no": "NGP-2026-042"}
    -- ]

    harvest_year    INTEGER,
    gtin            TEXT,

    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'draft', 'active', 'matched', 'expired', 'cancelled', 'sold'
    )),
    views_count     INTEGER DEFAULT 0,
    ai_score        NUMERIC,                -- AI-computed listing quality/relevance score
    ai_metadata     JSONB DEFAULT '{}',     -- AI model outputs: predicted demand, price confidence
    -- Example: {"price_confidence": 0.87, "demand_score": 72, "recommended_price": 218.50}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_listing_seller ON listing(seller_org_id);
CREATE INDEX idx_listing_commodity ON listing(commodity_id);
CREATE INDEX idx_listing_status ON listing(status) WHERE status = 'active';
CREATE INDEX idx_listing_price ON listing(commodity_id, price) WHERE status = 'active';
CREATE INDEX idx_listing_available ON listing(available_from, available_until) WHERE status = 'active';
CREATE INDEX idx_listing_quality ON listing USING gin(quality_attributes);
CREATE INDEX idx_listing_origin ON listing USING gin(origin);
CREATE INDEX idx_listing_certs ON listing USING gin(certifications);

-- Example JSONB containment query: find all organic corn listings in Iowa
-- SELECT * FROM listing
-- WHERE commodity_id = (SELECT id FROM commodity WHERE code = 'CORN_YEL')
--   AND status = 'active'
--   AND origin @> '{"country": "US", "subdivision": "US-IA"}'
--   AND certifications @> '[{"type": "organic"}]';

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID REFERENCES organization(id),
    name            TEXT NOT NULL,
    location_type   TEXT NOT NULL CHECK (location_type IN (
        'farm', 'warehouse', 'elevator', 'port', 'mill',
        'processing_plant', 'delivery_point', 'mandi'
    )),
    gln             TEXT,
    country_code    TEXT NOT NULL CHECK (length(country_code) = 2),
    address         JSONB DEFAULT '{}',
    -- Example: {"line1": "1234 Farm Rd", "city": "Des Moines", "state": "IA", "postal": "50309"}
    coordinates     JSONB,
    -- Example: {"lat": 41.878, "lng": -93.098}
    capacity        JSONB DEFAULT '{}',
    -- Example: {"storage_mt": 5000, "loading_rate_mt_hr": 200}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_org ON location(org_id);
CREATE INDEX idx_location_type ON location(location_type);
CREATE INDEX idx_location_country ON location(country_code);
```

---

## Orders & Bidding

```sql
CREATE TABLE bid (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES listing(id),
    buyer_org_id    UUID NOT NULL REFERENCES organization(id),
    created_by      UUID NOT NULL REFERENCES user_account(id),
    bid_price       NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    quantity        NUMERIC NOT NULL CHECK (quantity > 0),
    message         TEXT,
    counter_terms   JSONB,                   -- buyer's proposed changes to listing terms
    -- Example: {"delivery_terms": "CIF", "delivery_date": "2026-07-01"}
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'accepted', 'rejected', 'countered', 'expired', 'withdrawn'
    )),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bid_listing ON bid(listing_id);
CREATE INDEX idx_bid_buyer ON bid(buyer_org_id);

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number    TEXT NOT NULL UNIQUE,
    listing_id      UUID NOT NULL REFERENCES listing(id),
    bid_id          UUID REFERENCES bid(id),
    buyer_org_id    UUID NOT NULL REFERENCES organization(id),
    seller_org_id   UUID NOT NULL REFERENCES organization(id),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),

    -- Agreed terms (snapshot at time of order creation)
    quantity        NUMERIC NOT NULL CHECK (quantity > 0),
    uom             TEXT NOT NULL,
    agreed_price    NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    total_value     NUMERIC GENERATED ALWAYS AS (quantity * agreed_price) STORED,
    delivery_terms  TEXT,
    delivery_date   DATE,
    delivery_location_id UUID REFERENCES location(id),

    -- Agreed quality specifications (snapshot from listing)
    agreed_quality  JSONB DEFAULT '{}',

    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'accepted', 'in_transit', 'delivered',
        'inspected', 'completed', 'disputed', 'cancelled'
    )),
    status_history  JSONB DEFAULT '[]',
    -- Example: [
    --   {"status": "pending", "at": "2026-06-01T10:00:00Z", "by": "uuid"},
    --   {"status": "accepted", "at": "2026-06-01T14:30:00Z", "by": "uuid"}
    -- ]

    -- Exchange/derivatives linkage if hedged
    exchange_data   JSONB,
    -- Example: {"hedge_ref": "CME-ZC-2026N", "hedge_quantity": 200, "hedge_price": 450.25}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_buyer ON purchase_order(buyer_org_id, status);
CREATE INDEX idx_order_seller ON purchase_order(seller_org_id, status);
CREATE INDEX idx_order_listing ON purchase_order(listing_id);
CREATE INDEX idx_order_status ON purchase_order(status);
```

---

## Payments & Trade Finance

```sql
CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    payer_org_id    UUID NOT NULL REFERENCES organization(id),
    payee_org_id    UUID NOT NULL REFERENCES organization(id),
    amount          NUMERIC NOT NULL CHECK (amount > 0),
    currency_code   TEXT NOT NULL,
    payment_method  TEXT NOT NULL CHECK (payment_method IN (
        'bank_transfer', 'escrow', 'letter_of_credit',
        'mobile_money', 'platform_wallet', 'upi'
    )),
    -- Settlement details vary by method and jurisdiction
    settlement_data JSONB DEFAULT '{}',
    -- Example (ISO 20022 bank transfer):
    -- {
    --   "instruction_id": "INSTR-2026-00142",
    --   "end_to_end_id": "E2E-AGM-2026-00142",
    --   "payer_bic": "CHASUS33",
    --   "payee_bic": "BOFAUS3N",
    --   "payer_iban": "US33CHAS0000001234",
    --   "remittance_info": "AGM Order AGM-2026-00142"
    -- }
    -- Example (Indian UPI):
    -- {
    --   "upi_id": "farmer@upi",
    --   "utr_number": "UTR123456789",
    --   "ifsc_code": "SBIN0001234"
    -- }
    -- Example (mobile money, East Africa):
    -- {
    --   "provider": "M-Pesa",
    --   "phone": "+254712345678",
    --   "transaction_id": "MPESA-2026-ABC123"
    -- }

    platform_fee    NUMERIC DEFAULT 0,
    platform_fee_pct NUMERIC,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'processing', 'completed', 'failed', 'refunded'
    )),
    due_date        DATE,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_order ON payment(order_id);
CREATE INDEX idx_payment_payer ON payment(payer_org_id);
CREATE INDEX idx_payment_status ON payment(status);

CREATE TABLE trade_finance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    borrower_org_id UUID NOT NULL REFERENCES organization(id),
    lender_org_id   UUID NOT NULL REFERENCES organization(id),
    facility_type   TEXT NOT NULL CHECK (facility_type IN (
        'invoice_factoring', 'reverse_factoring', 'crop_advance',
        'warehouse_receipt', 'input_credit'
    )),
    credit_limit    NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    terms           JSONB NOT NULL,
    -- Example:
    -- {
    --   "interest_rate_bps": 450,
    --   "term_days": 90,
    --   "collateral_type": "warehouse_receipt",
    --   "advance_rate_pct": 80,
    --   "auto_repay_on_settlement": true
    -- }
    status          TEXT NOT NULL DEFAULT 'active',
    approved_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_finance_borrower ON trade_finance(borrower_org_id);

CREATE TABLE finance_drawdown (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES trade_finance(id),
    order_id        UUID REFERENCES purchase_order(id),
    amount          NUMERIC NOT NULL CHECK (amount > 0),
    currency_code   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    disbursed_at    TIMESTAMPTZ,
    repayment_due   DATE,
    repaid_at       TIMESTAMPTZ,
    details         JSONB DEFAULT '{}',
    -- Example: {"invoice_number": "INV-2026-042", "advance_rate_applied": 80}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drawdown_facility ON finance_drawdown(facility_id);
```

---

## Quality, Inspection & Traceability

```sql
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID REFERENCES listing(id),
    order_id        UUID REFERENCES purchase_order(id),
    lot_id          UUID REFERENCES lot(id),
    inspector_org_id UUID REFERENCES organization(id),
    inspection_type TEXT NOT NULL CHECK (inspection_type IN ('pre_sale', 'loading', 'discharge', 'delivery')),
    -- Quality results vary by commodity and grading system -- stored as JSONB
    results         JSONB NOT NULL,
    -- Example (GIPSA corn):
    -- {
    --   "grading_system": "GIPSA",
    --   "grade": "US_2",
    --   "test_weight_lb_bu": 54.2,
    --   "moisture_pct": 14.1,
    --   "damage_total_pct": 2.8,
    --   "foreign_material_pct": 0.9,
    --   "protein_pct": null,
    --   "aflatoxin_ppb": 15
    -- }
    -- Example (Indian pulses, Agmark):
    -- {
    --   "grading_system": "AGMARK",
    --   "grade": "Standard",
    --   "moisture_pct": 10.5,
    --   "foreign_matter_pct": 1.0,
    --   "damaged_pct": 3.0,
    --   "weevilled_pct": 0.5,
    --   "uric_acid_ppm": 50
    -- }
    certificate_number TEXT UNIQUE,
    certificate_url TEXT,
    inspection_date TIMESTAMPTZ NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_listing ON inspection(listing_id);
CREATE INDEX idx_inspection_order ON inspection(order_id);
CREATE INDEX idx_inspection_results ON inspection USING gin(results);

CREATE TABLE lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_number      TEXT NOT NULL,
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    producer_org_id UUID NOT NULL REFERENCES organization(id),
    origin_location_id UUID REFERENCES location(id),
    quantity        NUMERIC NOT NULL,
    uom             TEXT NOT NULL,
    gtin            TEXT,
    sscc            TEXT,
    harvest_date    DATE,
    harvest_year    INTEGER,
    provenance      JSONB DEFAULT '{}',
    -- Example: {
    --   "seed_variety": "Pioneer P0589AM",
    --   "field_id": "NW-40",
    --   "soil_type": "loam",
    --   "irrigation": "rainfed",
    --   "fertilizers_applied": ["urea", "DAP"],
    --   "pesticides_applied": []
    -- }
    status          TEXT NOT NULL DEFAULT 'available',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (producer_org_id, lot_number)
);

CREATE INDEX idx_lot_commodity ON lot(commodity_id);
CREATE INDEX idx_lot_producer ON lot(producer_org_id);

CREATE TABLE supply_chain_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_id          UUID NOT NULL REFERENCES lot(id),
    order_id        UUID REFERENCES purchase_order(id),
    event_type      TEXT NOT NULL,           -- EPCIS: 'object', 'aggregation', 'transformation', 'transaction'
    action          TEXT NOT NULL,           -- EPCIS: 'ADD', 'OBSERVE', 'DELETE'
    biz_step        TEXT NOT NULL,           -- EPCIS URN
    -- All EPCIS-specific and sensor fields in JSONB
    event_data      JSONB NOT NULL,
    -- Example:
    -- {
    --   "disposition": "urn:epcglobal:cbv:disp:in_transit",
    --   "read_point_gln": "0614141000012",
    --   "biz_location_gln": "0614141000015",
    --   "source_org_id": "uuid",
    --   "dest_org_id": "uuid",
    --   "quantity": 200,
    --   "uom": "mt",
    --   "transport_mode": "truck",
    --   "carrier": "Midwest Grain Transport",
    --   "tracking_number": "MGT-2026-8891",
    --   "bill_of_lading": "BOL-2026-442",
    --   "sensor_data": {
    --     "temperature_c": 22.5,
    --     "humidity_pct": 45.0,
    --     "device_id": "SENS-001"
    --   }
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sce_lot ON supply_chain_event(lot_id);
CREATE INDEX idx_sce_order ON supply_chain_event(order_id);
CREATE INDEX idx_sce_time ON supply_chain_event(occurred_at);
CREATE INDEX idx_sce_bizstep ON supply_chain_event(biz_step);
CREATE INDEX idx_sce_data ON supply_chain_event USING gin(event_data);
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
    ratings         JSONB NOT NULL,
    -- Example: {"overall": 4, "quality": 5, "reliability": 4, "communication": 3}
    comment         TEXT,
    is_public       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (order_id, reviewer_org_id)
);

CREATE INDEX idx_review_reviewee ON review(reviewee_org_id);

CREATE TABLE dispute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    raised_by_org_id UUID NOT NULL REFERENCES organization(id),
    dispute_type    TEXT NOT NULL,
    description     TEXT NOT NULL,
    evidence        JSONB DEFAULT '[]',
    -- Example: [
    --   {"type": "photo", "url": "https://...", "caption": "Damaged grain at delivery"},
    --   {"type": "document", "url": "https://...", "caption": "Original grade certificate"}
    -- ]
    status          TEXT NOT NULL DEFAULT 'open',
    resolution      JSONB,
    -- Example: {"outcome": "partial_refund", "refund_amount": 2500, "notes": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dispute_order ON dispute(order_id);
CREATE INDEX idx_dispute_status ON dispute(status);
```

---

## Market Data & AI

```sql
CREATE TABLE price_feed (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    source          TEXT NOT NULL,           -- 'platform', 'cme', 'usda_ams', 'dtn', 'agmarknet'
    quote_type      TEXT NOT NULL,           -- 'spot', 'bid', 'ask', 'settlement', 'modal'
    price           NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    uom             TEXT NOT NULL,
    location_context JSONB,
    -- Example (US): {"country": "US", "state": "IA", "elevator": "Cargill Des Moines"}
    -- Example (India): {"country": "IN", "state": "MH", "mandi": "Latur", "mandi_code": "MH-042"}
    contract_ref    TEXT,                    -- futures contract reference
    quoted_at       TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (quoted_at);

CREATE INDEX idx_price_commodity_time ON price_feed(commodity_id, quoted_at DESC);
CREATE INDEX idx_price_source ON price_feed(source, commodity_id);
CREATE INDEX idx_price_location ON price_feed USING gin(location_context);

CREATE TABLE ai_model_output (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name      TEXT NOT NULL,           -- e.g., 'pricing_engine_v2', 'fraud_detector_v1', 'match_maker_v3'
    model_version   TEXT NOT NULL,
    target_type     TEXT NOT NULL,           -- 'listing', 'order', 'organization', 'bid'
    target_id       UUID NOT NULL,
    scores          JSONB NOT NULL,
    -- Example (pricing): {"recommended_price": 218.50, "confidence": 0.87, "market_trend": "bullish"}
    -- Example (fraud):   {"fraud_risk": 0.12, "flags": ["new_seller", "price_outlier"], "action": "monitor"}
    -- Example (match):   {"match_score": 0.94, "factors": {"quality": 0.95, "proximity": 0.88, "history": 0.98}}
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_target ON ai_model_output(target_type, target_id);
CREATE INDEX idx_ai_model ON ai_model_output(model_name, computed_at DESC);

CREATE TABLE market_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES user_account(id),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    alert_config    JSONB NOT NULL,
    -- Example: {
    --   "type": "price_below",
    --   "threshold": 210.00,
    --   "currency": "USD",
    --   "country": "US",
    --   "channels": ["email", "push"]
    -- }
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
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB,
    context         JSONB DEFAULT '{}',     -- ip_address, user_agent, request_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_user_id);
CREATE INDEX idx_audit_time ON audit_log(created_at DESC);
```

---

## Example JSONB Queries

```sql
-- Find all active organic corn listings from Iowa with moisture below 14%
SELECT l.id, l.price, l.quantity, l.quality_attributes
FROM listing l
JOIN commodity c ON l.commodity_id = c.id
WHERE c.code = 'CORN_YEL'
  AND l.status = 'active'
  AND l.origin @> '{"country": "US", "subdivision": "US-IA"}'
  AND l.certifications @> '[{"type": "organic"}]'
  AND (l.quality_attributes->>'moisture_pct')::numeric < 14.0;

-- Find all organizations registered in Maharashtra, India with mandi licenses
SELECT id, legal_name, registration
FROM organization
WHERE country_code = 'IN'
  AND registration ? 'mandi_license'
  AND subdivision = 'IN-MH';

-- Aggregate AI fraud scores across listings
SELECT
    target_id AS listing_id,
    (scores->>'fraud_risk')::numeric AS fraud_risk,
    scores->'flags' AS flags
FROM ai_model_output
WHERE model_name = 'fraud_detector_v1'
  AND target_type = 'listing'
  AND (scores->>'fraud_risk')::numeric > 0.7
ORDER BY fraud_risk DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Participants | 3 | organization, user_account, org_member |
| Commodity Catalogue | 1 | commodity (extensions in JSONB) |
| Marketplace | 2 | listing, location |
| Orders & Bidding | 2 | bid, purchase_order |
| Payments & Finance | 3 | payment, trade_finance, finance_drawdown |
| Quality & Traceability | 3 | inspection, lot, supply_chain_event |
| Ratings & Disputes | 2 | review, dispute |
| Market Data & AI | 3 | price_feed, ai_model_output, market_alert |
| Audit | 1 | audit_log |
| **Total** | **20** | Fewer tables than normalized; flexibility absorbed by JSONB columns |

---

## Key Design Decisions

1. **JSONB for jurisdiction-specific registration data** -- Indian organizations need PAN, GSTIN, and mandi license numbers; US organizations need EIN and USDA licenses; EU organizations need VAT ID and EORI number. Rather than nullable columns for every possible field, `registration` JSONB absorbs all of these with GIN indexing for queries.

2. **Quality attributes as JSONB, not separate grade tables** -- GIPSA grades use test weight, moisture, and damage. Indian Agmark grades use foreign matter, shrivelled grains, and weevilled grains. Spice grading uses entirely different factors. JSONB with documented key conventions per grading system avoids a grade table that would need 50+ nullable columns.

3. **Settlement data as JSONB on payment** -- cross-border bank transfers need BIC/IBAN/ISO 20022 fields; Indian domestic payments need UPI ID and UTR number; East African payments need M-Pesa phone and transaction ID. The `settlement_data` JSONB column cleanly handles all payment rail variations.

4. **Status history as JSONB array on orders** -- instead of a separate `order_status_history` table, the full status timeline is stored inline. This makes order lifecycle queries a single-row read and reduces join complexity for the most common query pattern.

5. **AI model outputs as a dedicated table** -- rather than embedding AI scores on each entity, a separate `ai_model_output` table allows multiple models to score the same entity independently, tracks model versions, and enables A/B comparison of model performance.

6. **Certifications as JSONB array on listing** -- listings may carry zero, one, or many certifications (organic, fair trade, non-GMO, halal). A JSONB array with GIN index enables `@>` containment queries ("show me all organic listings") without a junction table.

7. **Supply chain events with EPCIS fields in JSONB** -- the EPCIS 2.0 standard defines many optional fields (sensor data, transport details, extension points). Storing these in `event_data` JSONB preserves EPCIS compatibility without mandating every optional field as a column.

8. **Generated column for `total_value`** -- `purchase_order.total_value` is computed from `quantity * agreed_price`, preventing calculation drift between application and database.

9. **Dispute evidence and resolution as JSONB** -- dispute evidence varies (photos, documents, sensor readings, witness statements) and resolution terms vary (full refund, partial refund, replacement shipment). JSONB handles both naturally.

10. **Separate `price_feed` table partitioned by time** -- external price data (CME, USDA AMS, Agmarknet, DTN) arrives continuously and is append-only. Range partitioning by `quoted_at` keeps queries fast and enables archival of historical data to cold storage.
