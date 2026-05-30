# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: AgriMarket Platform · Created: 2026-05-24

## Philosophy

This model adds a property graph layer on top of relational operational tables. The insight is that an agricultural marketplace is fundamentally a network problem: farmers connect to buyers through trades, lots flow through supply chains via logistics nodes, organizations have trust relationships built from trade history, and commodities move through geographic corridors. Graph traversal queries answer the platform's most valuable questions: "Which buyers near this farmer have reliably purchased this grade of corn?" "What is the complete provenance chain from field to mill for this shipment?" "Are there circular trading patterns that suggest wash trading?"

The architecture uses PostgreSQL for all CRUD operations (listings, orders, payments) and layers a graph model on top using either dedicated `graph_node` / `graph_edge` tables within PostgreSQL (using recursive CTEs and ltree) or an external graph database like Neo4j fed via CDC. The graph is not the system of record for transactional data -- the relational tables are. The graph is a materialised relationship index optimised for traversal, matching, and pattern detection.

This pattern is used by fraud detection systems in financial services (PayPal, Stripe), recommendation engines (LinkedIn, Amazon), and supply chain visibility platforms (project44, FourKites) where relationship traversal is a core product feature. For an AI-native agricultural marketplace, the graph becomes the substrate for the match-making engine, fraud detector, and supply chain traceability visualiser.

**Best for:** Teams where buyer-seller matching, supply chain traceability, fraud detection, and network analytics are primary differentiators. Ideal when the AI match-making engine and trust scoring system are core product features rather than afterthoughts.

**Trade-offs:**
- (+) Natural representation of supply chain networks and trade relationships
- (+) Efficient multi-hop queries: "find all buyers within 2 degrees of this farmer"
- (+) Pattern matching for fraud detection (cycles, unusual relationship density)
- (+) Supply chain provenance is a graph traversal, not a chain of JOINs
- (+) AI match-making operates directly on graph features (centrality, similarity)
- (-) Dual-model complexity -- developers must understand both relational and graph paradigms
- (-) Graph consistency must be maintained alongside relational writes (sync overhead)
- (-) Graph databases (Neo4j) add operational complexity; in-PostgreSQL graphs sacrifice some traversal performance
- (-) Fewer developers are experienced with graph query languages (Cypher, SPARQL)
- (-) Backup, migration, and monitoring tooling is less mature for graph databases

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 | Supply chain events modeled as graph edges linking lot nodes to location nodes |
| GS1 GTIN/GLN | Node properties on commodity and location nodes |
| GIPSA / USDA Grain Standards | Quality attributes as properties on listing and inspection nodes |
| ISO 20022 | Payment node properties carry ISO 20022 message identifiers |
| ISO 22005 | Lot provenance is a graph path from farm node through logistics to buyer |
| ISO 3166 | Jurisdiction hierarchy modeled as a tree in the graph |
| Codex Alimentarius | Commodity classification hierarchy as a tree of category nodes |
| FIX Protocol | Order nodes carry FIX-aligned fields (side, order_type) |

---

## Relational Layer (System of Record)

The relational tables handle CRUD operations, financial transactions, and data integrity. These are simpler than a fully normalised model because relationship complexity lives in the graph.

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_name      TEXT NOT NULL,
    trade_name      TEXT,
    org_type        TEXT NOT NULL CHECK (org_type IN (
        'farm', 'cooperative', 'processor', 'exporter',
        'importer', 'trader', 'lender', 'government', 'logistics'
    )),
    country_code    TEXT NOT NULL CHECK (length(country_code) = 2),
    subdivision     TEXT,
    gln             TEXT,
    lei             TEXT,
    verification_status TEXT NOT NULL DEFAULT 'pending',
    registration    JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_type ON organization(org_type);
CREATE INDEX idx_org_country ON organization(country_code);

CREATE TABLE user_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES user_account(id),
    org_id          UUID NOT NULL REFERENCES organization(id),
    role            TEXT NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, org_id, role)
);

