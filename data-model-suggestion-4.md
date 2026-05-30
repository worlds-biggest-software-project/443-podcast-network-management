# Data Model Suggestion 4: Polyglot Persistence — PostgreSQL + ClickHouse + Neo4j

> Project: Podcast Network Management (Candidate #443)
> Generated: 2026-05-25

## Overview

A domain-specific polyglot persistence architecture that assigns each data domain to the storage engine best suited to its access patterns:

| Domain | Storage Engine | Why |
|--------|---------------|-----|
| Operational core (shows, campaigns, royalties, payouts) | **PostgreSQL** | ACID transactions, referential integrity, financial accuracy |
| Ad impressions, download analytics, time-series metrics | **ClickHouse** | Columnar compression, sub-second aggregation over billions of rows, native partitioning |
| Content relationships, audience overlap, advertiser-show matching | **Neo4j** | Graph traversal for recommendation, overlap analysis, network effects |
| AI embeddings and similarity search | **pgvector** (in PostgreSQL) | Avoid separate vector DB; co-located with operational data |
| Real-time deduplication and caching | **Redis** | IAB v2.2 24-hour dedup windows, session counters, real-time dashboard caches |

This approach is motivated by the specific workload characteristics of podcast network management:

1. **Impression analytics is a write-heavy, aggregation-heavy time-series problem.** A mid-size network (50 shows, 10 active campaigns) generates 20-50M impression rows per month. ClickHouse handles this with 10-20x better compression than PostgreSQL and orders-of-magnitude faster aggregation without materialized views.

2. **Audience overlap and advertiser matching are graph problems.** "Which listeners of Show A also listen to Show B?" and "Which advertisers should target Show C based on audience similarity?" are multi-hop graph traversals that are awkward in SQL but natural in Cypher.

3. **Financial data demands ACID.** Royalty calculations, payout approvals, and campaign reconciliations must be transactionally correct. PostgreSQL provides this guarantee; ClickHouse and Neo4j do not.

4. **IAB deduplication is a real-time windowed uniqueness problem.** Redis sorted sets with TTL provide the fastest path for 24-hour sliding-window deduplication at scale.

---

## Architecture Diagram

```
                         ┌──────────────────────┐
                         │     API Gateway       │
                         │  (Node.js / Fastify)  │
                         └──────────┬─────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           │                        │                        │
    ┌──────▼──────┐          ┌──────▼──────┐          ┌──────▼──────┐
    │  Operational │          │  Analytics  │          │   Graph     │
    │   Service    │          │   Service   │          │   Service   │
    └──────┬──────┘          └──────┬──────┘          └──────┬──────┘
           │                        │                        │
    ┌──────▼──────┐          ┌──────▼──────┐          ┌──────▼──────┐
    │ PostgreSQL  │          │  ClickHouse │          │    Neo4j    │
    │  (primary)  │◄─CDC────►│  (analytics)│          │   (graph)   │
    └─────────────┘          └─────────────┘          └─────────────┘
           │                        ▲                        ▲
           │                        │                        │
           │                 ┌──────┴──────┐                 │
           │                 │    Kafka     │                 │
           │                 │ (impressions)│─────────────────┘
           │                 └─────────────┘
           │                        ▲
           │                 ┌──────┴──────┐
           └────────────────►│    Redis    │
                             │   (dedup)   │
                             └─────────────┘
```

---

## PostgreSQL Schema (Operational Core)

```sql
-- ============================================================
-- PostgreSQL: Financial & Operational Data
-- ============================================================
-- This is a streamlined version of the relational core from Suggestion 1.
-- Impression and analytics tables are NOT here — they live in ClickHouse.
-- Graph relationships are NOT here — they live in Neo4j.

CREATE TABLE networks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    description         TEXT,
    logo_url            VARCHAR(2048),
    website_url         VARCHAR(2048),
    default_currency    VARCHAR(3) NOT NULL DEFAULT 'USD',
    timezone            VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    settings            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(320) NOT NULL UNIQUE,
    name                VARCHAR(255) NOT NULL,
    avatar_url          VARCHAR(2048),
    password_hash       VARCHAR(255),
    auth_metadata       JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE network_memberships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    role                VARCHAR(30) NOT NULL DEFAULT 'analyst',
    permissions         JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (network_id, user_id)
);

CREATE TABLE shows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,
    language            VARCHAR(10) NOT NULL DEFAULT 'en',
    explicit            BOOLEAN NOT NULL DEFAULT FALSE,
    cover_image_url     VARCHAR(2048),
    rss_feed_url        VARCHAR(2048),
    hosting_platform    VARCHAR(50) NOT NULL DEFAULT 'self_hosted',
    hosting_external_id VARCHAR(255),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    podcast_metadata    JSONB NOT NULL DEFAULT '{}',
    hosting_config      JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    UNIQUE (network_id, slug)
);

CREATE TABLE show_hosts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    user_id             UUID REFERENCES users(id),
    person_name         VARCHAR(255) NOT NULL,
    person_role         VARCHAR(100) NOT NULL DEFAULT 'host',
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    started_at          DATE,
    ended_at            DATE,
    person_metadata     JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE episodes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    season_number       INTEGER,
    episode_number      INTEGER,
    audio_url           VARCHAR(2048),
    audio_duration_secs INTEGER,
    audio_size_bytes    BIGINT,
    audio_mime_type     VARCHAR(100) DEFAULT 'audio/mpeg',
    guid                VARCHAR(500) NOT NULL,
    published_at        TIMESTAMPTZ,
    episode_metadata    JSONB NOT NULL DEFAULT '{}',
    ai_annotations      JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    UNIQUE (show_id, slug)
);

CREATE INDEX idx_episodes_show_published ON episodes(show_id, published_at DESC)
    WHERE deleted_at IS NULL;

CREATE TABLE ad_insertion_points (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    position            VARCHAR(20) NOT NULL,
    offset_secs         NUMERIC(10,3),
    max_duration_secs   INTEGER DEFAULT 60,
    is_auto_detected    BOOLEAN NOT NULL DEFAULT FALSE,
    confidence_score    NUMERIC(5,4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE advertisers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    company_name        VARCHAR(500) NOT NULL,
    contact_name        VARCHAR(255),
    contact_email       VARCHAR(320),
    industry_category   VARCHAR(255),
    billing_currency    VARCHAR(3) NOT NULL DEFAULT 'USD',
    advertiser_profile  JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE campaigns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    advertiser_id       UUID NOT NULL REFERENCES advertisers(id),
    name                VARCHAR(500) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    pricing_model       VARCHAR(30) NOT NULL DEFAULT 'cpm',
    inventory_type      VARCHAR(30) NOT NULL DEFAULT 'guaranteed',
    cpm_rate            NUMERIC(10,4),
    flat_rate_amount    NUMERIC(12,2),
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    booked_impressions  BIGINT,
    budget_amount       NUMERIC(12,2),
    flight_start        DATE NOT NULL,
    flight_end          DATE NOT NULL,
    campaign_config     JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT valid_flight CHECK (flight_end >= flight_start)
);

CREATE TABLE campaign_show_targets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id),
    show_id             UUID NOT NULL REFERENCES shows(id),
    ad_position         VARCHAR(20) NOT NULL DEFAULT 'mid_roll',
    allocated_impressions BIGINT,
    weight              NUMERIC(5,4) DEFAULT 1.0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (campaign_id, show_id, ad_position)
);

-- === Royalty Management (fully normalized for financial integrity) ===

CREATE TABLE royalty_rules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    effective_from      DATE NOT NULL,
    effective_until     DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    rule_config         JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE royalty_splits (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    royalty_rule_id     UUID NOT NULL REFERENCES royalty_rules(id),
    recipient_host_id   UUID NOT NULL REFERENCES show_hosts(id),
    split_percentage    NUMERIC(7,4) NOT NULL,
    minimum_amount      NUMERIC(10,2) DEFAULT 0,
    cap_amount          NUMERIC(10,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT valid_pct CHECK (split_percentage > 0 AND split_percentage <= 100)
);

CREATE TABLE royalty_calculations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    royalty_rule_id     UUID NOT NULL REFERENCES royalty_rules(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    total_revenue       NUMERIC(14,2) NOT NULL,
    ad_revenue          NUMERIC(14,2) NOT NULL DEFAULT 0,
    subscription_revenue NUMERIC(14,2) NOT NULL DEFAULT 0,
    other_revenue       NUMERIC(14,2) NOT NULL DEFAULT 0,
    network_commission  NUMERIC(14,2) NOT NULL DEFAULT 0,
    distributable_amount NUMERIC(14,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    revenue_breakdown   JSONB NOT NULL DEFAULT '{}',
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE royalty_line_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calculation_id      UUID NOT NULL REFERENCES royalty_calculations(id),
    recipient_host_id   UUID NOT NULL REFERENCES show_hosts(id),
    split_percentage    NUMERIC(7,4) NOT NULL,
    gross_amount        NUMERIC(14,2) NOT NULL,
    adjustments         NUMERIC(14,2) NOT NULL DEFAULT 0,
    net_amount          NUMERIC(14,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    adjustment_detail   JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- === Payment Disbursement ===

CREATE TABLE payment_profiles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    preferred_currency  VARCHAR(3) NOT NULL DEFAULT 'USD',
    payment_method      VARCHAR(50) NOT NULL DEFAULT 'bank_transfer',
    payment_config      JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payouts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    recipient_user_id   UUID NOT NULL REFERENCES users(id),
    payment_profile_id  UUID NOT NULL REFERENCES payment_profiles(id),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    amount              NUMERIC(14,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL,
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    processing_detail   JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- === Revenue & P&L ===

CREATE TABLE revenue_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    show_id             UUID REFERENCES shows(id),
    source_type         VARCHAR(50) NOT NULL,
    source_id           UUID,
    amount              NUMERIC(14,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    revenue_date        DATE NOT NULL,
    description         TEXT,
    source_detail       JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (revenue_date);

CREATE TABLE expense_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    show_id             UUID REFERENCES shows(id),
    category            VARCHAR(100) NOT NULL,
    amount              NUMERIC(14,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    expense_date        DATE NOT NULL,
    description         TEXT,
    reference_id        UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (expense_date);

-- === Subscriptions ===

CREATE TABLE subscription_plans (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID REFERENCES shows(id),
    network_id          UUID NOT NULL REFERENCES networks(id),
    name                VARCHAR(255) NOT NULL,
    tier                VARCHAR(20) NOT NULL DEFAULT 'supporter',
    price_amount        NUMERIC(10,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    billing_interval    VARCHAR(20) NOT NULL DEFAULT 'monthly',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    plan_config         JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id             UUID NOT NULL REFERENCES subscription_plans(id),
    subscriber_email    VARCHAR(320) NOT NULL,
    subscriber_name     VARCHAR(255),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    stripe_subscription_id VARCHAR(255),
    started_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    current_period_start TIMESTAMPTZ,
    current_period_end  TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ,
    subscriber_metadata JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- === Content Embeddings (pgvector) ===

CREATE TABLE content_embeddings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id) UNIQUE,
    embedding_vector    vector(3072),
    embedding_model     VARCHAR(100) NOT NULL,
    transcript_hash     VARCHAR(64),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_embeddings_vector ON content_embeddings
    USING hnsw (embedding_vector vector_cosine_ops);

-- === Audit Log ===

CREATE TABLE audit_log (
    id                  BIGSERIAL,
    network_id          UUID,
    user_id             UUID,
    action              VARCHAR(100) NOT NULL,
    entity_type         VARCHAR(100) NOT NULL,
    entity_id           UUID NOT NULL,
    changes             JSONB,
    request_metadata    JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

---

## ClickHouse Schema (Analytics & Impressions)

```sql
-- ============================================================
-- ClickHouse: Ad Impressions
-- ============================================================
-- ClickHouse handles the high-volume, write-heavy, aggregation-heavy
-- impression and analytics workloads where PostgreSQL would struggle.

CREATE DATABASE IF NOT EXISTS podcast_analytics;

-- Ad impression fact table
CREATE TABLE podcast_analytics.ad_impressions (
    -- Identifiers
    impression_id       UUID,
    campaign_id         UUID,
    episode_id          UUID,
    show_id             UUID,
    network_id          UUID,
    insertion_point_id  Nullable(UUID),
    
    -- Dimensions (LowCardinality for high compression)
    ad_position         LowCardinality(String),     -- 'pre_roll', 'mid_roll', 'post_roll'
    country_code        LowCardinality(String),
    region_code         LowCardinality(String),
    city                LowCardinality(String),
    device_type         LowCardinality(String),     -- 'mobile', 'desktop', 'smart_speaker'
    app_name            LowCardinality(String),     -- 'Apple Podcasts', 'Spotify', etc.
    os_name             LowCardinality(String),
    connection_type     LowCardinality(String),     -- 'wifi', 'cellular'
    
    -- IAB compliance
    listener_ip_hash    String,
    user_agent_hash     String,
    is_valid            UInt8 DEFAULT 1,            -- 1 = valid, 0 = invalid
    invalid_reason      LowCardinality(String) DEFAULT '',  -- 'iab_dedup', 'bot', 'fraud'
    
    -- VAST tracking events
    vast_impression     UInt8 DEFAULT 0,
    vast_first_quartile UInt8 DEFAULT 0,
    vast_midpoint       UInt8 DEFAULT 0,
    vast_third_quartile UInt8 DEFAULT 0,
    vast_complete       UInt8 DEFAULT 0,
    
    -- OpenRTB details (for programmatic)
    openrtb_deal_id     LowCardinality(String) DEFAULT '',
    openrtb_seat_id     LowCardinality(String) DEFAULT '',
    openrtb_win_price   Float64 DEFAULT 0,
    
    -- Timestamps
    delivered_at        DateTime64(3, 'UTC'),
    delivered_date      Date MATERIALIZED toDate(delivered_at)
    
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(delivered_at)
ORDER BY (network_id, campaign_id, show_id, delivered_at)
TTL delivered_at + INTERVAL 24 MONTH DELETE
SETTINGS index_granularity = 8192;

-- Secondary indexes for common query patterns
ALTER TABLE podcast_analytics.ad_impressions
    ADD INDEX idx_dedup (listener_ip_hash, user_agent_hash)
    TYPE bloom_filter GRANULARITY 4;

ALTER TABLE podcast_analytics.ad_impressions
    ADD INDEX idx_country (country_code)
    TYPE set(100) GRANULARITY 4;

-- ============================================================
-- ClickHouse: Episode Download/Listen Analytics
-- ============================================================

CREATE TABLE podcast_analytics.episode_downloads (
    -- Identifiers
    download_id         UUID,
    episode_id          UUID,
    show_id             UUID,
    network_id          UUID,
    source_platform     LowCardinality(String),     -- 'megaphone', 'libsyn', etc.
    
    -- Listener dimensions
    country_code        LowCardinality(String),
    region_code         LowCardinality(String),
    city                LowCardinality(String),
    device_type         LowCardinality(String),
    app_name            LowCardinality(String),
    os_name             LowCardinality(String),
    
    -- IAB compliance
    listener_ip_hash    String,
    user_agent_hash     String,
    is_valid            UInt8 DEFAULT 1,
    iab_version         LowCardinality(String),     -- '2.1', '2.2'
    
    -- Engagement metrics (when available)
    listen_duration_secs    UInt32 DEFAULT 0,
    completion_pct          Float32 DEFAULT 0,       -- 0.0 to 100.0
    
    -- Timestamps
    downloaded_at       DateTime64(3, 'UTC'),
    downloaded_date     Date MATERIALIZED toDate(downloaded_at)
    
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(downloaded_at)
ORDER BY (network_id, show_id, episode_id, downloaded_at)
TTL downloaded_at + INTERVAL 36 MONTH DELETE
SETTINGS index_granularity = 8192;

-- ============================================================
-- ClickHouse: Aggregated Daily Metrics (MaterializedView)
-- ============================================================

-- Campaign daily delivery summary
CREATE MATERIALIZED VIEW podcast_analytics.mv_campaign_daily
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(delivery_date)
ORDER BY (network_id, campaign_id, show_id, ad_position, delivery_date)
AS SELECT
    network_id,
    campaign_id,
    show_id,
    ad_position,
    delivered_date AS delivery_date,
    count() AS total_impressions,
    countIf(is_valid = 1) AS valid_impressions,
    countIf(is_valid = 0) AS invalid_impressions,
    uniqHLL12(listener_ip_hash) AS approx_unique_listeners,
    countIf(vast_complete = 1) AS completed_impressions,
    sumIf(openrtb_win_price, openrtb_win_price > 0) AS programmatic_revenue
FROM podcast_analytics.ad_impressions
GROUP BY network_id, campaign_id, show_id, ad_position, delivered_date;

-- Show daily download summary
CREATE MATERIALIZED VIEW podcast_analytics.mv_show_daily_downloads
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(download_date)
ORDER BY (network_id, show_id, episode_id, source_platform, download_date)
AS SELECT
    network_id,
    show_id,
    episode_id,
    source_platform,
    downloaded_date AS download_date,
    count() AS total_downloads,
    countIf(is_valid = 1) AS valid_downloads,
    uniqHLL12(listener_ip_hash) AS approx_unique_listeners,
    avg(completion_pct) AS avg_completion_pct,
    avg(listen_duration_secs) AS avg_listen_duration
FROM podcast_analytics.episode_downloads
GROUP BY network_id, show_id, episode_id, source_platform, downloaded_date;

-- Geographic breakdown
CREATE MATERIALIZED VIEW podcast_analytics.mv_geo_daily
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(download_date)
ORDER BY (network_id, show_id, country_code, region_code, download_date)
AS SELECT
    network_id,
    show_id,
    country_code,
    region_code,
    downloaded_date AS download_date,
    count() AS total_downloads,
    uniqHLL12(listener_ip_hash) AS approx_unique_listeners
FROM podcast_analytics.episode_downloads
GROUP BY network_id, show_id, country_code, region_code, downloaded_date;

-- App breakdown
CREATE MATERIALIZED VIEW podcast_analytics.mv_app_daily
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(download_date)
ORDER BY (network_id, show_id, app_name, device_type, download_date)
AS SELECT
    network_id,
    show_id,
    app_name,
    device_type,
    downloaded_date AS download_date,
    count() AS total_downloads,
    uniqHLL12(listener_ip_hash) AS approx_unique_listeners
FROM podcast_analytics.episode_downloads
GROUP BY network_id, show_id, app_name, device_type, downloaded_date;

-- ============================================================
-- ClickHouse: Anomaly Detection Support
-- ============================================================

-- Rolling hourly stats for anomaly detection
CREATE MATERIALIZED VIEW podcast_analytics.mv_hourly_stats
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (network_id, show_id, hour)
TTL hour + INTERVAL 90 DAY DELETE
AS SELECT
    network_id,
    show_id,
    toStartOfHour(downloaded_at) AS hour,
    count() AS download_count,
    uniqHLL12(listener_ip_hash) AS unique_ips,
    -- High ratio of downloads to unique IPs suggests bot traffic
    count() / greatest(uniqHLL12(listener_ip_hash), 1) AS downloads_per_ip_ratio
FROM podcast_analytics.episode_downloads
GROUP BY network_id, show_id, toStartOfHour(downloaded_at);
```

### ClickHouse Query Examples

```sql
-- Campaign delivery report: How is campaign X performing against its booking?
SELECT
    show_id,
    sum(valid_impressions) AS total_valid,
    sum(total_impressions) AS total_delivered,
    sum(approx_unique_listeners) AS unique_reach,
    sum(completed_impressions) / greatest(sum(valid_impressions), 1) * 100 AS completion_rate_pct
FROM podcast_analytics.mv_campaign_daily
WHERE campaign_id = '...'
  AND delivery_date BETWEEN '2026-05-01' AND '2026-05-31'
GROUP BY show_id
ORDER BY total_valid DESC;

-- Network-wide daily downloads (single pane of glass across all platforms)
SELECT
    download_date,
    source_platform,
    sum(total_downloads) AS downloads,
    sum(approx_unique_listeners) AS unique_listeners,
    avg(avg_completion_pct) AS avg_completion
FROM podcast_analytics.mv_show_daily_downloads
WHERE network_id = '...'
  AND download_date BETWEEN '2026-04-01' AND '2026-05-25'
GROUP BY download_date, source_platform
ORDER BY download_date DESC;

-- Geographic heatmap data for a show
SELECT
    country_code,
    region_code,
    sum(total_downloads) AS downloads
FROM podcast_analytics.mv_geo_daily
WHERE show_id = '...'
  AND download_date BETWEEN '2026-05-01' AND '2026-05-25'
GROUP BY country_code, region_code
ORDER BY downloads DESC
LIMIT 50;

-- Anomaly detection: Find shows with suspicious download-to-IP ratios
SELECT
    show_id,
    hour,
    download_count,
    unique_ips,
    downloads_per_ip_ratio
FROM podcast_analytics.mv_hourly_stats
WHERE network_id = '...'
  AND hour >= now() - INTERVAL 24 HOUR
  AND downloads_per_ip_ratio > 10  -- Threshold for investigation
ORDER BY downloads_per_ip_ratio DESC;

-- IAB v2.2 compliance report: valid vs. invalid impression breakdown
SELECT
    invalid_reason,
    count() AS impression_count,
    count() * 100.0 / sum(count()) OVER () AS pct_of_total
FROM podcast_analytics.ad_impressions
WHERE campaign_id = '...'
  AND delivered_at BETWEEN '2026-05-01' AND '2026-05-31'
  AND is_valid = 0
GROUP BY invalid_reason
ORDER BY impression_count DESC;
```

---

## Neo4j Schema (Content Graph)

```cypher
// ============================================================
// Neo4j: Graph Model for Content Relationships & Recommendations
// ============================================================

// --- Node Types ---

// Network
CREATE CONSTRAINT network_id IF NOT EXISTS
FOR (n:Network) REQUIRE n.id IS UNIQUE;

// Show
CREATE CONSTRAINT show_id IF NOT EXISTS
FOR (s:Show) REQUIRE s.id IS UNIQUE;

CREATE INDEX show_category IF NOT EXISTS
FOR (s:Show) ON (s.category);

// Episode
CREATE CONSTRAINT episode_id IF NOT EXISTS
FOR (e:Episode) REQUIRE e.id IS UNIQUE;

// Host/Person
CREATE CONSTRAINT person_id IF NOT EXISTS
FOR (p:Person) REQUIRE p.id IS UNIQUE;

CREATE INDEX person_name IF NOT EXISTS
FOR (p:Person) ON (p.name);

// Advertiser
CREATE CONSTRAINT advertiser_id IF NOT EXISTS
FOR (a:Advertiser) REQUIRE a.id IS UNIQUE;

// Category/Topic
CREATE CONSTRAINT category_name IF NOT EXISTS
FOR (c:Category) REQUIRE c.name IS UNIQUE;

// Listener Cohort (anonymized audience segment)
CREATE CONSTRAINT cohort_id IF NOT EXISTS
FOR (lc:ListenerCohort) REQUIRE lc.id IS UNIQUE;

// --- Relationship Types ---

// Network -> Show
// (:Network)-[:OPERATES]-(:Show)

// Show -> Episode
// (:Show)-[:PUBLISHED]-(:Episode)

// Person -> Show (with role property)
// (:Person)-[:HOSTS {role: 'host', isPrimary: true, since: date('2024-01-01')}]-(:Show)

// Person -> Episode (guest appearances)
// (:Person)-[:APPEARED_ON {role: 'guest'}]-(:Episode)

// Show -> Category (content classification)
// (:Show)-[:IN_CATEGORY]-(:Category)

// Category -> Category (hierarchy)
// (:Category)-[:SUBCATEGORY_OF]-(:Category)

// Advertiser -> Category (targeting preferences)
// (:Advertiser)-[:TARGETS {preference: 'primary'}]-(:Category)
// (:Advertiser)-[:EXCLUDES]-(:Category)

// Advertiser -> Show (campaign history)
// (:Advertiser)-[:ADVERTISED_ON {campaignId: 'xxx', startDate: date, endDate: date, spend: 5000}]-(:Show)

// Show -> Show (cross-promotion, podroll)
// (:Show)-[:CROSS_PROMOTES {type: 'podroll'}]-(:Show)

// ListenerCohort -> Show (audience presence)
// (:ListenerCohort)-[:LISTENS_TO {downloadCount: 500, avgCompletion: 0.78}]-(:Show)

// Advertiser -> Advertiser (competitive exclusion)
// (:Advertiser)-[:COMPETES_WITH]-(:Advertiser)


// ============================================================
// Seed example data
// ============================================================

// Create network and shows
CREATE (n:Network {id: 'net-1', name: 'TechPod Network'})
CREATE (s1:Show {id: 'show-1', title: 'Tech Talk Daily', category: 'Technology',
    monthlyDownloads: 250000, hostCount: 2})
CREATE (s2:Show {id: 'show-2', title: 'Code & Coffee', category: 'Technology',
    monthlyDownloads: 80000, hostCount: 1})
CREATE (s3:Show {id: 'show-3', title: 'Founder Stories', category: 'Business',
    monthlyDownloads: 150000, hostCount: 2})

CREATE (n)-[:OPERATES]->(s1)
CREATE (n)-[:OPERATES]->(s2)
CREATE (n)-[:OPERATES]->(s3)

// Create categories
CREATE (tech:Category {name: 'Technology'})
CREATE (sw:Category {name: 'Software How-To'})
CREATE (biz:Category {name: 'Business'})
CREATE (startup:Category {name: 'Entrepreneurship'})

CREATE (sw)-[:SUBCATEGORY_OF]->(tech)
CREATE (startup)-[:SUBCATEGORY_OF]->(biz)

CREATE (s1)-[:IN_CATEGORY]->(tech)
CREATE (s1)-[:IN_CATEGORY]->(sw)
CREATE (s2)-[:IN_CATEGORY]->(tech)
CREATE (s3)-[:IN_CATEGORY]->(biz)
CREATE (s3)-[:IN_CATEGORY]->(startup)

// Create listener cohorts for audience overlap
CREATE (lc1:ListenerCohort {id: 'cohort-tech-sf', description: 'Tech professionals, SF Bay Area',
    size: 15000})
CREATE (lc2:ListenerCohort {id: 'cohort-dev-global', description: 'Software developers, global',
    size: 25000})
CREATE (lc3:ListenerCohort {id: 'cohort-founders', description: 'Startup founders & execs',
    size: 8000})

CREATE (lc1)-[:LISTENS_TO {downloadCount: 12000, avgCompletion: 0.82}]->(s1)
CREATE (lc1)-[:LISTENS_TO {downloadCount: 5000, avgCompletion: 0.75}]->(s2)
CREATE (lc1)-[:LISTENS_TO {downloadCount: 3000, avgCompletion: 0.68}]->(s3)
CREATE (lc2)-[:LISTENS_TO {downloadCount: 8000, avgCompletion: 0.71}]->(s1)
CREATE (lc2)-[:LISTENS_TO {downloadCount: 7000, avgCompletion: 0.85}]->(s2)
CREATE (lc3)-[:LISTENS_TO {downloadCount: 2000, avgCompletion: 0.65}]->(s1)
CREATE (lc3)-[:LISTENS_TO {downloadCount: 6000, avgCompletion: 0.79}]->(s3);
```

### Neo4j Query Examples

```cypher
// ============================================================
// 1. Cross-show audience overlap analysis
// "What % of Show A's audience also listens to Show B?"
// ============================================================

MATCH (s1:Show {id: 'show-1'})<-[r1:LISTENS_TO]-(lc:ListenerCohort)-[r2:LISTENS_TO]->(s2:Show)
WHERE s2.id <> s1.id
RETURN
    s2.title AS overlapping_show,
    sum(r1.downloadCount) AS shared_listener_downloads,
    sum(r2.downloadCount) AS their_downloads_on_other,
    count(lc) AS shared_cohort_count
ORDER BY shared_listener_downloads DESC;


// ============================================================
// 2. Advertiser-show matching using graph relationships
// "Find shows that match advertiser targeting preferences"
// ============================================================

MATCH (a:Advertiser {id: 'adv-1'})-[:TARGETS]->(c:Category)<-[:IN_CATEGORY]-(s:Show)
WHERE NOT (a)-[:EXCLUDES]->(:Category)<-[:IN_CATEGORY]-(s)
  AND NOT (a)-[:COMPETES_WITH]->(:Advertiser)-[:ADVERTISED_ON]->(s)
  AND s.monthlyDownloads >= 10000
RETURN s.title, s.monthlyDownloads, collect(c.name) AS matching_categories
ORDER BY s.monthlyDownloads DESC;


// ============================================================
// 3. Network packaging proposal
// "Bundle shows for cross-network advertising deals"
// ============================================================

// Find shows with high audience overlap for bundled ad sales
MATCH (s1:Show)<-[:LISTENS_TO]-(lc:ListenerCohort)-[:LISTENS_TO]->(s2:Show)
WHERE s1.id < s2.id  // Avoid duplicates
WITH s1, s2, count(lc) AS shared_cohorts,
     sum(size(lc)) AS total_shared_audience
WHERE shared_cohorts >= 2
RETURN s1.title, s2.title, shared_cohorts, total_shared_audience
ORDER BY total_shared_audience DESC;


// ============================================================
// 4. Host network analysis
// "Find hosts who appear across multiple shows (collaboration graph)"
// ============================================================

MATCH (p:Person)-[h:HOSTS|APPEARED_ON]->(s:Show)<-[:OPERATES]-(n:Network)
WITH p, collect({show: s.title, role: h.role}) AS appearances
WHERE size(appearances) > 1
RETURN p.name, appearances
ORDER BY size(appearances) DESC;


// ============================================================
// 5. Content similarity path
// "Find shows similar to Show X through shared categories and hosts"
// ============================================================

MATCH path = (s1:Show {id: 'show-1'})-[:IN_CATEGORY|HOSTS*1..3]-(s2:Show)
WHERE s1 <> s2
RETURN DISTINCT s2.title, length(path) AS degrees_of_separation,
    [n IN nodes(path) | labels(n)[0] + ': ' + coalesce(n.title, n.name, '')] AS path_through
ORDER BY degrees_of_separation ASC
LIMIT 10;


// ============================================================
// 6. Competitive landscape for a campaign
// "Show all active advertiser campaigns to detect ad clashing"
// ============================================================

MATCH (a:Advertiser)-[r:ADVERTISED_ON]->(s:Show {id: 'show-1'})
WHERE r.endDate >= date()
OPTIONAL MATCH (a)-[:COMPETES_WITH]-(competitor:Advertiser)
RETURN a.companyName, r.startDate, r.endDate, r.spend,
    collect(DISTINCT competitor.companyName) AS competitors
ORDER BY r.startDate;
```

---

## Data Synchronization (CDC Pipeline)

```
PostgreSQL ──(Debezium CDC)──► Kafka ──► ClickHouse (analytics)
                                    ──► Neo4j (graph sync)
                                    ──► Redis (cache invalidation)
```

### CDC Configuration

```yaml
# Debezium connector configuration for PostgreSQL -> Kafka
connector.class: io.debezium.connector.postgresql.PostgresConnector
database.hostname: postgres-primary
database.port: 5432
database.dbname: podcast_network
database.server.name: podcast

# Tables to capture
table.include.list: >
  public.shows,
  public.episodes,
  public.show_hosts,
  public.campaigns,
  public.campaign_show_targets,
  public.advertisers,
  public.revenue_entries

# Transform: route to topic per table
transforms: route
transforms.route.type: io.debezium.transforms.ByLogicalTableRouter
transforms.route.topic.regex: podcast\\.public\\.(.*)
transforms.route.topic.replacement: podcast.cdc.$1
```

### Kafka Connect Sinks

```yaml
# ClickHouse sink (for analytics tables synced from PostgreSQL)
# Note: Impressions go directly to ClickHouse via application, not CDC
connector.class: com.clickhouse.kafka.connect.ClickHouseSinkConnector
topics: podcast.cdc.shows,podcast.cdc.episodes
clickhouse.server.url: clickhouse:8123
clickhouse.server.database: podcast_analytics

---

# Neo4j sink (graph relationship sync)
connector.class: streams.kafka.connect.sink.Neo4jSinkConnector
topics: podcast.cdc.shows,podcast.cdc.show_hosts,podcast.cdc.advertisers,podcast.cdc.campaign_show_targets
neo4j.server.uri: bolt://neo4j:7687
neo4j.topic.cypher.podcast.cdc.shows: >
  MERGE (s:Show {id: event.after.id})
  SET s.title = event.after.title,
      s.category = event.after.podcast_metadata::json->>'itunes'->>'category',
      s.monthlyDownloads = 0,
      s.updatedAt = timestamp()
```

---

## Redis Schema (Real-time Layer)

```
Redis Key Patterns:
├── dedup:{campaign_id}:{ip_hash}:{ua_hash}  (STRING, TTL 86400s)
│   └── IAB v2.2 24-hour dedup window
│
├── counter:campaign:{campaign_id}:impressions  (INCR counter)
│   └── Real-time impression counter for dashboard
│
├── counter:campaign:{campaign_id}:valid        (INCR counter)
│   └── Real-time valid impression counter
│
├── counter:show:{show_id}:downloads:daily:{date}  (INCR counter, TTL 172800s)
│   └── Daily download counter for real-time dashboard
│
├── cache:campaign:{campaign_id}:delivery       (HASH, TTL 300s)
│   └── Cached delivery summary for API responses
│
├── cache:network:{network_id}:dashboard        (JSON string, TTL 60s)
│   └── Cached network dashboard data
│
├── sorted:show:{show_id}:top_episodes:{period}  (SORTED SET)
│   └── Top episodes by downloads for a show
│
└── stream:impressions                           (STREAM)
    └── Redis Stream for real-time impression processing
```

```python
# IAB v2.2 Deduplication with Redis

import redis
import hashlib
from datetime import datetime

class IABDeduplicator:
    """
    Implements IAB Podcast Measurement v2.2 deduplication:
    - Unique combination of IP address + User Agent
    - 24-hour sliding window
    - IPv4 full address, IPv6 prefix /64 (first 64 bits)
    """
    
    DEDUP_WINDOW_SECONDS = 86400  # 24 hours
    
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def _normalize_ip(self, ip: str) -> str:
        """Truncate IPv6 to /64 prefix per IAB v2.2"""
        if ':' in ip:  # IPv6
            parts = ip.split(':')
            # Keep first 4 groups (64 bits)
            return ':'.join(parts[:4]) + '::'
        return ip  # IPv4 unchanged
    
    def _hash(self, value: str) -> str:
        return hashlib.sha256(value.encode()).hexdigest()
    
    def is_unique(self, campaign_id: str, ip: str, user_agent: str) -> bool:
        normalized_ip = self._normalize_ip(ip)
        ip_hash = self._hash(normalized_ip)
        ua_hash = self._hash(user_agent)
        
        key = f"dedup:{campaign_id}:{ip_hash}:{ua_hash}"
        
        # SET NX returns True only if key didn't exist
        is_new = self.redis.set(key, '1', nx=True, ex=self.DEDUP_WINDOW_SECONDS)
        return bool(is_new)
    
    def record_impression(self, campaign_id: str, show_id: str,
                          ip: str, user_agent: str) -> dict:
        is_valid = self.is_unique(campaign_id, ip, user_agent)
        
        if is_valid:
            # Increment real-time counters
            pipe = self.redis.pipeline()
            pipe.incr(f"counter:campaign:{campaign_id}:impressions")
            pipe.incr(f"counter:campaign:{campaign_id}:valid")
            pipe.incr(f"counter:show:{show_id}:downloads:daily:{datetime.utcnow().strftime('%Y-%m-%d')}")
            # Invalidate cached delivery summary
            pipe.delete(f"cache:campaign:{campaign_id}:delivery")
            pipe.execute()
        else:
            self.redis.incr(f"counter:campaign:{campaign_id}:impressions")
        
        return {'is_valid': is_valid, 'reason': '' if is_valid else 'iab_dedup'}
```

---

## Pros and Cons

### Pros

1. **Best-in-class performance for each workload**: ClickHouse delivers sub-second aggregation over billions of impression rows with 10-20x compression vs. PostgreSQL. Neo4j answers "which shows share audiences?" in milliseconds via native graph traversal that would require expensive self-joins in SQL. PostgreSQL provides ACID for financial transactions. Each engine does what it does best.

2. **ClickHouse compression for impression data**: At 50M impressions/month, PostgreSQL would use approximately 50-100 GB/year. ClickHouse's columnar compression (LZ4 + delta encoding on sorted columns) typically achieves 5-10x compression, reducing this to 5-10 GB/year. The `LowCardinality` type on dimension columns (country, device, app) further reduces storage by dictionary-encoding repeated values.

3. **ClickHouse materialized views for zero-cost aggregation**: The `mv_campaign_daily`, `mv_show_daily_downloads`, and `mv_geo_daily` materialized views are incrementally maintained on insert -- no background refresh job required. The aggregated data is always up-to-date without the `REFRESH MATERIALIZED VIEW CONCURRENTLY` latency of PostgreSQL.

4. **Neo4j for audience overlap and recommendation**: The "cross-show audience overlap for network packaging proposals" use case is a first-class graph traversal in Neo4j. The query "find listener cohorts that overlap between Show A and Show B" is a 2-hop traversal, expressed naturally in Cypher. The equivalent SQL query requires self-joins on a many-to-many table with aggregation, and becomes unwieldy for 3+ show combinations.

5. **Real-time IAB deduplication**: Redis provides O(1) lookup for the 24-hour dedup window, handling 1000+ impressions/second without impacting either PostgreSQL or ClickHouse write paths. The TTL-based expiry automatically cleans up dedup windows without scheduled jobs.

6. **Clean separation of concerns**: The operational service (PostgreSQL), analytics service (ClickHouse), and graph service (Neo4j) can scale independently. A spike in ad delivery doesn't impact campaign management or royalty calculations because they are served by different databases.

7. **ClickHouse native anomaly detection support**: The `mv_hourly_stats` materialized view with `downloads_per_ip_ratio` provides a built-in signal for bot traffic detection. ClickHouse's statistical functions (`quantile`, `stddevPop`, `arrayJoin`) enable sophisticated anomaly detection queries directly on the analytics engine.

### Cons

1. **Operational complexity**: Running PostgreSQL, ClickHouse, Neo4j, Redis, Kafka, and Debezium requires expertise across five distinct technologies. Each has its own backup strategy, monitoring requirements, upgrade path, and failure modes. A team of fewer than 3-4 engineers will struggle to maintain this stack.

2. **Data consistency across stores**: The CDC pipeline introduces eventual consistency between PostgreSQL (source of truth) and ClickHouse/Neo4j (derived stores). If the CDC pipeline fails, ClickHouse analytics and Neo4j graph data become stale. Unlike a single-database system where consistency is guaranteed by the database, cross-store consistency requires monitoring and alerting infrastructure.

3. **Transaction boundaries are limited**: A single business operation (e.g., "approve a royalty calculation and create payouts") can be transactional within PostgreSQL, but updating the corresponding analytics data in ClickHouse and graph data in Neo4j is asynchronous. If the CDC consumer crashes after PostgreSQL commits but before ClickHouse/Neo4j are updated, the system is temporarily inconsistent.

4. **Development complexity**: Developers must learn SQL (PostgreSQL), ClickHouse SQL (subtly different: no UPDATE, no DELETE of individual rows, different JOIN semantics), Cypher (Neo4j), and Redis commands. The cognitive load is substantially higher than a single-database approach.

5. **Infrastructure cost**: At small scale (under 50 shows), the overhead of running 5 services is not justified by the performance benefits. A single PostgreSQL instance with partitioning handles the workload adequately, and the polyglot approach adds cost without proportional benefit.

6. **Testing complexity**: Integration tests must verify behavior across all stores. A test for "campaign delivery reconciliation" must seed data in PostgreSQL and ClickHouse, run the reconciliation logic, and verify results in both stores. Test fixtures are significantly more complex than single-database tests.

7. **Vendor lock-in for ClickHouse and Neo4j**: While PostgreSQL is universally available, ClickHouse managed hosting options are more limited (ClickHouse Cloud, Altinity, or self-hosted). Neo4j's managed offering (Aura) has fewer cloud region options. Self-hosting these engines requires container orchestration expertise.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Operational DB | PostgreSQL 16+ (Neon or RDS) | ACID, pgvector, managed hosting |
| Analytics DB | ClickHouse (ClickHouse Cloud or Altinity) | Columnar, materialized views, compression |
| Graph DB | Neo4j 5+ (Aura or self-hosted) | Native graph traversal, Cypher query language |
| Cache/Dedup | Redis 7+ (Elasticache or Upstash) | TTL-based dedup, real-time counters |
| Message broker | Apache Kafka (Confluent Cloud) | CDC routing, impression ingestion |
| CDC | Debezium | PostgreSQL WAL-based capture |
| Orchestration | Kubernetes (EKS/GKE) | Multi-service deployment management |
| Monitoring | Grafana + Prometheus + OpenTelemetry | Unified observability across all stores |
| API framework | Node.js/Fastify or Go | Service-per-store pattern |

---

## Migration and Scaling Considerations

### Phased Adoption Strategy

The polyglot approach should not be adopted all at once. The recommended adoption sequence:

**Phase 1 (MVP): PostgreSQL only**
Start with the hybrid PostgreSQL model (Suggestion 3) for all data. This is sufficient for 0-100 shows and under 10M impressions/month.

**Phase 2 (Scale trigger: 10M+ impressions/month): Add ClickHouse**
When impression volume exceeds what PostgreSQL partitioning handles comfortably, introduce ClickHouse for impression and analytics data:
1. Deploy ClickHouse alongside PostgreSQL
2. Dual-write impressions to both PostgreSQL and ClickHouse for 30 days
3. Validate ClickHouse query results against PostgreSQL
4. Switch analytics queries to ClickHouse
5. Stop writing impressions to PostgreSQL (keep historical data for reconciliation)

**Phase 3 (Scale trigger: audience overlap/matching features): Add Neo4j**
When advertiser-show matching and audience overlap analysis become production features:
1. Deploy Neo4j
2. Seed graph from PostgreSQL data
3. Set up Debezium CDC for ongoing sync
4. Route graph queries (matching, overlap, packaging) to Neo4j
5. Keep PostgreSQL as the write source of truth

**Phase 4 (Scale trigger: real-time dashboard requirements): Add Redis layer**
When real-time dashboard requirements exceed PostgreSQL's ability to serve:
1. Add Redis counters for real-time metrics
2. Move IAB dedup from application-level to Redis
3. Cache frequently-accessed dashboard queries

### Scaling Each Component

| Component | Vertical scaling | Horizontal scaling |
|-----------|-----------------|-------------------|
| PostgreSQL | Upgrade instance size; add read replicas | Citus for distributed tables (if needed) |
| ClickHouse | Upgrade instance; increase shard count | ReplicatedMergeTree + Distributed tables |
| Neo4j | Increase memory for graph cache | Read replicas (Enterprise); fabric for multi-graph |
| Redis | Increase memory | Redis Cluster for data >25 GB |
| Kafka | Increase partition count | Add brokers for throughput |

### Data Retention Policy

| Data | PostgreSQL | ClickHouse | Neo4j | Redis |
|------|-----------|-----------|-------|-------|
| Impressions | 3 months (reconciliation reference) | 24 months (analytics) | N/A | 24 hours (dedup) |
| Episode downloads | N/A | 36 months | N/A | 48 hours (counters) |
| Aggregated metrics | N/A | Indefinite (materialized views) | N/A | 5 min (dashboard cache) |
| Graph relationships | N/A | N/A | Indefinite | N/A |
| Financial records | Indefinite | N/A | N/A | N/A |
| Audit log | 36 months | N/A | N/A | N/A |

### Disaster Recovery

Each store requires its own backup and recovery strategy:
- **PostgreSQL**: WAL archiving to S3 with point-in-time recovery. RPO: seconds. RTO: minutes.
- **ClickHouse**: Native backup to S3 (incremental). RPO: 1 hour. RTO: 30 minutes. Analytical data can be rebuilt from raw impression events if needed.
- **Neo4j**: Database dump or online backup. RPO: 1 hour. RTO: 15 minutes. Graph can be rebuilt from PostgreSQL source data via CDC replay.
- **Redis**: AOF persistence for dedup state. RTO: seconds. Loss of dedup state is acceptable (worst case: a few duplicate impressions counted in the 24-hour window after recovery).
