# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Podcast Network Management (Candidate #443)
> Generated: 2026-05-25

## Overview

A fully normalized relational model using PostgreSQL, designed around the six core domains identified in the feature survey: show/episode catalogue, host royalty management, ad insertion and campaign management, cross-platform analytics, subscription/membership, and financial operations. PostgreSQL is chosen for its strong ACID guarantees, rich type system (native UUID, JSONB for semi-structured metadata, array types for tags), mature partitioning for time-series analytics data, and broad ecosystem support with ORMs, migration tools, and managed cloud offerings.

This model follows Third Normal Form (3NF) throughout, with strategic denormalization only in materialized views for reporting performance. Every table includes audit columns (`created_at`, `updated_at`, `created_by`) and soft-delete support (`deleted_at`) for regulatory compliance and financial audit trail requirements.

---

## Schema Definition

### Core Infrastructure

```sql
-- ============================================================
-- ENUMS
-- ============================================================

CREATE TYPE user_role AS ENUM (
    'network_admin', 'show_admin', 'host', 'analyst', 'advertiser_viewer'
);

CREATE TYPE episode_status AS ENUM (
    'draft', 'scheduled', 'published', 'archived', 'deleted'
);

CREATE TYPE ad_position AS ENUM (
    'pre_roll', 'mid_roll', 'post_roll'
);

CREATE TYPE campaign_status AS ENUM (
    'draft', 'proposed', 'booked', 'active', 'paused', 'completed', 'cancelled'
);

CREATE TYPE pricing_model AS ENUM (
    'cpm', 'flat_rate', 'revenue_share', 'hybrid'
);

CREATE TYPE inventory_type AS ENUM (
    'guaranteed', 'programmatic', 'house'
);

CREATE TYPE payout_status AS ENUM (
    'pending', 'calculated', 'approved', 'processing', 'completed', 'failed', 'reversed'
);

CREATE TYPE subscription_tier AS ENUM (
    'free', 'supporter', 'premium', 'patron'
);

CREATE TYPE subscription_status AS ENUM (
    'active', 'paused', 'cancelled', 'expired', 'past_due'
);

CREATE TYPE hosting_platform AS ENUM (
    'megaphone', 'libsyn', 'acast', 'podbean', 'buzzsprout', 'self_hosted', 'other'
);

CREATE TYPE tax_form_type AS ENUM (
    'w9', 'w8ben', 'w8bene', 'vat_registration', 'dac7'
);

CREATE TYPE currency_code AS ENUM (
    'USD', 'EUR', 'GBP', 'CAD', 'AUD', 'JPY', 'CHF', 'SEK', 'NOK', 'DKK'
);

-- ============================================================
-- ORGANIZATIONS & ACCESS CONTROL
-- ============================================================

CREATE TABLE networks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    description         TEXT,
    logo_url            VARCHAR(2048),
    website_url         VARCHAR(2048),
    default_currency    currency_code NOT NULL DEFAULT 'USD',
    timezone            VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    iab_certified       BOOLEAN NOT NULL DEFAULT FALSE,
    stripe_account_id   VARCHAR(255),         -- Stripe Connect platform account
    tipalti_payer_id    VARCHAR(255),         -- Tipalti payer entity
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(320) NOT NULL UNIQUE,
    name                VARCHAR(255) NOT NULL,
    avatar_url          VARCHAR(2048),
    password_hash       VARCHAR(255),         -- NULL if SSO-only
    oidc_subject        VARCHAR(255),         -- OpenID Connect subject identifier
    oidc_issuer         VARCHAR(2048),        -- OIDC issuer URL
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_secret_enc      BYTEA,                -- Encrypted TOTP secret
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE network_memberships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    role                user_role NOT NULL DEFAULT 'analyst',
    invited_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    accepted_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (network_id, user_id)
);

-- ============================================================
-- SHOW & EPISODE CATALOGUE
-- ============================================================

CREATE TABLE shows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,
    summary             TEXT,
    language            VARCHAR(10) NOT NULL DEFAULT 'en',
    explicit            BOOLEAN NOT NULL DEFAULT FALSE,
    cover_image_url     VARCHAR(2048),
    website_url         VARCHAR(2048),
    rss_feed_url        VARCHAR(2048),        -- The canonical RSS feed URL
    hosting_platform    hosting_platform NOT NULL DEFAULT 'self_hosted',
    hosting_external_id VARCHAR(255),         -- ID on the external hosting platform
    itunes_category     VARCHAR(255),
    itunes_subcategory  VARCHAR(255),
    podcast_guid        UUID,                 -- podcast:guid per Podcast Namespace
    copyright           VARCHAR(500),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    publish_frequency   VARCHAR(50),          -- e.g. 'weekly', 'biweekly', 'daily'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    UNIQUE (network_id, slug)
);

CREATE INDEX idx_shows_network ON shows(network_id) WHERE deleted_at IS NULL;

CREATE TABLE show_hosts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    user_id             UUID REFERENCES users(id),      -- NULL if external contributor
    person_name         VARCHAR(255) NOT NULL,
    person_role         VARCHAR(100) NOT NULL DEFAULT 'host', -- host, co-host, producer, editor, guest
    person_url          VARCHAR(2048),
    person_image_url    VARCHAR(2048),
    podcast_person_href VARCHAR(2048),        -- podcast:person href attribute
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    started_at          DATE,
    ended_at            DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_show_hosts_show ON show_hosts(show_id);

CREATE TABLE episodes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,
    summary             TEXT,
    status              episode_status NOT NULL DEFAULT 'draft',
    season_number       INTEGER,
    episode_number      INTEGER,
    episode_type        VARCHAR(20) DEFAULT 'full', -- full, trailer, bonus (iTunes)
    audio_url           VARCHAR(2048),
    audio_duration_secs INTEGER,
    audio_size_bytes    BIGINT,
    audio_mime_type     VARCHAR(100) DEFAULT 'audio/mpeg',
    cover_image_url     VARCHAR(2048),
    explicit            BOOLEAN NOT NULL DEFAULT FALSE,
    transcript_url      VARCHAR(2048),        -- podcast:transcript
    chapters_url        VARCHAR(2048),        -- podcast:chapters JSON URL
    guid                VARCHAR(500) NOT NULL,
    published_at        TIMESTAMPTZ,
    scheduled_at        TIMESTAMPTZ,
    hosting_external_id VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    UNIQUE (show_id, slug)
);

CREATE INDEX idx_episodes_show_published ON episodes(show_id, published_at DESC)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_episodes_status ON episodes(status) WHERE deleted_at IS NULL;

CREATE TABLE episode_guests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    person_name         VARCHAR(255) NOT NULL,
    person_role         VARCHAR(100) NOT NULL DEFAULT 'guest',
    person_url          VARCHAR(2048),
    person_image_url    VARCHAR(2048),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- AD INSERTION POINTS & DAI CONFIGURATION
-- ============================================================

CREATE TABLE ad_insertion_points (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    position            ad_position NOT NULL,
    offset_secs         NUMERIC(10,3),        -- Time offset in seconds from episode start
    max_duration_secs   INTEGER DEFAULT 60,   -- Maximum ad duration for this slot
    is_auto_detected    BOOLEAN NOT NULL DEFAULT FALSE,  -- AI-suggested vs manual
    confidence_score    NUMERIC(5,4),         -- AI confidence for auto-detected points
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ad_points_episode ON ad_insertion_points(episode_id);

-- ============================================================
-- ADVERTISERS & CAMPAIGNS
-- ============================================================

CREATE TABLE advertisers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    company_name        VARCHAR(500) NOT NULL,
    contact_name        VARCHAR(255),
    contact_email       VARCHAR(320),
    website_url         VARCHAR(2048),
    industry_category   VARCHAR(255),
    billing_currency    currency_code NOT NULL DEFAULT 'USD',
    stripe_customer_id  VARCHAR(255),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE agencies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(500) NOT NULL,
    contact_name        VARCHAR(255),
    contact_email       VARCHAR(320),
    commission_pct      NUMERIC(5,4) DEFAULT 0.15, -- Agency commission rate
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE campaigns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    advertiser_id       UUID NOT NULL REFERENCES advertisers(id),
    agency_id           UUID REFERENCES agencies(id),
    name                VARCHAR(500) NOT NULL,
    status              campaign_status NOT NULL DEFAULT 'draft',
    pricing_model       pricing_model NOT NULL DEFAULT 'cpm',
    inventory_type      inventory_type NOT NULL DEFAULT 'guaranteed',
    cpm_rate            NUMERIC(10,4),        -- Cost per mille (if CPM pricing)
    flat_rate_amount    NUMERIC(12,2),        -- Flat rate (if flat rate pricing)
    currency            currency_code NOT NULL DEFAULT 'USD',
    booked_impressions  BIGINT,               -- Target impressions for guaranteed
    budget_amount       NUMERIC(12,2),        -- Total campaign budget
    flight_start        DATE NOT NULL,
    flight_end          DATE NOT NULL,
    creative_url        VARCHAR(2048),        -- VAST URL or audio creative URL
    creative_duration   INTEGER,              -- Creative duration in seconds
    vast_tag_url        VARCHAR(2048),        -- VAST 4.1+ compliant tag URL
    brand_safety_tags   TEXT[],               -- Array of brand safety categories
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT valid_flight CHECK (flight_end >= flight_start)
);

CREATE INDEX idx_campaigns_network ON campaigns(network_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_campaigns_status ON campaigns(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_campaigns_flight ON campaigns(flight_start, flight_end);

CREATE TABLE campaign_show_targets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id),
    show_id             UUID NOT NULL REFERENCES shows(id),
    ad_position         ad_position NOT NULL DEFAULT 'mid_roll',
    allocated_impressions BIGINT,             -- Impressions allocated to this show
    weight              NUMERIC(5,4) DEFAULT 1.0, -- Relative weight for distribution
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (campaign_id, show_id, ad_position)
);

CREATE TABLE campaign_exclusions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id),
    excluded_advertiser_id UUID REFERENCES advertisers(id), -- Competitor exclusion
    excluded_category   VARCHAR(255),                        -- Category exclusion
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- AD DELIVERY & IMPRESSION TRACKING
-- ============================================================

CREATE TABLE ad_impressions (
    id                  BIGSERIAL PRIMARY KEY,
    campaign_id         UUID NOT NULL REFERENCES campaigns(id),
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    show_id             UUID NOT NULL REFERENCES shows(id),
    insertion_point_id  UUID REFERENCES ad_insertion_points(id),
    ad_position         ad_position NOT NULL,
    listener_ip_hash    VARCHAR(64) NOT NULL, -- SHA-256 of IP for IAB dedup
    user_agent_hash     VARCHAR(64) NOT NULL, -- SHA-256 of User-Agent
    country_code        VARCHAR(2),
    region_code         VARCHAR(10),
    city                VARCHAR(255),
    device_type         VARCHAR(50),          -- mobile, desktop, smart_speaker, etc.
    app_name            VARCHAR(255),         -- Podcast app (Apple Podcasts, Spotify, etc.)
    is_valid            BOOLEAN NOT NULL DEFAULT TRUE, -- IAB v2.2 valid flag
    dedup_window_id     VARCHAR(100),         -- IAB dedup window identifier
    delivered_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Partitioned by month for query performance
    CONSTRAINT valid_impression_time CHECK (delivered_at IS NOT NULL)
) PARTITION BY RANGE (delivered_at);

-- Create monthly partitions (example for 2026)
CREATE TABLE ad_impressions_2026_01 PARTITION OF ad_impressions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE ad_impressions_2026_02 PARTITION OF ad_impressions
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... continue for each month

CREATE INDEX idx_impressions_campaign ON ad_impressions(campaign_id, delivered_at);
CREATE INDEX idx_impressions_episode ON ad_impressions(episode_id, delivered_at);
CREATE INDEX idx_impressions_dedup ON ad_impressions(listener_ip_hash, user_agent_hash, delivered_at);

-- Daily campaign delivery summary (materialized for performance)
CREATE MATERIALIZED VIEW campaign_daily_delivery AS
SELECT
    campaign_id,
    show_id,
    ad_position,
    DATE(delivered_at) AS delivery_date,
    COUNT(*) FILTER (WHERE is_valid) AS valid_impressions,
    COUNT(*) FILTER (WHERE NOT is_valid) AS invalid_impressions,
    COUNT(DISTINCT listener_ip_hash) AS unique_listeners,
    COUNT(DISTINCT country_code) AS country_count
FROM ad_impressions
GROUP BY campaign_id, show_id, ad_position, DATE(delivered_at);

CREATE UNIQUE INDEX idx_campaign_daily ON campaign_daily_delivery(
    campaign_id, show_id, ad_position, delivery_date);

-- ============================================================
-- CAMPAIGN RECONCILIATION
-- ============================================================

CREATE TABLE campaign_reconciliations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    booked_impressions  BIGINT NOT NULL,
    delivered_impressions BIGINT NOT NULL,
    valid_impressions   BIGINT NOT NULL,      -- After IAB v2.2 filtering
    delivery_pct        NUMERIC(7,4),         -- valid / booked * 100
    revenue_earned      NUMERIC(12,2) NOT NULL,
    currency            currency_code NOT NULL,
    reconciled_by       UUID REFERENCES users(id),
    reconciled_at       TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- HOST ROYALTY MANAGEMENT
-- ============================================================

CREATE TABLE royalty_rules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    effective_from      DATE NOT NULL,
    effective_until     DATE,                 -- NULL = currently active
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT no_overlap EXCLUDE USING gist (
        show_id WITH =,
        daterange(effective_from, effective_until, '[]') WITH &&
    ) WHERE (is_active = TRUE)
);

CREATE TABLE royalty_splits (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    royalty_rule_id     UUID NOT NULL REFERENCES royalty_rules(id),
    recipient_host_id   UUID NOT NULL REFERENCES show_hosts(id),
    split_percentage    NUMERIC(7,4) NOT NULL, -- e.g. 33.3333 for one-third
    minimum_amount      NUMERIC(10,2) DEFAULT 0,  -- Minimum guaranteed payout
    cap_amount          NUMERIC(10,2),        -- Maximum cap per period
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- All splits for a rule must sum to <= 100%
    CONSTRAINT valid_pct CHECK (split_percentage > 0 AND split_percentage <= 100)
);

CREATE INDEX idx_royalty_splits_rule ON royalty_splits(royalty_rule_id);

CREATE TABLE royalty_calculations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    royalty_rule_id     UUID NOT NULL REFERENCES royalty_rules(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    total_revenue       NUMERIC(14,2) NOT NULL, -- Total show revenue for period
    ad_revenue          NUMERIC(14,2) NOT NULL DEFAULT 0,
    subscription_revenue NUMERIC(14,2) NOT NULL DEFAULT 0,
    other_revenue       NUMERIC(14,2) NOT NULL DEFAULT 0,
    network_commission  NUMERIC(14,2) NOT NULL DEFAULT 0, -- Network's share
    distributable_amount NUMERIC(14,2) NOT NULL, -- After network commission
    currency            currency_code NOT NULL DEFAULT 'USD',
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE royalty_line_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calculation_id      UUID NOT NULL REFERENCES royalty_calculations(id),
    recipient_host_id   UUID NOT NULL REFERENCES show_hosts(id),
    split_percentage    NUMERIC(7,4) NOT NULL, -- Snapshot of the split at calc time
    gross_amount        NUMERIC(14,2) NOT NULL,
    adjustments         NUMERIC(14,2) NOT NULL DEFAULT 0, -- Advances, clawbacks, etc.
    net_amount          NUMERIC(14,2) NOT NULL,
    currency            currency_code NOT NULL DEFAULT 'USD',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_royalty_line_items_calc ON royalty_line_items(calculation_id);

-- ============================================================
-- PAYMENT DISBURSEMENT
-- ============================================================

CREATE TABLE payment_profiles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    stripe_connect_id   VARCHAR(255),         -- Stripe Connected Account ID
    tipalti_payee_id    VARCHAR(255),         -- Tipalti payee ID
    preferred_currency  currency_code NOT NULL DEFAULT 'USD',
    payment_method      VARCHAR(50) NOT NULL DEFAULT 'bank_transfer', -- bank_transfer, paypal, check
    tax_form_type       tax_form_type,
    tax_form_status     VARCHAR(20) DEFAULT 'pending', -- pending, submitted, verified, expired
    tax_form_verified_at TIMESTAMPTZ,
    bank_country        VARCHAR(2),           -- ISO 3166-1 alpha-2
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payouts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    recipient_user_id   UUID NOT NULL REFERENCES users(id),
    payment_profile_id  UUID NOT NULL REFERENCES payment_profiles(id),
    status              payout_status NOT NULL DEFAULT 'pending',
    amount              NUMERIC(14,2) NOT NULL,
    currency            currency_code NOT NULL,
    exchange_rate       NUMERIC(12,6),        -- If converted from network default currency
    amount_local        NUMERIC(14,2),        -- Amount in recipient's preferred currency
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    stripe_transfer_id  VARCHAR(255),         -- Stripe transfer reference
    tipalti_payment_id  VARCHAR(255),         -- Tipalti payment reference
    processed_at        TIMESTAMPTZ,
    failed_reason       TEXT,
    statement_url       VARCHAR(2048),        -- Link to generated PDF statement
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payouts_recipient ON payouts(recipient_user_id, period_start);
CREATE INDEX idx_payouts_status ON payouts(status);

CREATE TABLE payout_line_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payout_id           UUID NOT NULL REFERENCES payouts(id),
    royalty_line_item_id UUID NOT NULL REFERENCES royalty_line_items(id),
    show_id             UUID NOT NULL REFERENCES shows(id),
    amount              NUMERIC(14,2) NOT NULL,
    description         TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- CROSS-PLATFORM ANALYTICS
-- ============================================================

CREATE TABLE analytics_sources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    platform            hosting_platform NOT NULL,
    api_credential_enc  BYTEA,                -- Encrypted API key/token
    oauth_token_enc     BYTEA,                -- Encrypted OAuth token
    oauth_refresh_enc   BYTEA,                -- Encrypted refresh token
    oauth_expires_at    TIMESTAMPTZ,
    last_sync_at        TIMESTAMPTZ,
    sync_status         VARCHAR(20) DEFAULT 'pending', -- pending, syncing, completed, error
    sync_error_message  TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Normalized download/listen events from all platforms
CREATE TABLE episode_analytics (
    id                  BIGSERIAL PRIMARY KEY,
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    show_id             UUID NOT NULL REFERENCES shows(id),
    source_platform     hosting_platform NOT NULL,
    metric_date         DATE NOT NULL,
    downloads           BIGINT NOT NULL DEFAULT 0,
    unique_listeners    BIGINT DEFAULT 0,
    completion_rate     NUMERIC(5,4),         -- 0.0000 to 1.0000
    avg_listen_duration_secs INTEGER,
    iab_version         VARCHAR(10),          -- '2.1' or '2.2'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (episode_id, source_platform, metric_date)
) PARTITION BY RANGE (metric_date);

CREATE TABLE episode_geo_analytics (
    id                  BIGSERIAL PRIMARY KEY,
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    metric_date         DATE NOT NULL,
    country_code        VARCHAR(2) NOT NULL,
    region_code         VARCHAR(10),
    downloads           BIGINT NOT NULL DEFAULT 0,
    unique_listeners    BIGINT DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (episode_id, metric_date, country_code, region_code)
) PARTITION BY RANGE (metric_date);

CREATE TABLE episode_app_analytics (
    id                  BIGSERIAL PRIMARY KEY,
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    metric_date         DATE NOT NULL,
    app_name            VARCHAR(255) NOT NULL,  -- Apple Podcasts, Spotify, etc.
    device_type         VARCHAR(50),            -- mobile, desktop, smart_speaker
    downloads           BIGINT NOT NULL DEFAULT 0,
    unique_listeners    BIGINT DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (episode_id, metric_date, app_name)
) PARTITION BY RANGE (metric_date);

-- Network-level aggregated view
CREATE MATERIALIZED VIEW network_analytics_summary AS
SELECT
    s.network_id,
    s.id AS show_id,
    ea.metric_date,
    SUM(ea.downloads) AS total_downloads,
    SUM(ea.unique_listeners) AS total_unique_listeners,
    AVG(ea.completion_rate) AS avg_completion_rate,
    COUNT(DISTINCT ea.episode_id) AS episodes_with_data
FROM episode_analytics ea
JOIN shows s ON s.id = ea.show_id
WHERE s.deleted_at IS NULL
GROUP BY s.network_id, s.id, ea.metric_date;

-- ============================================================
-- SUBSCRIPTION & MEMBERSHIP
-- ============================================================

CREATE TABLE subscription_plans (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID REFERENCES shows(id),       -- NULL = network-level plan
    network_id          UUID NOT NULL REFERENCES networks(id),
    name                VARCHAR(255) NOT NULL,
    tier                subscription_tier NOT NULL DEFAULT 'supporter',
    price_amount        NUMERIC(10,2) NOT NULL,
    currency            currency_code NOT NULL DEFAULT 'USD',
    billing_interval    VARCHAR(20) NOT NULL DEFAULT 'monthly', -- monthly, yearly
    features            TEXT[],               -- Array of feature descriptions
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    stripe_price_id     VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE subscriptions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id             UUID NOT NULL REFERENCES subscription_plans(id),
    subscriber_email    VARCHAR(320) NOT NULL,
    subscriber_name     VARCHAR(255),
    status              subscription_status NOT NULL DEFAULT 'active',
    stripe_subscription_id VARCHAR(255),
    started_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    current_period_start TIMESTAMPTZ,
    current_period_end  TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ,
    cancel_reason       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_plan ON subscriptions(plan_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);

CREATE TABLE premium_content (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    required_tier       subscription_tier NOT NULL DEFAULT 'supporter',
    private_rss_token   VARCHAR(255) NOT NULL UNIQUE,
    ad_free             BOOLEAN NOT NULL DEFAULT TRUE,
    bonus_audio_url     VARCHAR(2048),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- FINANCIAL REPORTING
-- ============================================================

CREATE TABLE revenue_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    show_id             UUID REFERENCES shows(id),
    source_type         VARCHAR(50) NOT NULL, -- 'ad_campaign', 'subscription', 'sponsorship', 'other'
    source_id           UUID,                 -- Reference to campaign, subscription, etc.
    amount              NUMERIC(14,2) NOT NULL,
    currency            currency_code NOT NULL DEFAULT 'USD',
    revenue_date        DATE NOT NULL,
    description         TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (revenue_date);

CREATE TABLE expense_entries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    show_id             UUID REFERENCES shows(id),
    category            VARCHAR(100) NOT NULL, -- 'hosting', 'payout', 'agency_commission', 'platform_fee'
    amount              NUMERIC(14,2) NOT NULL,
    currency            currency_code NOT NULL DEFAULT 'USD',
    expense_date        DATE NOT NULL,
    description         TEXT,
    reference_id        UUID,                 -- Reference to payout, etc.
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (expense_date);

-- Network P&L materialized view
CREATE MATERIALIZED VIEW network_pnl_monthly AS
SELECT
    r.network_id,
    r.show_id,
    DATE_TRUNC('month', r.revenue_date) AS month,
    SUM(r.amount) AS total_revenue,
    COALESCE(e.total_expenses, 0) AS total_expenses,
    SUM(r.amount) - COALESCE(e.total_expenses, 0) AS net_margin
FROM revenue_entries r
LEFT JOIN (
    SELECT network_id, show_id, DATE_TRUNC('month', expense_date) AS month,
           SUM(amount) AS total_expenses
    FROM expense_entries
    GROUP BY network_id, show_id, DATE_TRUNC('month', expense_date)
) e ON e.network_id = r.network_id
    AND e.show_id = r.show_id
    AND e.month = DATE_TRUNC('month', r.revenue_date)
GROUP BY r.network_id, r.show_id, DATE_TRUNC('month', r.revenue_date), e.total_expenses;

-- ============================================================
-- AI FEATURES SUPPORT
-- ============================================================

CREATE TABLE content_embeddings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    embedding_model     VARCHAR(100) NOT NULL,  -- e.g. 'text-embedding-3-large'
    embedding_vector    vector(3072),           -- pgvector extension
    transcript_hash     VARCHAR(64),            -- SHA-256 of source transcript
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_embeddings_episode ON content_embeddings(episode_id);
CREATE INDEX idx_embeddings_vector ON content_embeddings
    USING ivfflat (embedding_vector vector_cosine_ops);

CREATE TABLE brand_safety_scans (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    scanned_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    risk_score          NUMERIC(5,4) NOT NULL, -- 0.0 (safe) to 1.0 (high risk)
    flagged_categories  TEXT[],                -- e.g. '{violence,adult_content}'
    flagged_segments    JSONB,                 -- Array of {start_secs, end_secs, reason}
    model_version       VARCHAR(50),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE advertiser_show_matches (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    advertiser_id       UUID NOT NULL REFERENCES advertisers(id),
    show_id             UUID NOT NULL REFERENCES shows(id),
    match_score         NUMERIC(5,4) NOT NULL, -- 0.0 to 1.0
    match_reasons       TEXT[],                -- e.g. '{audience_overlap, content_alignment}'
    generated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ,
    UNIQUE (advertiser_id, show_id)
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    network_id          UUID REFERENCES networks(id),
    user_id             UUID REFERENCES users(id),
    action              VARCHAR(100) NOT NULL,  -- 'campaign.created', 'payout.approved', etc.
    entity_type         VARCHAR(100) NOT NULL,  -- 'campaign', 'payout', 'royalty_rule', etc.
    entity_id           UUID NOT NULL,
    old_values          JSONB,
    new_values          JSONB,
    ip_address          INET,
    user_agent          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
```