CREATE TABLE commodity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    category        TEXT NOT NULL,
    codex_code      TEXT,
    hs_code         TEXT,
    default_uom     TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID REFERENCES organization(id),
    name            TEXT NOT NULL,
    location_type   TEXT NOT NULL,
    gln             TEXT,
    country_code    TEXT NOT NULL,
    latitude        NUMERIC,
    longitude       NUMERIC,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_location_org ON location(org_id);

CREATE TABLE listing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_org_id   UUID NOT NULL REFERENCES organization(id),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    listing_type    TEXT NOT NULL,
    quantity        NUMERIC NOT NULL CHECK (quantity > 0),
    remaining_qty   NUMERIC NOT NULL,
    price           NUMERIC,
    currency_code   TEXT NOT NULL,
    price_type      TEXT NOT NULL,
    quality_attrs   JSONB DEFAULT '{}',
    origin_location_id UUID REFERENCES location(id),
    delivery_location_id UUID REFERENCES location(id),
    available_from  DATE NOT NULL,
    available_until DATE,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_listing_seller ON listing(seller_org_id);
CREATE INDEX idx_listing_commodity ON listing(commodity_id);
CREATE INDEX idx_listing_status ON listing(status) WHERE status = 'active';

CREATE TABLE bid (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    listing_id      UUID NOT NULL REFERENCES listing(id),
    buyer_org_id    UUID NOT NULL REFERENCES organization(id),
    bid_price       NUMERIC NOT NULL,
    quantity        NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bid_listing ON bid(listing_id);
CREATE INDEX idx_bid_buyer ON bid(buyer_org_id);

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number    TEXT NOT NULL UNIQUE,
    listing_id      UUID NOT NULL REFERENCES listing(id),
    buyer_org_id    UUID NOT NULL REFERENCES organization(id),
    seller_org_id   UUID NOT NULL REFERENCES organization(id),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    quantity        NUMERIC NOT NULL,
    agreed_price    NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    delivery_terms  TEXT,
    delivery_date   DATE,
    status          TEXT NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_buyer ON purchase_order(buyer_org_id);
CREATE INDEX idx_order_seller ON purchase_order(seller_org_id);
CREATE INDEX idx_order_status ON purchase_order(status);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    payer_org_id    UUID NOT NULL REFERENCES organization(id),
    payee_org_id    UUID NOT NULL REFERENCES organization(id),
    amount          NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    payment_method  TEXT NOT NULL,
    settlement_data JSONB DEFAULT '{}',
    status          TEXT NOT NULL DEFAULT 'pending',
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_order ON payment(order_id);

CREATE TABLE lot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_number      TEXT NOT NULL,
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    producer_org_id UUID NOT NULL REFERENCES organization(id),
    origin_location_id UUID REFERENCES location(id),
    quantity        NUMERIC NOT NULL,
    uom             TEXT NOT NULL,
    gtin            TEXT,
    harvest_date    DATE,
    status          TEXT NOT NULL DEFAULT 'available',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (producer_org_id, lot_number)
);

CREATE INDEX idx_lot_commodity ON lot(commodity_id);
CREATE INDEX idx_lot_producer ON lot(producer_org_id);

CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lot_id          UUID REFERENCES lot(id),
    order_id        UUID REFERENCES purchase_order(id),
    inspector_org_id UUID REFERENCES organization(id),
    results         JSONB NOT NULL,
    certificate_number TEXT UNIQUE,
    inspection_date TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE trade_finance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    borrower_org_id UUID NOT NULL REFERENCES organization(id),
    lender_org_id   UUID NOT NULL REFERENCES organization(id),
    facility_type   TEXT NOT NULL,
    credit_limit    NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    terms           JSONB NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE review (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    reviewer_org_id UUID NOT NULL REFERENCES organization(id),
    reviewee_org_id UUID NOT NULL REFERENCES organization(id),
    ratings         JSONB NOT NULL,
    comment         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (order_id, reviewer_org_id)
);

