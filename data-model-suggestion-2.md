# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: AgriMarket Platform · Created: 2026-05-24

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The event store is the single source of truth; all queryable tables (listings, orders, balances, ratings) are materialised projections rebuilt from the event stream. The pattern is known as CQRS (Command Query Responsibility Segregation) -- writes go to the event store, reads come from purpose-built projections.

This approach is drawn from financial trading systems where regulatory compliance demands a complete, tamper-evident record of every action. Commodity exchanges, clearinghouses, and fintech platforms (as documented in real-world CQRS implementations) use event sourcing to provide full audit trails, temporal queries ("what was the listing price at 3pm on Tuesday?"), and the ability to replay history to build new analytical models. For an AI-native agricultural marketplace, the event stream doubles as a training dataset for the pricing engine, fraud detection, and match-making algorithms.

The architecture naturally separates the write path (command handlers that validate business rules and emit events) from the read path (projections optimised for specific query patterns). This means the marketplace search can be powered by Elasticsearch, the analytics dashboard by a columnar store, and the order management by PostgreSQL -- all fed from the same event stream.

**Best for:** Teams prioritising regulatory compliance, AI/ML analytics on historical patterns, temporal queries, and the ability to reconstruct any past state. Ideal when the platform must demonstrate to regulators exactly what happened, when, and why.

**Trade-offs:**
- (+) Complete, immutable audit trail -- every state change is permanently recorded
- (+) Temporal queries ("show me this farmer's listing history over 3 years") are trivial
- (+) Event stream serves as a natural training dataset for AI pricing and fraud models
- (+) New read models can be added without changing the write path
- (+) Supports event replay for debugging, compliance investigations, and disaster recovery
- (-) Higher implementation complexity -- requires event store, projectors, and materialised views
- (-) Eventual consistency between event store and projections requires careful handling
- (-) More storage consumed due to immutable event history (mitigated by snapshots)
- (-) Developers must think in events rather than CRUD, steeper learning curve
- (-) Schema evolution of events requires versioning strategy

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GIPSA / USDA Grain Standards | Grade fields embedded in `ListingCreated` and `InspectionCompleted` events |
| GS1 EPCIS 2.0 | EPCIS events map directly to domain events (`ShipmentDispatched`, `GoodsReceived`) with GTIN/GLN identifiers |
| ISO 20022 | Payment events carry ISO 20022 message identifiers (InstrId, EndToEndId, BIC) |
| ISO 22005 | Lot provenance reconstructed by replaying all events for a given lot_id |
| Codex Alimentarius | Commodity classification codes stored in reference data, referenced by events |
| FIX Protocol | Order event fields (side, order_type, time_in_force) align with FIX tag semantics |
| W3C Verifiable Credentials | `CertificateIssued` events can carry VC proof payloads for grain quality certificates |

---

## Event Store

```sql
-- The central event store: append-only, immutable, partitioned by time
CREATE TABLE domain_event (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,          -- aggregate root identifier (e.g., listing_id, order_id)
    stream_type     TEXT NOT NULL,           -- aggregate type: 'Listing', 'Order', 'Payment', 'Lot', etc.
    event_type      TEXT NOT NULL,           -- e.g., 'ListingCreated', 'BidPlaced', 'OrderAccepted'
    event_version   INTEGER NOT NULL,        -- sequence number within the stream (for optimistic concurrency)
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "actor_user_id": "uuid",
    --   "actor_org_id": "uuid",
    --   "ip_address": "192.168.1.1",
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid",
    --   "schema_version": 1
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL,    -- when the event happened in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions (example for 2026)
CREATE TABLE domain_event_2026_01 PARTITION OF domain_event
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE domain_event_2026_02 PARTITION OF domain_event
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created by automated job

CREATE INDEX idx_event_stream ON domain_event(stream_id, event_version);
CREATE INDEX idx_event_type ON domain_event(event_type, recorded_at);
CREATE INDEX idx_event_stream_type ON domain_event(stream_type, recorded_at);
CREATE INDEX idx_event_correlation ON domain_event((metadata->>'correlation_id'))
    WHERE metadata->>'correlation_id' IS NOT NULL;

-- Snapshot store for performance: avoids replaying full history for long-lived aggregates
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INTEGER NOT NULL,      -- corresponds to event_version at snapshot time
    state           JSONB NOT NULL,         -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

---

## Event Catalogue

Below are the core event types with their payload structures. Each event is a self-contained record of a business fact.

### Listing Events

```jsonc
// ListingCreated
{
  "listing_id": "uuid",
  "seller_org_id": "uuid",
  "commodity_code": "CORN_YEL_2",
  "grade_code": "US_2",
  "grading_system": "GIPSA",
  "gtin": "01234567890128",
  "listing_type": "spot",
  "quantity": 500,
  "unit_of_measure": "mt",
  "price_per_unit": 215.50,
  "currency_code": "USD",
  "price_type": "fixed",
  "origin_country": "US",
  "origin_subdivision": "US-IA",
  "harvest_year": 2026,
  "available_from": "2026-06-01",
  "delivery_terms": "FOB",
  "delivery_location_gln": "0614141000012",
  "organic_certified": false,
  "certifications": ["GIPSA_OFFICIAL"]
}