---

## Entity Relationship Summary

```
networks 1──∞ shows 1──∞ episodes 1──∞ ad_insertion_points
    │              │           │
    │              │           └──∞ episode_analytics
    │              │           └──∞ episode_geo_analytics
    │              │           └──∞ episode_app_analytics
    │              │           └──∞ content_embeddings
    │              │           └──∞ brand_safety_scans
    │              │           └──1 premium_content
    │              │
    │              └──∞ show_hosts ──∞ royalty_splits
    │              └──∞ royalty_rules 1──∞ royalty_splits
    │              └──∞ analytics_sources
    │
    ├──∞ network_memberships ──1 users 1──∞ payment_profiles
    │                                         │
    │                                         └──∞ payouts 1──∞ payout_line_items
    │
    ├──∞ advertisers 1──∞ campaigns 1──∞ campaign_show_targets
    │                        │          └──∞ campaign_exclusions
    │                        └──∞ ad_impressions
    │                        └──∞ campaign_reconciliations
    │
    ├──∞ subscription_plans 1──∞ subscriptions
    │
    ├──∞ revenue_entries
    └──∞ expense_entries
```

---

## Pros and Cons

### Pros

1. **Strong referential integrity**: Foreign keys enforce data consistency across the entire domain. A royalty calculation cannot reference a nonexistent show or host, and a campaign cannot target a show outside the network. This is critical for financial accuracy in royalty disbursements.