CREATE TABLE dispute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES purchase_order(id),
    raised_by_org_id UUID NOT NULL REFERENCES organization(id),
    dispute_type    TEXT NOT NULL,
    description     TEXT NOT NULL,
    evidence        JSONB DEFAULT '[]',
    status          TEXT NOT NULL DEFAULT 'open',
    resolution      JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Graph Layer (Relationship Index)

The graph layer can be implemented either in-PostgreSQL (using dedicated tables with recursive CTEs) or in Neo4j. Below shows the PostgreSQL approach, with Neo4j Cypher equivalents noted.

### Graph Tables

```sql
-- Generic graph node table: every entity in the system gets a node
CREATE TABLE graph_node (
    node_id         UUID PRIMARY KEY,       -- same as entity ID in relational tables
    node_type       TEXT NOT NULL,           -- 'organization', 'listing', 'lot', 'location', 'commodity', 'order'
    label           TEXT NOT NULL,           -- display name
    properties      JSONB DEFAULT '{}',     -- denormalized attributes for graph queries
    -- Example (organization node):
    -- {
    --   "org_type": "farm",
    --   "country": "US",
    --   "subdivision": "US-IA",
    --   "verified": true,
    --   "avg_rating": 4.2,
    --   "total_trades": 47,
    --   "member_since": "2024-03-15"
    -- }
    -- Example (commodity node):
    -- {
    --   "category": "grains",
    --   "codex_code": "GC 0645",
    --   "default_uom": "bushel"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gnode_type ON graph_node(node_type);
CREATE INDEX idx_gnode_props ON graph_node USING gin(properties);

-- Generic graph edge table: typed, directed, weighted relationships
CREATE TABLE graph_edge (
    edge_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_node(node_id),
    target_id       UUID NOT NULL REFERENCES graph_node(node_id),
    edge_type       TEXT NOT NULL,
    -- Edge types:
    --   TRADED_WITH         org -> org (weight = trade count)
    --   LISTED_BY           listing -> org
    --   ORDERED_FROM        order -> org (seller)
    --   ORDERED_BY          order -> org (buyer)
    --   COMMODITY_OF        listing/order -> commodity
    --   LOCATED_AT          org/lot -> location
    --   ORIGINATED_FROM     lot -> location (farm)
    --   SHIPPED_TO          lot -> location (destination)
    --   SHIPPED_VIA         lot -> location (intermediate)
    --   INSPECTED_BY        lot/order -> org (inspector)
    --   FINANCED_BY         org -> org (lender)
    --   PART_OF_CATEGORY    commodity -> commodity_category
    --   IN_JURISDICTION      location -> jurisdiction
    --   PRODUCED            org -> lot
    --   CONTAINS_LOT        order -> lot
    weight          NUMERIC DEFAULT 1.0,    -- relationship strength/count
    properties      JSONB DEFAULT '{}',
    -- Example (TRADED_WITH edge):
    -- {
    --   "first_trade": "2024-06-10",
    --   "last_trade": "2026-05-20",
    --   "trade_count": 12,
    --   "total_value_usd": 485000,
    --   "avg_rating": 4.5,
    --   "commodities_traded": ["CORN_YEL", "SOY"],
    --   "on_time_pct": 92
    -- }
    -- Example (SHIPPED_VIA edge):
    -- {
    --   "transport_mode": "truck",
    --   "carrier": "Midwest Grain Transport",
    --   "departed_at": "2026-06-14T08:00:00Z",
    --   "arrived_at": "2026-06-14T14:00:00Z",
    --   "tracking": "MGT-2026-8891"
    -- }
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until     TIMESTAMPTZ,            -- null = currently valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_source ON graph_edge(source_id, edge_type);
CREATE INDEX idx_gedge_target ON graph_edge(target_id, edge_type);
CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_props ON graph_edge USING gin(properties);
CREATE INDEX idx_gedge_validity ON graph_edge(valid_from, valid_until);

-- Materialised view: organisation trust network
CREATE MATERIALIZED VIEW org_trade_network AS
SELECT
    ge.source_id AS org_a,
    ge.target_id AS org_b,
    ge.weight AS trade_count,
    (ge.properties->>'total_value_usd')::numeric AS total_value,
    (ge.properties->>'avg_rating')::numeric AS avg_rating,
    (ge.properties->>'on_time_pct')::numeric AS on_time_pct,
    ge.properties->'commodities_traded' AS commodities
FROM graph_edge ge
WHERE ge.edge_type = 'TRADED_WITH'
  AND ge.valid_until IS NULL;

CREATE INDEX idx_otn_org_a ON org_trade_network(org_a);
CREATE INDEX idx_otn_org_b ON org_trade_network(org_b);
```