// ListingPriceUpdated
{
  "listing_id": "uuid",
  "old_price": 215.50,
  "new_price": 218.00,
  "reason": "market_adjustment"
}

// ListingCancelled
{
  "listing_id": "uuid",
  "reason": "sold_externally"
}
```

### Order Events

```jsonc
// BidPlaced
{
  "bid_id": "uuid",
  "listing_id": "uuid",
  "buyer_org_id": "uuid",
  "bid_price": 214.00,
  "quantity": 200,
  "unit_of_measure": "mt",
  "expires_at": "2026-06-05T23:59:59Z"
}

// BidAccepted
{
  "bid_id": "uuid",
  "listing_id": "uuid",
  "order_id": "uuid"  // new order created from accepted bid
}

// OrderCreated
{
  "order_id": "uuid",
  "order_number": "AGM-2026-00142",
  "listing_id": "uuid",
  "buyer_org_id": "uuid",
  "seller_org_id": "uuid",
  "commodity_code": "CORN_YEL_2",
  "grade_code": "US_2",
  "quantity": 200,
  "unit_of_measure": "mt",
  "agreed_price": 214.00,
  "currency_code": "USD",
  "delivery_terms": "FOB",
  "delivery_date": "2026-06-15",
  "delivery_location_gln": "0614141000012"
}

// OrderShipped
{
  "order_id": "uuid",
  "shipment_id": "uuid",
  "carrier_name": "Midwest Grain Transport",
  "tracking_number": "MGT-2026-8891",
  "transport_mode": "truck",
  "estimated_arrival": "2026-06-14T10:00:00Z",
  "lot_ids": ["uuid1", "uuid2"],
  "weight_mt": 200,
  "bill_of_lading": "BOL-2026-442"
}

// OrderDelivered
{
  "order_id": "uuid",
  "shipment_id": "uuid",
  "received_by": "John Smith",
  "actual_weight_mt": 199.8,
  "delivery_location_gln": "0614141000012",
  "delivered_at": "2026-06-14T14:22:00Z"
}
```

### Payment Events

```jsonc
// PaymentInstructed
{
  "payment_id": "uuid",
  "order_id": "uuid",
  "payer_org_id": "uuid",
  "payee_org_id": "uuid",
  "amount": 42800.00,
  "currency_code": "USD",
  "payment_method": "bank_transfer",
  "instruction_id": "INSTR-2026-00142",     // ISO 20022 InstrId
  "end_to_end_id": "E2E-AGM-2026-00142",    // ISO 20022 EndToEndId
  "payer_bic": "CHASUS33",
  "payee_bic": "BOFAUS3N"
}

// EscrowFunded
{
  "escrow_id": "uuid",
  "order_id": "uuid",
  "amount": 42800.00,
  "currency_code": "USD",
  "funded_at": "2026-06-02T09:15:00Z"
}

// PaymentCompleted
{
  "payment_id": "uuid",
  "settlement_reference": "SWIFT-REF-12345",
  "completed_at": "2026-06-15T11:00:00Z"
}
```

### Quality & Traceability Events

```jsonc
// InspectionCompleted
{
  "inspection_id": "uuid",
  "lot_id": "uuid",
  "order_id": "uuid",
  "inspector_org_id": "uuid",
  "grading_system": "GIPSA",
  "assigned_grade": "US_2",
  "test_weight_lb_bu": 54.2,
  "moisture_pct": 14.1,
  "damage_pct": 2.8,
  "foreign_material_pct": 0.9,
  "certificate_number": "FGIS-2026-88412"
}