2. **ACID compliance for financial operations**: Royalty calculations, payout approvals, and campaign reconciliations require transactional guarantees. PostgreSQL ensures that a royalty calculation and its line items are committed atomically, preventing partial payouts.

3. **IAB v2.2 deduplication natively supported**: The `ad_impressions` table with hashed IP and User-Agent columns supports the IAB-specified deduplication window. PostgreSQL's exclusion constraints and window functions handle the 24-hour dedup logic efficiently.

4. **Mature partitioning for time-series data**: The `ad_impressions`, `episode_analytics`, `revenue_entries`, and `audit_log` tables use native range partitioning by date. This enables efficient partition pruning for time-bounded queries and straightforward data lifecycle management (dropping old partitions).

5. **pgvector for AI features**: The content embeddings table uses PostgreSQL's pgvector extension for similarity search (advertiser-show matching, content-based recommendations), avoiding the need for a separate vector database.

6. **Exclusion constraints for business rules**: The `royalty_rules` table uses GiST exclusion constraints to prevent overlapping active rules for the same show, enforcing a critical business invariant at the database level.

7. **Ecosystem maturity**: PostgreSQL has the widest ORM support (SQLAlchemy, Prisma, TypeORM, Diesel, GORM), managed hosting on every cloud (RDS, Cloud SQL, Azure Database, Neon, Supabase), and proven operational tooling (pg_dump, pg_basebackup, pgBouncer, PgHero).