---

## Graph Queries (PostgreSQL Recursive CTEs)

### Find all buyers who have traded with a farmer (direct and indirect)

```sql
-- 2-hop trade network: "buyers my trusted partners also trade with"
WITH RECURSIVE trade_network AS (
    -- Direct trading partners
    SELECT
        ge.target_id AS org_id,
        gn.label AS org_name,
        1 AS depth,
        ARRAY[ge.source_id, ge.target_id] AS path,
        ge.weight AS connection_strength
    FROM graph_edge ge
    JOIN graph_node gn ON gn.node_id = ge.target_id
    WHERE ge.source_id = '<<farmer-org-uuid>>'
      AND ge.edge_type = 'TRADED_WITH'
      AND ge.valid_until IS NULL
      AND gn.properties->>'org_type' IN ('processor', 'exporter', 'trader')

    UNION ALL

    -- Partners of partners (2nd hop)
    SELECT
        ge.target_id,
        gn.label,
        tn.depth + 1,
        tn.path || ge.target_id,
        tn.connection_strength * ge.weight * 0.5  -- decay factor
    FROM trade_network tn
    JOIN graph_edge ge ON ge.source_id = tn.org_id
    JOIN graph_node gn ON gn.node_id = ge.target_id
    WHERE ge.edge_type = 'TRADED_WITH'
      AND ge.valid_until IS NULL
      AND ge.target_id != ALL(tn.path)  -- prevent cycles
      AND tn.depth < 2
)
SELECT org_id, org_name, depth, connection_strength
FROM trade_network
ORDER BY connection_strength DESC
LIMIT 20;
```

### Supply chain provenance: trace a lot from field to buyer

```sql
-- Full provenance chain for a lot
WITH RECURSIVE provenance AS (
    -- Start at the lot's origin
    SELECT
        ge.edge_id,
        ge.source_id,
        ge.target_id,
        ge.edge_type,
        gn_src.label AS from_label,
        gn_tgt.label AS to_label,
        ge.properties,
        1 AS step,
        ARRAY[ge.source_id] AS visited
    FROM graph_edge ge
    JOIN graph_node gn_src ON gn_src.node_id = ge.source_id
    JOIN graph_node gn_tgt ON gn_tgt.node_id = ge.target_id
    WHERE ge.source_id = '<<lot-uuid>>'
      AND ge.edge_type IN ('ORIGINATED_FROM', 'SHIPPED_VIA', 'SHIPPED_TO', 'INSPECTED_BY')

    UNION ALL

    SELECT
        ge.edge_id,
        ge.source_id,
        ge.target_id,
        ge.edge_type,
        gn_src.label,
        gn_tgt.label,
        ge.properties,
        p.step + 1,
        p.visited || ge.source_id
    FROM provenance p
    JOIN graph_edge ge ON ge.source_id = p.target_id
    JOIN graph_node gn_src ON gn_src.node_id = ge.source_id
    JOIN graph_node gn_tgt ON gn_tgt.node_id = ge.target_id
    WHERE ge.edge_type IN ('SHIPPED_VIA', 'SHIPPED_TO', 'INSPECTED_BY')
      AND ge.source_id != ALL(p.visited)
      AND p.step < 10
)
SELECT step, edge_type, from_label, to_label, properties
FROM provenance
ORDER BY step;
```