// LotCreated
{
  "lot_id": "uuid",
  "lot_number": "LOT-2026-IA-0042",
  "commodity_code": "CORN_YEL_2",
  "producer_org_id": "uuid",
  "origin_location_gln": "0614141000015",
  "harvest_date": "2026-05-28",
  "quantity": 250,
  "unit_of_measure": "mt",
  "gtin": "01234567890128"
}

// TraceabilityEventRecorded (EPCIS 2.0 aligned)
{
  "epcis_event_type": "object",
  "action": "OBSERVE",
  "biz_step": "urn:epcglobal:cbv:bizstep:shipping",
  "disposition": "urn:epcglobal:cbv:disp:in_transit",
  "lot_id": "uuid",
  "read_point_gln": "0614141000012",
  "sensor_data": {
    "temperature_c": 22.5,
    "humidity_pct": 45.0,
    "recorded_at": "2026-06-14T08:00:00Z"
  }
}
```

### Finance Events

```jsonc
// FinanceFacilityApproved
{
  "facility_id": "uuid",
  "borrower_org_id": "uuid",
  "lender_org_id": "uuid",
  "facility_type": "invoice_factoring",
  "credit_limit": 500000.00,
  "currency_code": "USD",
  "interest_rate_bps": 450,
  "term_days": 90
}

// DrawdownDisbursed
{
  "drawdown_id": "uuid",
  "facility_id": "uuid",
  "order_id": "uuid",
  "amount": 42800.00,
  "currency_code": "USD",
  "repayment_due": "2026-09-14"
}
```

---

## Materialised Read Models (Projections)

These tables are rebuilt from events. They can be dropped and rebuilt at any time.

```sql
-- Projection: active listings for marketplace search
CREATE TABLE projection_listing (
    listing_id      UUID PRIMARY KEY,
    seller_org_id   UUID NOT NULL,
    seller_name     TEXT,
    commodity_code  TEXT NOT NULL,
    commodity_name  TEXT NOT NULL,
    grade_code      TEXT,
    listing_type    TEXT NOT NULL,
    quantity        NUMERIC NOT NULL,
    remaining_qty   NUMERIC NOT NULL,
    price_per_unit  NUMERIC,
    currency_code   TEXT NOT NULL,
    price_type      TEXT NOT NULL,
    origin_country  TEXT,
    origin_subdivision TEXT,
    harvest_year    INTEGER,
    available_from  DATE,
    available_until DATE,
    delivery_terms  TEXT,
    organic         BOOLEAN DEFAULT false,
    status          TEXT NOT NULL,
    bid_count       INTEGER DEFAULT 0,
    lowest_bid      NUMERIC,
    highest_bid     NUMERIC,
    views_count     INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL           -- watermark: last event processed
);

CREATE INDEX idx_proj_listing_status ON projection_listing(status) WHERE status = 'active';
CREATE INDEX idx_proj_listing_commodity ON projection_listing(commodity_code, status);
CREATE INDEX idx_proj_listing_price ON projection_listing(price_per_unit) WHERE status = 'active';
CREATE INDEX idx_proj_listing_origin ON projection_listing(origin_country, origin_subdivision);

-- Projection: order lifecycle
CREATE TABLE projection_order (
    order_id        UUID PRIMARY KEY,
    order_number    TEXT NOT NULL,
    listing_id      UUID NOT NULL,
    buyer_org_id    UUID NOT NULL,
    buyer_name      TEXT,
    seller_org_id   UUID NOT NULL,
    seller_name     TEXT,
    commodity_code  TEXT NOT NULL,
    grade_code      TEXT,
    quantity        NUMERIC NOT NULL,
    agreed_price    NUMERIC NOT NULL,
    total_value     NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    delivery_terms  TEXT,
    delivery_date   DATE,
    status          TEXT NOT NULL,
    payment_status  TEXT DEFAULT 'pending',
    shipment_status TEXT DEFAULT 'pending',
    inspection_grade TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL
);

CREATE INDEX idx_proj_order_buyer ON projection_order(buyer_org_id, status);
CREATE INDEX idx_proj_order_seller ON projection_order(seller_org_id, status);
CREATE INDEX idx_proj_order_status ON projection_order(status);