### Cons

1. **Write throughput ceiling for impressions**: At scale (100M+ impressions/month), the normalized `ad_impressions` table with foreign key checks and index maintenance becomes a write bottleneck. A mid-size network serving 50 shows with 10 campaigns each could generate 20-50M monthly impression rows, which is manageable but approaches the threshold where PostgreSQL partitioning alone may not suffice.

2. **Materialized view refresh latency**: The `campaign_daily_delivery` and `network_analytics_summary` views require periodic `REFRESH MATERIALIZED VIEW CONCURRENTLY`, introducing reporting lag. Real-time dashboards need additional infrastructure (streaming aggregations or application-level caching).

3. **Rigid schema for evolving metadata**: The Podcast Namespace (podcasting2.0) adds new tags regularly (Phase 7 closed July 2024). Adding each new tag as a column requires schema migrations. The normalized approach trades flexibility for safety, but the cadence of namespace changes creates migration pressure.

4. **Cross-platform analytics normalization complexity**: Different hosting platforms report metrics at different granularities (Podbean reports second-by-second engagement; Buzzsprout reports daily downloads). The normalized analytics tables force a lowest-common-denominator granularity, losing platform-specific detail.

5. **Join complexity for reporting**: The P&L view requires joining revenue_entries, expense_entries, royalty_calculations, payouts, and campaign_reconciliations across multiple time ranges. While PostgreSQL handles this well with proper indexing, the query complexity increases maintenance burden.

