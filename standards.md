# Standards & API Reference

> Project: AgriMarket Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 22000:2018 — Food Safety Management Systems**
- URL: https://www.iso.org/standard/65464.html
- Specifies requirements for a food safety management system applicable to all organisations in the food chain, from farm to fork. Relevant to agricultural commodity marketplaces whose listings involve food-grade produce requiring demonstrable food safety controls and hazard analysis (HACCP integration). Widely adopted: over 51,000 certified sites globally.

**ISO 22005:2007 — Traceability in the Feed and Food Chain**
- URL: https://www.iso.org/standard/36297.html
- Defines general principles and requirements for designing and implementing traceability systems in the feed and food chain. Directly applicable to commodity provenance tracking on an agricultural marketplace — from harvest and storage through transport to final buyer delivery. Enables food safety recall, quality assurance claims, and regulatory compliance.

**ISO 8000-100:2016 — Data Quality: Master Data Exchange**
- URL: https://www.iso.org/standard/62392.html
- Defines requirements for exchanging master data (product descriptions, counterparty identities, location data) between trading partners. Relevant to ensuring that commodity product listings, buyer and seller records, and location data on the platform meet interoperability and quality standards for cross-organisation data exchange.

**ISO 28219:2017 — Packaging: Labelling and Direct Product Marking**
- URL: https://www.iso.org/standard/65014.html
- Specifies machine-readable labelling (linear barcodes and 2D symbols) for items, parts, and components. Applicable to physical commodity shipments that flow through marketplace transactions requiring scan-verifiable provenance, delivery confirmation, and inventory management at warehouse and logistics points.

**ISO 11783 (ISOBUS) — Agricultural Machinery Serial Data Network**
- URL: https://www.iso.org/standard/57556.html
- Communication protocol for tractors and agricultural machinery (based on SAE J1939/CAN bus), marketed as ISOBUS. While primarily a machinery-level standard, it is the foundation for precision agriculture data feeds (yield maps, field sensor data) that may be ingested by an AI-native marketplace for supply forecasting and quality prediction.

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines the semantics of HTTP methods, status codes, and headers underpinning all REST API interactions on the marketplace platform, including listing CRUD operations, order placement, and price feed subscriptions.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header for hypermedia-driven APIs. Relevant to paginated commodity listing feeds, HATEOAS-style marketplace APIs, and embedded resource linking in API responses.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Industry-standard protocol for delegated authorisation, enabling farmers and buyers to grant third-party agri-apps limited access to their marketplace accounts (grain marketing tools, ERP integrations, logistics providers) without sharing credentials.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for compact, self-contained access tokens widely used with OAuth 2.0 to authenticate API calls between marketplace services, mobile apps, and third-party integrations.

**W3C Verifiable Credentials Data Model**
- URL: https://www.w3.org/TR/vc-data-model-2.0/
- Emerging standard for tamper-evident, cryptographically verifiable digital certificates. Applicable to AI-native marketplace features for issuing and verifying digital grain quality certificates, organic certifications, and export compliance documents attached to commodity listings.

### Data Model & API Specifications

**OpenAPI Specification 3.x (OAS3)**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- The dominant standard for describing REST APIs in machine-readable YAML/JSON. Agricultural commodity data providers (DTN, Twelve Data, Commodities-API) already publish OAS/Swagger documentation. The marketplace platform API should be specified in OAS3 to enable automated SDK generation, API documentation portals, and partner integration.

**GS1 EPCIS 2.0 — Electronic Product Code Information Services**
- URL: https://www.gs1.org/standards/epcis
- The global standard for capturing and sharing supply chain traceability events (receiving, packing, shipping, transforming). EPCIS 2.0 adds a Web API with JSON/JSON-LD payloads and IoT sensor data support (temperature, humidity), making it the key standard for integrating real-time commodity movement data from logistics partners into marketplace transaction records.

**GS1 Global Traceability Standard (GTS2)**
- URL: https://www.gs1.org/standards/gs1-global-traceability-standard/current-standard
- Defines Critical Tracking Events (CTEs) and Key Data Elements (KDEs) as the canonical model for interoperable supply chain traceability. GTIN (Global Trade Item Number) and GLN (Global Location Number) uniquely identify commodities and locations flowing through marketplace transactions.

**FIX Protocol / FIXML**
- URL: https://www.fixtrading.org/standards/
- The Financial Information eXchange (FIX) protocol is the established standard for electronic communications in financial markets including commodity futures and options (CME, NYMEX, CBOT). FIXML is the XML encoding adopted globally for derivatives post-trade clearing and settlement. Relevant to any marketplace feature integrating with exchange-traded agricultural futures for hedging or price reference.

**ISO 20022 — Financial Messaging Standard**
- URL: https://www.swift.com/standards/iso-20022/iso-20022-standards
- Since November 2025, the exclusive standard for high-value cross-border payments on the SWIFT network, replacing legacy MT formats. Directly relevant to cross-border agricultural commodity settlement flows between buyer and seller parties in different jurisdictions.

### Security & Authentication Standards

**OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- Authentication layer built on OAuth 2.0 providing identity tokens for farmer and buyer user accounts. The standard for federated login (Google, Microsoft, government digital IDs) that reduces onboarding friction for participants in emerging markets.

**Financial-Grade API (FAPI 2.0)**
- URL: https://fapi.openid.net/
- Enhanced OAuth 2.0 security profile designed for high-assurance financial API access (payment initiation, sensitive data sharing). Applicable to embedded supply chain finance features (early payment, invoice factoring) within the marketplace that require elevated API security beyond standard OAuth.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- De facto checklist of the ten most critical API security risks (broken object level authorisation, authentication flaws, injection, etc.). Essential reference for securing the commodity listing, bidding, order, and payment APIs against the adversarial environment of high-value commodity transactions.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.eu/
- Applies to personal data of farmers and buyers in the EU/EEA. Agricultural marketplace data (farm location, crop yields, income indicators) frequently constitutes personal data under GDPR's broad definition. Compliance requires explicit consent, data minimisation, right to erasure, and cross-border transfer safeguards. Relevant for any EU market launch.

### WTO Trade Compliance Frameworks

**WTO SPS Agreement — Sanitary and Phytosanitary Measures**
- URL: https://www.wto.org/english/tratop_e/sps_e/spsund_e.htm
- Governs food safety, animal health, and plant health measures affecting agricultural commodity trade between WTO member countries. Commodity quality and certification claims on the marketplace must align with Codex Alimentarius, WOAH, and IPPC standards referenced by the SPS Agreement.

**WTO TBT Agreement — Technical Barriers to Trade**
- URL: https://www.wto.org/english/tratop_e/tbt_e/tbt_e.htm
- Covers technical regulations, quality grading standards, and labelling requirements for agricultural commodities in cross-border trade. Marketplace product listings for export must reflect compliance with applicable national and international technical standards.

**Codex Alimentarius**
- URL: https://www.fao.org/fao-who-codexalimentarius
- The international food safety and quality standards body (FAO/WHO), comprising 221 commodity standards and 78 guidelines. The WTO SPS Agreement designates Codex as the reference for food safety in trade disputes. Marketplace commodity grades, quality claims, and export certificates must be aligned with Codex commodity standards for the relevant crop.

### MCP Server Specifications

**Model Context Protocol (MCP) — 2025-11-25 Specification**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- Open standard (donated to the Linux Foundation / Agentic AI Foundation in December 2025) for connecting AI systems to external tools and data sources. Highly relevant for an AI-native agricultural marketplace: MCP servers can expose commodity price feeds, farmer supply data, weather APIs, logistics partner data, and government market data (Agmarknet, USDA AMS) as structured tools callable by AI agents performing match-making, pricing, and trade finance recommendations. Over 10,000 public MCP servers exist as of March 2026, with enterprise-wide adoption accelerating.

---

## Similar Products — Developer Documentation & APIs

### CME Group Market Data API

- **Description:** The Chicago Mercantile Exchange provides benchmark price discovery for agricultural commodity futures and options (corn, wheat, soybeans, cattle, hogs, dairy, and more). CME data is the global reference for agricultural commodity pricing.
- **API Documentation:** https://www.cmegroup.com/market-data/market-data-api.html
- **Real-Time WebSocket API:** https://www.cmegroup.com/market-data/real-time-futures-and-options-data-api.html
- **Reference Data API:** https://www.cmegroup.com/trading/market-tech-and-data-services/cme-reference-data-api.html
- **Historical Data (DataMine):** https://www.cmegroup.com/market-data/browse-data/agriculture-data.html
- **Standards:** REST/JSON, WebSocket/JSON, FIXML (clearing)
- **Authentication:** API key via self-service portal; subscription-based access tiers

### DTN Markets API

- **Description:** DTN is a major agricultural market data provider delivering cash grain bids, futures quotes, options data, and market scans for farming, agri-business, and trading applications. The Markets API covers grain elevator cash bids by location — a critical data type for local price discovery on agricultural marketplaces.
- **API Documentation:** https://cs-docs.dtn.com/apis/markets-api
- **Grain Bids REST API:** https://cs-docs.dtn.com/api/rest-api-for-markets-grain-bids
- **Developer Portal:** https://devportal.dtn.com/catalog/Agriculture/agcore/documentation
- **Standards:** REST/JSON, OpenAPI 3 (Swagger UI available)
- **Authentication:** API key (passed as `apikey` query parameter or header)

### USDA Agricultural Marketing Service (AMS) — MyMarketNews API

- **Description:** The US Department of Agriculture publishes free, unbiased market data for hundreds of agricultural commodities through its Market News service. The MARS API provides historical time series data for grains, livestock, fruits, and vegetables from regulated US markets.
- **API Documentation:** https://mymarketnews.ams.usda.gov/mymarketnews-api
- **Getting Started:** https://mymarketnews.ams.usda.gov/mars-api/getting-started/technical-instructions
- **LMPR API (Livestock, Poultry, Grain):** https://mpr.datamart.ams.usda.gov/
- **Standards:** REST/JSON, XLSX output available
- **Authentication:** API key (MyMarketNews); LMPR API is open/unrestricted