### Fraud detection: find circular trading patterns

```sql
-- Detect potential wash trading: org A -> org B -> org C -> org A cycles
WITH RECURSIVE cycle_finder AS (
    SELECT
        ge.source_id AS start_org,
        ge.target_id AS current_org,
        ARRAY[ge.source_id, ge.target_id] AS path,
        1 AS depth
    FROM graph_edge ge
    WHERE ge.edge_type = 'TRADED_WITH'
      AND ge.valid_until IS NULL
      AND (ge.properties->>'last_trade')::date > CURRENT_DATE - INTERVAL '90 days'

    UNION ALL

    SELECT
        cf.start_org,
        ge.target_id,
        cf.path || ge.target_id,
        cf.depth + 1
    FROM cycle_finder cf
    JOIN graph_edge ge ON ge.source_id = cf.current_org
    WHERE ge.edge_type = 'TRADED_WITH'
      AND ge.valid_until IS NULL
      AND cf.depth < 4
      AND (
          ge.target_id = cf.start_org  -- found a cycle
          OR ge.target_id != ALL(cf.path)  -- or continue exploring
      )
)
SELECT
    path,
    depth AS cycle_length,
    start_org
FROM cycle_finder
WHERE current_org = start_org  -- only cycles
  AND depth >= 3               -- at least 3-hop cycle
ORDER BY depth ASC;
```

### AI match-making: find best buyers for a listing

```sql
-- Score potential buyers for a specific listing based on graph features
WITH listing_context AS (
    SELECT
        l.id AS listing_id,
        l.seller_org_id,
        l.commodity_id,
        c.code AS commodity_code,
        l.price,
        l.quality_attrs,
        loc.country_code,
        loc.latitude AS origin_lat,
        loc.longitude AS origin_lng
    FROM listing l
    JOIN commodity c ON c.id = l.commodity_id
    LEFT JOIN location loc ON loc.id = l.origin_location_id
    WHERE l.id = '<<listing-uuid>>'
),
candidate_buyers AS (
    SELECT DISTINCT
        ge.target_id AS buyer_org_id,
        gn.label AS buyer_name,
        gn.properties->>'org_type' AS buyer_type,
        gn.properties->>'country' AS buyer_country,
        -- Trade history with this seller
        COALESCE((
            SELECT (tw.properties->>'trade_count')::int
            FROM graph_edge tw
            WHERE tw.source_id = lc.seller_org_id
              AND tw.target_id = ge.target_id
              AND tw.edge_type = 'TRADED_WITH'
              AND tw.valid_until IS NULL
        ), 0) AS prior_trades_with_seller,
        -- Buyer's overall reputation
        COALESCE((gn.properties->>'avg_rating')::numeric, 0) AS buyer_rating,
        COALESCE((gn.properties->>'total_trades')::int, 0) AS buyer_total_trades
    FROM listing_context lc
    -- Find orgs that have previously traded this commodity
    JOIN graph_edge ge ON ge.edge_type = 'TRADED_WITH'
      AND ge.valid_until IS NULL
    JOIN graph_node gn ON gn.node_id = ge.target_id
      AND gn.node_type = 'organization'
    WHERE gn.properties->>'org_type' IN ('processor', 'exporter', 'trader', 'importer')
      AND ge.properties->'commodities_traded' @> to_jsonb(lc.commodity_code)
)
SELECT
    buyer_org_id,
    buyer_name,
    buyer_type,
    buyer_country,
    prior_trades_with_seller,
    buyer_rating,
    buyer_total_trades,
    -- Composite match score
    (
        prior_trades_with_seller * 10 +     -- strong weight for existing relationship
        buyer_rating * 5 +                   -- reputation matters
        LEAST(buyer_total_trades, 100) * 0.5 -- experience, capped
    ) AS match_score
FROM candidate_buyers
ORDER BY match_score DESC
LIMIT 25;
```