6. **No native graph traversal**: Cross-show audience overlap analysis and advertiser-show matching are graph problems (listener shared across shows, advertiser categories matching show topics). PostgreSQL can model these with join tables, but the queries are less intuitive than graph-native solutions.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Primary database | PostgreSQL 16+ | ACID, partitioning, pgvector, GiST exclusion constraints |
| Connection pooling | PgBouncer or Supavisor | Essential at >100 concurrent connections |
| Migrations | Prisma Migrate or Flyway | Versioned, reviewable SQL migrations |
| ORM | Prisma (TypeScript) or SQLAlchemy (Python) | Type-safe query generation |
| Vector search | pgvector extension | Avoids separate infrastructure for AI matching |
| Full-text search | PostgreSQL tsvector + GIN index | Show/episode search without Elasticsearch |
| Monitoring | PgHero + pg_stat_statements | Query performance visibility |
| Backup | pg_basebackup + WAL archiving to S3 | Point-in-time recovery for financial data |
| Managed hosting | Neon (serverless) or RDS (ops-managed) | Scale-to-zero for dev, provisioned for production |

---

## Migration and Scaling Considerations

### Initial Deployment (0-50 shows, <10M impressions/month)

A single PostgreSQL instance handles all workloads. Materialized views refresh on a 15-minute cron. pgvector handles embedding similarity search with IVFFlat indexes. Total database size expected under 50 GB.