-- Projection: organisation reputation / trust score
CREATE TABLE projection_org_reputation (
    org_id              UUID PRIMARY KEY,
    org_name            TEXT NOT NULL,
    org_type            TEXT NOT NULL,
    total_trades        INTEGER DEFAULT 0,
    total_trade_value   NUMERIC DEFAULT 0,
    avg_rating          NUMERIC DEFAULT 0,
    rating_count        INTEGER DEFAULT 0,
    on_time_delivery_pct NUMERIC DEFAULT 0,
    dispute_count       INTEGER DEFAULT 0,
    dispute_resolved_pct NUMERIC DEFAULT 0,
    member_since        TIMESTAMPTZ,
    last_trade_at       TIMESTAMPTZ,
    last_event_id       UUID NOT NULL
);

CREATE INDEX idx_proj_rep_rating ON projection_org_reputation(avg_rating DESC);

-- Projection: commodity price time series (for charts and AI)
CREATE TABLE projection_price_series (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commodity_code  TEXT NOT NULL,
    price_source    TEXT NOT NULL,          -- 'platform_avg', 'cme', 'usda_ams', 'agmarknet'
    price           NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    unit_of_measure TEXT NOT NULL,
    origin_country  TEXT,
    observation_time TIMESTAMPTZ NOT NULL,
    last_event_id   UUID NOT NULL
) PARTITION BY RANGE (observation_time);

CREATE INDEX idx_proj_price_commodity ON projection_price_series(commodity_code, observation_time DESC);

-- Projection: lot traceability chain
CREATE TABLE projection_lot_trace (
    lot_id          UUID NOT NULL,
    event_seq       INTEGER NOT NULL,
    event_type      TEXT NOT NULL,
    biz_step        TEXT,
    location_gln    TEXT,
    actor_org_id    UUID,
    actor_org_name  TEXT,
    quantity        NUMERIC,
    occurred_at     TIMESTAMPTZ NOT NULL,
    sensor_summary  JSONB,
    last_event_id   UUID NOT NULL,
    PRIMARY KEY (lot_id, event_seq)
);