---

## Neo4j Cypher Equivalents

If deploying with Neo4j as the graph engine (fed via CDC from PostgreSQL), the same queries become more concise.

### Trade network traversal

```cypher
// Find buyers 2 hops from a farmer
MATCH (farmer:Organization {id: $farmerId})-[:TRADED_WITH*1..2]-(buyer:Organization)
WHERE buyer.org_type IN ['processor', 'exporter', 'trader']
  AND buyer.id <> farmer.id
RETURN buyer.id, buyer.legal_name, buyer.avg_rating,
       length(shortestPath((farmer)-[:TRADED_WITH*]-(buyer))) AS hops
ORDER BY buyer.avg_rating DESC
LIMIT 20
```

### Supply chain provenance

```cypher
// Trace a lot from origin to destination
MATCH path = (lot:Lot {id: $lotId})-[:ORIGINATED_FROM|SHIPPED_VIA|SHIPPED_TO*]->(dest)
RETURN [n IN nodes(path) | {id: n.id, label: n.label, type: labels(n)[0]}] AS chain,
       [r IN relationships(path) | {type: type(r), props: properties(r)}] AS steps
```

### Circular trading detection

```cypher
// Find wash trading cycles
MATCH (a:Organization)-[:TRADED_WITH]->(b:Organization)-[:TRADED_WITH]->(c:Organization)-[:TRADED_WITH]->(a)
WHERE a.id < b.id AND b.id < c.id  // avoid duplicates
RETURN a.legal_name, b.legal_name, c.legal_name
```

---

## Graph Synchronisation

The graph layer is kept in sync with the relational layer via application-level event handlers or Change Data Capture (CDC).

```sql
-- Trigger function to create/update graph nodes when organizations change
CREATE OR REPLACE FUNCTION sync_org_to_graph() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (node_id, node_type, label, properties)
    VALUES (
        NEW.id,
        'organization',
        COALESCE(NEW.trade_name, NEW.legal_name),
        jsonb_build_object(
            'org_type', NEW.org_type,
            'country', NEW.country_code,
            'subdivision', NEW.subdivision,
            'verified', NEW.verification_status = 'verified'
        )
    )
    ON CONFLICT (node_id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_org_graph_sync
    AFTER INSERT OR UPDATE ON organization
    FOR EACH ROW EXECUTE FUNCTION sync_org_to_graph();

-- Trigger to update TRADED_WITH edges when orders complete
CREATE OR REPLACE FUNCTION sync_trade_edge() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'completed' AND OLD.status != 'completed' THEN
        -- Upsert TRADED_WITH edge between buyer and seller
        INSERT INTO graph_edge (source_id, target_id, edge_type, weight, properties)
        VALUES (
            NEW.buyer_org_id,
            NEW.seller_org_id,
            'TRADED_WITH',
            1,
            jsonb_build_object(
                'first_trade', now()::date,
                'last_trade', now()::date,
                'trade_count', 1,
                'total_value_usd', NEW.quantity * NEW.agreed_price
            )
        )
        ON CONFLICT DO NOTHING;  -- simplified; real impl uses upsert with increment

        -- Also create reverse edge for bidirectional traversal
        INSERT INTO graph_edge (source_id, target_id, edge_type, weight, properties)
        VALUES (
            NEW.seller_org_id,
            NEW.buyer_org_id,
            'TRADED_WITH',
            1,
            jsonb_build_object(
                'first_trade', now()::date,
                'last_trade', now()::date,
                'trade_count', 1,
                'total_value_usd', NEW.quantity * NEW.agreed_price
            )
        )
        ON CONFLICT DO NOTHING;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_trade_graph_sync
    AFTER UPDATE ON purchase_order
    FOR EACH ROW EXECUTE FUNCTION sync_trade_edge();
```