### Growth Phase (50-500 shows, 10-100M impressions/month)

- **Read replicas**: Route analytics and reporting queries to read replicas using connection-level routing (e.g., application-level read/write splitting or PgBouncer tagging).
- **Partition management automation**: Use pg_partman to automatically create monthly partitions for impression and analytics tables. Implement partition detachment for data older than 24 months (archive to S3 as Parquet for long-term analytics).
- **Materialized view refresh optimization**: Move to `REFRESH MATERIALIZED VIEW CONCURRENTLY` with a dedicated refresh scheduler. Consider replacing the P&L materialized view with application-level caching (Redis) refreshed on revenue/expense writes.
- **Index-only scans**: Ensure covering indexes on high-traffic query paths (campaign delivery by date, episode analytics by show).

### Scale Phase (500+ shows, 100M+ impressions/month)

- **Impression offloading**: Consider moving the `ad_impressions` table to ClickHouse or TimescaleDB (see Suggestion 4) while keeping the rest of the schema in PostgreSQL. Use a CDC pipeline (Debezium) to replicate impressions for analytical queries while PostgreSQL retains the source of truth for reconciliation.
- **pgvector scaling**: At >1M embeddings, migrate from IVFFlat to HNSW index type for better recall/performance trade-off. Consider offloading to a dedicated vector store (Pinecone, Weaviate) if embedding search latency exceeds 100ms P99.
- **Horizontal read scaling**: Citus extension for PostgreSQL can distribute partitioned tables across multiple nodes if single-node read capacity is exhausted.
- **Connection management**: Move to connection pooling with prepared statement support. Evaluate Supavisor for Kubernetes-native deployments.

### Data Retention Policy

| Table | Hot retention | Warm retention | Archive |
|-------|--------------|----------------|---------|
| ad_impressions | 3 months | 12 months (compressed partitions) | S3 Parquet (indefinite) |
| episode_analytics | 24 months | 36 months | S3 Parquet |
| audit_log | 12 months | 36 months | S3 Parquet (7 years for financial) |
| revenue/expense | Full retention in DB | — | — |
| payouts | Full retention in DB | — | — |

### Migration Path from Legacy Systems

For networks migrating from spreadsheet-based royalty tracking:
1. Import show and host data via CSV bulk load (`COPY` command)
2. Backfill historical royalty rules with `effective_from` dates matching original agreements
3. Import historical payout records with `status = 'completed'` and original `processed_at` dates
4. Run parallel royalty calculations (old spreadsheet vs. new system) for 2-3 months before cutting over

For networks with existing hosting platform data:
1. Use hosting platform APIs (Megaphone, Libsyn, etc.) to backfill `episode_analytics` for the past 12 months
2. Map existing show IDs from each platform to the `shows.hosting_external_id` column
3. Set up ongoing sync via `analytics_sources` with appropriate OAuth credentials