CREATE INDEX idx_proj_lot_trace_lot ON projection_lot_trace(lot_id);
```

---

## Reference Data Tables

Reference data is mutable (not event-sourced) because it represents shared domain knowledge, not transactional state.

```sql
CREATE TABLE ref_commodity (
    code            TEXT PRIMARY KEY,       -- e.g., 'CORN_YEL_2'
    name            TEXT NOT NULL,
    category        TEXT NOT NULL,
    codex_code      TEXT,
    gpc_code        TEXT,
    variety         TEXT,
    default_uom     TEXT NOT NULL,
    hs_code         TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_grade (
    commodity_code  TEXT NOT NULL REFERENCES ref_commodity(code),
    grading_system  TEXT NOT NULL,
    grade_code      TEXT NOT NULL,
    grade_name      TEXT NOT NULL,
    specifications  JSONB DEFAULT '{}',
    PRIMARY KEY (commodity_code, grading_system, grade_code)
);

CREATE TABLE ref_jurisdiction (
    country_code    TEXT NOT NULL CHECK (length(country_code) = 2),
    subdivision_code TEXT NOT NULL DEFAULT '',
    name            TEXT NOT NULL,
    currency_code   TEXT NOT NULL,
    timezone        TEXT NOT NULL,
    PRIMARY KEY (country_code, subdivision_code)
);

CREATE TABLE ref_organization (
    org_id          UUID PRIMARY KEY,
    legal_name      TEXT NOT NULL,
    trade_name      TEXT,
    org_type        TEXT NOT NULL,
    gln             TEXT,
    lei             TEXT,
    country_code    TEXT NOT NULL,
    verified        BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE ref_user (
    user_id         UUID PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true
);
```

---

## Example Queries

### Rebuild an order's full history

```sql
-- Replay all events for a specific order to see its complete lifecycle
SELECT
    event_type,
    occurred_at,
    payload,
    metadata->>'actor_user_id' AS actor
FROM domain_event
WHERE stream_id = '<<order-uuid>>'
  AND stream_type = 'Order'
ORDER BY event_version ASC;
```

### Time-travel: what was the listing price at a specific time?

```sql
-- Find the effective price of a listing at a given point in time
SELECT payload->>'new_price' AS price_at_time, occurred_at
FROM domain_event
WHERE stream_id = '<<listing-uuid>>'
  AND event_type IN ('ListingCreated', 'ListingPriceUpdated')
  AND occurred_at <= '2026-06-10T12:00:00Z'
ORDER BY occurred_at DESC
LIMIT 1;
```

### AI training: extract all price change events for a commodity

```sql
-- Feed the pricing engine: all price updates for corn across the platform
SELECT
    de.stream_id AS listing_id,
    de.payload->>'commodity_code' AS commodity,
    de.payload->>'old_price' AS old_price,
    de.payload->>'new_price' AS new_price,
    de.payload->>'reason' AS reason,
    de.occurred_at
FROM domain_event de
WHERE de.event_type = 'ListingPriceUpdated'
  AND de.payload->>'commodity_code' LIKE 'CORN%'
ORDER BY de.occurred_at ASC;
```

### Detect suspicious patterns: rapid price changes

```sql
-- Fraud signal: listings with more than 3 price changes in 24 hours
WITH price_changes AS (
    SELECT
        stream_id AS listing_id,
        occurred_at,
        LAG(occurred_at) OVER (PARTITION BY stream_id ORDER BY occurred_at) AS prev_change
    FROM domain_event
    WHERE event_type = 'ListingPriceUpdated'
      AND occurred_at > now() - INTERVAL '7 days'
)
SELECT listing_id, COUNT(*) AS changes_in_24h
FROM price_changes
WHERE occurred_at - prev_change < INTERVAL '24 hours'
GROUP BY listing_id
HAVING COUNT(*) >= 3
ORDER BY changes_in_24h DESC;
```

---

## Projection Rebuild Process

```sql
-- Example: rebuild the listing projection from scratch
-- Step 1: Truncate the projection
TRUNCATE projection_listing;

-- Step 2: Replay events in order
-- (This is typically done by application-level projector code, not raw SQL)
-- Pseudocode:
-- FOR EACH event IN (SELECT * FROM domain_event WHERE stream_type = 'Listing' ORDER BY recorded_at, event_version):
--   CASE event.event_type:
--     'ListingCreated':  INSERT INTO projection_listing ...
--     'ListingPriceUpdated': UPDATE projection_listing SET price_per_unit = ...
--     'ListingCancelled': UPDATE projection_listing SET status = 'cancelled'
--     'BidPlaced': UPDATE projection_listing SET bid_count = bid_count + 1
--   END CASE
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | domain_event (partitioned), event_snapshot |
| Reference Data | 5 | ref_commodity, ref_grade, ref_jurisdiction, ref_organization, ref_user |
| Projections | 5 | projection_listing, projection_order, projection_org_reputation, projection_price_series, projection_lot_trace |
| **Total** | **12** | Far fewer tables than normalized; complexity lives in event types and projector code |

---

## Key Design Decisions

1. **Single `domain_event` table, not one table per aggregate** -- simplifies infrastructure and makes cross-aggregate queries (correlation analysis, global audit) straightforward. The `stream_type` column enables filtered indexes for per-aggregate access.

2. **JSONB payload rather than typed columns per event** -- event types evolve over time (new fields added, old fields deprecated). JSONB with a `schema_version` in metadata allows the projector to handle multiple versions gracefully without DDL migrations on the event store.

3. **Monthly partitioning on `recorded_at`** -- the event store grows indefinitely (by design, it is never deleted). Partitioning keeps query performance stable and enables archiving old partitions to cold storage.

4. **Snapshots for long-lived aggregates** -- an order that has been amended 50 times does not require replaying all 50 events on every read. Periodic snapshots capture the current state, and replay starts from the most recent snapshot.

5. **Projections are disposable** -- every projection table can be dropped and rebuilt from the event store. This means new analytics requirements (e.g., "average time from listing to sale by commodity") can be added as new projections without touching the write path.

6. **Reference data is mutable, not event-sourced** -- commodity codes, grade definitions, and jurisdiction data change infrequently and do not need an audit trail. Keeping them as simple CRUD tables avoids unnecessary complexity.

7. **Metadata captures context** -- every event records who did it (`actor_user_id`), what triggered it (`causation_id`), and which business flow it belongs to (`correlation_id`). This enables end-to-end request tracing and compliance investigations.

8. **Optimistic concurrency via `event_version`** -- the `UNIQUE (stream_id, event_version)` constraint prevents two concurrent commands from writing conflicting events to the same aggregate. The application increments the version and retries on conflict.

9. **Event stream as AI training data** -- the pricing engine, fraud detector, and match-maker can all consume the event stream directly (via CDC or periodic batch export) rather than requiring separate ETL pipelines from normalised tables.

10. **EPCIS events are domain events** -- rather than maintaining a separate traceability system, GS1 EPCIS events are recorded as `TraceabilityEventRecorded` domain events in the same store, ensuring a single timeline for all platform activity.