---

## Price Data & Audit

```sql
CREATE TABLE price_feed (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    commodity_id    UUID NOT NULL REFERENCES commodity(id),
    source          TEXT NOT NULL,
    quote_type      TEXT NOT NULL,
    price           NUMERIC NOT NULL,
    currency_code   TEXT NOT NULL,
    uom             TEXT NOT NULL,
    location_context JSONB,
    quoted_at       TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (quoted_at);

CREATE INDEX idx_price_commodity_time ON price_feed(commodity_id, quoted_at DESC);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_user_id   UUID REFERENCES user_account(id),
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_time ON audit_log(created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Relational: Participants | 3 | organization, user_account, org_member |
| Relational: Catalogue | 2 | commodity, location |
| Relational: Marketplace | 4 | listing, bid, purchase_order, payment |
| Relational: Quality & Supply Chain | 2 | lot, inspection |
| Relational: Finance & Reviews | 3 | trade_finance, review, dispute |
| Relational: Market Data & Audit | 2 | price_feed, audit_log |
| Graph Layer | 2 | graph_node, graph_edge |
| Materialised Views | 1 | org_trade_network |
| **Total** | **19** | Plus triggers for graph sync; or replace graph tables with Neo4j |

---

## Key Design Decisions

1. **Graph as an index, not a system of record** -- the relational tables own the data. The graph is a derived, traversal-optimised index. If the graph is lost, it can be rebuilt from relational data. This avoids the risk of graph-relational divergence causing data integrity issues.

2. **Generic node/edge tables rather than typed graph tables** -- a single `graph_node` and `graph_edge` pair with `node_type` / `edge_type` columns enables any new entity or relationship to be added without DDL changes. Properties go in JSONB. This mirrors the property graph model used by Neo4j and Apache TinkerPop.

3. **Bidirectional TRADED_WITH edges** -- trading is a mutual relationship, so both directions are stored. This allows efficient queries from either party ("who does this buyer trade with?" and "who trades with this seller?").

4. **Temporal validity on edges** -- `valid_from` and `valid_until` on `graph_edge` enable historical graph queries ("what was this farmer's trading network 12 months ago?"). Edges are never deleted; they are superseded by setting `valid_until`.

5. **Materialised view for hot queries** -- `org_trade_network` pre-computes the most frequently accessed graph data (who trades with whom, how much, how reliably). This avoids recursive CTE overhead for the match-making engine's most common query pattern.

6. **Trigger-based sync** -- PostgreSQL triggers on `organization`, `purchase_order`, `lot`, and `listing` automatically maintain graph node/edge consistency. For Neo4j deployments, these triggers are replaced by a CDC pipeline (Debezium) that streams changes to Neo4j.

7. **Graph features feed AI match-making** -- the composite match score in the buyer-matching query uses graph-native features (prior trades, network distance, reputation) that would be expensive multi-join queries in a purely relational model but are natural in a graph.

8. **Recursive CTE depth limits** -- all recursive queries are capped (typically at 2-4 hops) to prevent runaway traversals. The agricultural trade network is relatively dense, so unbounded traversal would be expensive.

9. **Fraud detection as pattern matching** -- circular trading (wash trades) is detected by finding cycles in the graph. This is the canonical graph use case and is dramatically more efficient than the equivalent self-join chain in pure SQL.

10. **Dual deployment option** -- the architecture supports both in-PostgreSQL graphs (simpler operations, good for MVP) and Neo4j (better traversal performance, better for production at scale). The transition path is clear: move from `graph_node` / `graph_edge` tables to Neo4j nodes/relationships with the same property structures, switching query language from recursive CTEs to Cypher.