### India Agmarknet API (data.gov.in)

- **Description:** The Indian government's Agricultural Marketing Information Network (Agmarknet) provides daily wholesale commodity prices from over 3,000 regulated markets (mandis) nationwide. Covers 200+ commodities with minimum, maximum, and modal prices. Essential reference for emerging-market agricultural marketplace development.
- **API Documentation:** https://www.data.gov.in/apis/?sector=Agriculture
- **Commodity Price Resource:** https://www.data.gov.in/resource/current-daily-price-various-commodities-various-markets-mandi
- **Standards:** REST/JSON, CSV
- **Authentication:** API key (data.gov.in registration)

### Commodities-API

- **Description:** Commercial REST API providing real-time and historical prices for 100+ commodities in 170 currencies, including agricultural commodities (wheat, corn, soybeans, coffee, sugar, palm oil, rice, rubber). Supports currency conversion, time-series, and fluctuation endpoints. Designed for fast developer integration.
- **API Documentation:** https://commodities-api.com/documentation
- **Standards:** REST/JSON
- **Authentication:** API key; free tier and paid subscription tiers available
- **SDKs:** Code examples in multiple languages; integration in under 10 minutes claimed

### Twelve Data — Commodities API

- **Description:** Twelve Data provides real-time and historical market data for stocks, forex, ETFs, and commodities including agricultural futures (corn, wheat, soybeans, cocoa, coffee, sugar, cotton, and more). Designed for developers and financial institutions needing reliable commodity pricing.
- **API Documentation:** https://twelvedata.com/docs
- **Commodities Overview:** https://twelvedata.com/commodities
- **Request Builder:** https://twelvedata.com/request-builder
- **Standards:** REST/JSON, WebSocket for real-time streaming
- **SDKs:** Python, JavaScript, PHP; available on RapidAPI
- **Authentication:** API key; tiered pricing (free, basic, pro, enterprise)

### Tradyon (Agricultural Commodity Trader Platform)

- **Description:** AI-first platform launched January 2026 for agricultural commodity traders (exporters, importers, trading firms) focused on buyer/seller discovery, trade context management, market signals, and follow-up automation. Active in spices, pulses, coffee, and seafood markets.
- **Company/Platform:** https://www.tradyon.com (UAE-headquartered, founded 2025)
- **Developer Documentation:** No public API documentation found as of research date; platform is primarily a closed SaaS application
- **Standards:** Not publicly disclosed
- **Authentication:** Not publicly disclosed
- **Note:** A relevant emerging competitor to monitor; public API availability expected as platform matures

### Open Food Facts API

- **Description:** Open-source, crowd-sourced database of food products with ingredients, nutritional values, allergens, and labels. Not a commodity trading platform, but provides a free reference data layer for product identification and quality attributes relevant to value-added agricultural products on a marketplace.
- **API Documentation:** https://openfoodfacts.github.io/openfoodfacts-server/api/
- **GitHub:** https://github.com/openfoodfacts/openfoodfacts-server
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** Open read access; authenticated write for contributions
- **SDKs:** Dart, multiple community SDKs available via pub.dev and GitHub

### GS1 EPCIS / OpenEPCIS

- **Description:** OpenEPCIS is an open-source, GS1-compliant EPCIS 2.0 implementation for supply chain traceability. EPCIS 2.0 introduces a REST/JSON-LD Web API with IoT sensor data support. The key integration standard for connecting commodity logistics partners (warehouses, transporters, grain elevators) to marketplace transaction records.
- **EPCIS Standard:** https://www.gs1.org/standards/epcis
- **OpenEPCIS (Open-source implementation):** https://openepcis.io/
- **Standards:** REST, JSON-LD, EPCIS 2.0 event model (What/When/Where/Why)
- **Authentication:** Implementation-dependent; reference implementation is open-source under Apache 2.0

---

## Notes

- **Emerging standard to watch:** The EU Digital Product Passport (DPP) regulation (coming into force progressively from 2026–2030) will mandate machine-readable product lifecycle data for a growing range of product categories, potentially including agricultural commodities in regulated supply chains. ISO 8000 master data quality standards are explicitly referenced as aligned with DPP requirements.
- **FAPI adoption gap:** Financial-grade API security (FAPI 2.0) is well established in open banking but not yet widely adopted by agricultural fintech platforms. An AI-native marketplace offering embedded supply chain finance could gain a security-differentiation advantage by adopting FAPI from the outset.
- **Agmarknet API limitations:** The Indian government Agmarknet API is a critical reference for emerging market price data but has known reliability and data completeness challenges. Production systems typically supplement it with private data partnerships or scraping, pending government API improvement.
- **No public FBN or agribazaar APIs found:** Farmers Business Network (FBN) and agribazaar, two significant agricultural marketplace incumbents, do not publish public developer APIs as of the research date. Integration into their ecosystems likely requires direct business partnerships.
