# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

> Project: Podcast Network Management (Candidate #443)
> Generated: 2026-05-25

## Overview

A hybrid model that combines normalized relational tables for stable, frequently-joined, financially-critical data with JSONB columns for flexible, schema-evolving, or platform-specific data. This approach exploits PostgreSQL's native JSONB capabilities -- GIN indexing, containment operators, JSON path queries, and partial indexing on JSONB fields -- to avoid the rigidity of a fully normalized schema while retaining ACID guarantees and referential integrity where they matter most.

The key insight for podcast network management is that the domain has two distinct data character profiles:

1. **Stable, relational data**: Network structure, show-host relationships, royalty rules, campaign contracts, payout records, and financial ledger entries. These have well-defined schemas, require referential integrity, and are queried via joins. They belong in normalized tables.

2. **Flexible, evolving data**: Episode metadata (Podcast Namespace tags that change with each phase), hosting platform-specific analytics payloads (Podbean's second-by-second engagement vs. Buzzsprout's daily downloads), ad creative specifications (VAST parameters, OpenRTB bid request fields), and AI-generated annotations (brand safety details, content embeddings metadata). These change frequently, vary by source, and are queried via containment or path extraction. They belong in JSONB columns.

This hybrid approach is particularly natural for a platform that must normalize data from 5+ hosting APIs, each with different response schemas, while maintaining a stable financial and operational core.

---

## Schema Definition

### Core Relational Tables (Normalized)

```sql
-- ============================================================
-- ENUMS (shared with relational core)
-- ============================================================

CREATE TYPE user_role AS ENUM (
    'network_admin', 'show_admin', 'host', 'analyst', 'advertiser_viewer'
);

CREATE TYPE campaign_status AS ENUM (
    'draft', 'proposed', 'booked', 'active', 'paused', 'completed', 'cancelled'
);

CREATE TYPE payout_status AS ENUM (
    'pending', 'calculated', 'approved', 'processing', 'completed', 'failed', 'reversed'
);

CREATE TYPE hosting_platform AS ENUM (
    'megaphone', 'libsyn', 'acast', 'podbean', 'buzzsprout', 'self_hosted', 'other'
);

-- ============================================================
-- ORGANIZATIONS & ACCESS CONTROL (Fully Normalized)
-- ============================================================

CREATE TABLE networks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    description         TEXT,
    logo_url            VARCHAR(2048),
    website_url         VARCHAR(2048),
    default_currency    VARCHAR(3) NOT NULL DEFAULT 'USD',
    timezone            VARCHAR(50) NOT NULL DEFAULT 'America/New_York',

    -- Flexible network-level settings stored as JSONB
    settings            JSONB NOT NULL DEFAULT '{}',
    /*
    settings schema:
    {
        "iab_certified": true,
        "iab_version": "2.2",
        "stripe_account_id": "acct_xxx",
        "tipalti_payer_id": "pay_xxx",
        "network_commission_default_pct": 20.0,
        "branding": {
            "primary_color": "#1a1a2e",
            "secondary_color": "#16213e",
            "font_family": "Inter"
        },
        "notification_preferences": {
            "payout_email": true,
            "campaign_alerts": true,
            "anomaly_alerts": true
        },
        "feature_flags": {
            "ai_ad_placement": true,
            "programmatic_enabled": false,
            "subscription_module": true
        }
    }
    */

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
    /*
    auth_metadata schema:
    {
        "oidc_subject": "sub_xxx",
        "oidc_issuer": "https://accounts.google.com",
        "mfa_enabled": true,
        "mfa_method": "totp",
        "last_login_at": "2026-05-25T10:00:00Z",
        "last_login_ip": "192.168.1.1",
        "login_count": 42,
        "failed_attempts": 0
    }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE network_memberships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    role                user_role NOT NULL DEFAULT 'analyst',
    permissions         JSONB NOT NULL DEFAULT '{}',
    /*
    permissions schema (role-based with per-resource overrides):
    {
        "shows": ["read", "write"],
        "campaigns": ["read"],
        "royalties": ["read", "approve"],
        "payouts": [],
        "show_overrides": {
            "show-uuid-1": ["read", "write", "delete"],
            "show-uuid-2": ["read"]
        }
    }
    */
    invited_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    accepted_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (network_id, user_id)
);

-- ============================================================
-- SHOW & EPISODE CATALOGUE (Relational core + JSONB metadata)
-- ============================================================

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
    hosting_platform    hosting_platform NOT NULL DEFAULT 'self_hosted',
    hosting_external_id VARCHAR(255),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- Podcast Namespace and iTunes metadata as JSONB
    -- This avoids schema migrations when new namespace tags are added
    podcast_metadata    JSONB NOT NULL DEFAULT '{}',
    /*
    podcast_metadata schema:
    {
        "itunes": {
            "category": "Technology",
            "subcategory": "Software How-To",
            "type": "episodic",
            "author": "Network Name",
            "owner_name": "John Doe",
            "owner_email": "john@example.com",
            "new_feed_url": null
        },
        "podcast_namespace": {
            "guid": "a1b2c3d4-e5f6-...",
            "medium": "podcast",
            "locked": true,
            "locked_owner": "john@example.com",
            "funding": [
                {"url": "https://patreon.com/show", "text": "Support on Patreon"}
            ],
            "value": {
                "type": "lightning",
                "method": "keysend",
                "recipients": [
                    {"name": "Host 1", "type": "wallet", "address": "xxx", "split": 60},
                    {"name": "Host 2", "type": "wallet", "address": "yyy", "split": 40}
                ]
            },
            "podroll": [
                {"feed_guid": "other-show-guid", "feed_url": "https://..."}
            ],
            "txt": [
                {"purpose": "verify", "value": "podcastindex-xxx"}
            ]
        },
        "social_links": {
            "twitter": "@showhandle",
            "instagram": "showhandle",
            "youtube": "UCxxx",
            "website": "https://show.example.com"
        },
        "tags": ["technology", "software", "interviews"],
        "publish_frequency": "weekly",
        "publish_day": "tuesday"
    }
    */

    -- Platform-specific hosting configuration
    hosting_config      JSONB NOT NULL DEFAULT '{}',
    /*
    hosting_config schema (varies by platform):
    
    For Megaphone:
    {
        "megaphone_network_id": "net_xxx",
        "megaphone_podcast_id": "pod_xxx",
        "api_base_url": "https://cms.megaphone.fm/api",
        "metrics_export_enabled": true,
        "dai_enabled": true
    }
    
    For Libsyn:
    {
        "libsyn_show_id": 12345,
        "api_slug": "myshow",
        "plan_tier": "advanced",
        "storage_quota_mb": 50000
    }
    
    For Podbean:
    {
        "podbean_podcast_id": "xxx",
        "podads_enabled": true,
        "engagement_intel_enabled": true
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    UNIQUE (network_id, slug)
);

-- GIN index on podcast_metadata for containment queries
CREATE INDEX idx_shows_podcast_metadata ON shows USING GIN (podcast_metadata);
CREATE INDEX idx_shows_network ON shows(network_id) WHERE deleted_at IS NULL;
-- Partial index for active shows with specific categories
CREATE INDEX idx_shows_category ON shows((podcast_metadata->'itunes'->>'category'))
    WHERE deleted_at IS NULL AND is_active = TRUE;

CREATE TABLE show_hosts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    user_id             UUID REFERENCES users(id),
    person_name         VARCHAR(255) NOT NULL,
    person_role         VARCHAR(100) NOT NULL DEFAULT 'host',
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    started_at          DATE,
    ended_at            DATE,

    -- Podcast Namespace person metadata
    person_metadata     JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "href": "https://example.com/host",
        "img": "https://example.com/host.jpg",
        "podcast_person_group": "hosts",
        "podcast_person_role": "host",
        "taxonomy_url": "https://podcasttaxonomy.com/role/host"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_show_hosts_show ON show_hosts(show_id);

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
    scheduled_at        TIMESTAMPTZ,
    hosting_external_id VARCHAR(255),

    -- Rich episode metadata (varies frequently with Podcast Namespace updates)
    episode_metadata    JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "itunes": {
            "type": "full",
            "explicit": false,
            "image": "https://...",
            "block": false
        },
        "podcast_namespace": {
            "transcript": [
                {"url": "https://.../transcript.srt", "type": "application/srt", "language": "en"},
                {"url": "https://.../transcript.vtt", "type": "text/vtt", "language": "en"}
            ],
            "chapters": {
                "url": "https://.../chapters.json",
                "type": "application/json+chapters"
            },
            "soundbites": [
                {"startTime": 120.5, "duration": 30.0, "title": "Key insight about..."}
            ],
            "persons": [
                {"name": "Guest Name", "role": "guest", "href": "https://...", "img": "https://..."}
            ],
            "location": {
                "geo": "geo:37.7749,-122.4194",
                "osm": "R111968",
                "name": "San Francisco, CA"
            },
            "season": {"name": "Season 3: The AI Era"},
            "alternateEnclosure": [
                {"url": "https://.../episode.opus", "type": "audio/opus", "length": 15000000, "bitrate": 96000}
            ]
        },
        "description_html": "<p>Full HTML description...</p>",
        "summary": "Plain text summary for feeds",
        "show_notes": "Extended show notes with links",
        "keywords": ["ai", "machine learning", "interview"]
    }
    */

    -- AI-generated annotations
    ai_annotations      JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "brand_safety": {
            "risk_score": 0.12,
            "flagged_categories": [],
            "flagged_segments": [],
            "scanned_at": "2026-05-25T10:00:00Z",
            "model_version": "v3.2"
        },
        "content_embedding": {
            "model": "text-embedding-3-large",
            "dimensions": 3072,
            "transcript_hash": "sha256:abc123...",
            "generated_at": "2026-05-25T10:05:00Z"
        },
        "suggested_ad_points": [
            {
                "offset_secs": 180.5,
                "position": "mid_roll",
                "confidence": 0.92,
                "reason": "natural_pause_after_topic_transition"
            },
            {
                "offset_secs": 720.0,
                "position": "mid_roll",
                "confidence": 0.87,
                "reason": "engagement_drop_off_recovery"
            }
        ],
        "auto_chapters": [
            {"startTime": 0, "title": "Introduction", "confidence": 0.95},
            {"startTime": 120.5, "title": "Topic: AI in Podcasting", "confidence": 0.88}
        ],
        "auto_tags": ["artificial intelligence", "podcasting", "technology trends"],
        "auto_summary": "In this episode, the hosts discuss..."
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    UNIQUE (show_id, slug)
);

CREATE INDEX idx_episodes_show_published ON episodes(show_id, published_at DESC)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_episodes_metadata ON episodes USING GIN (episode_metadata);
CREATE INDEX idx_episodes_ai ON episodes USING GIN (ai_annotations);
-- Partial index for episodes with brand safety concerns
CREATE INDEX idx_episodes_brand_safety ON episodes(
    (ai_annotations->'brand_safety'->>'risk_score'))
    WHERE (ai_annotations->'brand_safety'->>'risk_score')::numeric > 0.5;

-- Content embeddings stored separately for vector search (pgvector)
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

-- ============================================================
-- AD INSERTION POINTS (Relational — referenced by campaigns)
-- ============================================================

CREATE TABLE ad_insertion_points (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id),
    position            VARCHAR(20) NOT NULL,    -- 'pre_roll', 'mid_roll', 'post_roll'
    offset_secs         NUMERIC(10,3),
    max_duration_secs   INTEGER DEFAULT 60,
    is_auto_detected    BOOLEAN NOT NULL DEFAULT FALSE,
    confidence_score    NUMERIC(5,4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ad_points_episode ON ad_insertion_points(episode_id);

-- ============================================================
-- ADVERTISERS & CAMPAIGNS (Relational core + JSONB for specs)
-- ============================================================

CREATE TABLE advertisers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    company_name        VARCHAR(500) NOT NULL,
    contact_name        VARCHAR(255),
    contact_email       VARCHAR(320),
    industry_category   VARCHAR(255),
    billing_currency    VARCHAR(3) NOT NULL DEFAULT 'USD',

    -- Flexible advertiser profile for AI matching
    advertiser_profile  JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "website_url": "https://advertiser.example.com",
        "brand_guidelines_url": "https://...",
        "stripe_customer_id": "cus_xxx",
        "target_demographics": {
            "age_range": [25, 54],
            "gender": "all",
            "income_level": "upper_middle",
            "interests": ["technology", "business", "finance"]
        },
        "content_preferences": {
            "preferred_categories": ["Technology", "Business"],
            "excluded_categories": ["True Crime", "Politics"],
            "preferred_show_sizes": {"min_monthly_downloads": 10000},
            "brand_safety_threshold": 0.3
        },
        "billing_info": {
            "billing_address": {...},
            "payment_terms_days": 30,
            "net_terms": "net30"
        },
        "historical_performance": {
            "total_campaigns": 12,
            "total_spend": 150000,
            "avg_cpm": 25.50,
            "best_performing_shows": ["show-uuid-1", "show-uuid-2"]
        }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_advertisers_profile ON advertisers USING GIN (advertiser_profile);

CREATE TABLE campaigns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    network_id          UUID NOT NULL REFERENCES networks(id),
    advertiser_id       UUID NOT NULL REFERENCES advertisers(id),
    name                VARCHAR(500) NOT NULL,
    status              campaign_status NOT NULL DEFAULT 'draft',

    -- Core financial terms (normalized for reporting and joins)
    pricing_model       VARCHAR(30) NOT NULL DEFAULT 'cpm',
    inventory_type      VARCHAR(30) NOT NULL DEFAULT 'guaranteed',
    cpm_rate            NUMERIC(10,4),
    flat_rate_amount    NUMERIC(12,2),
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    booked_impressions  BIGINT,
    budget_amount       NUMERIC(12,2),
    flight_start        DATE NOT NULL,
    flight_end          DATE NOT NULL,

    -- Flexible campaign configuration
    campaign_config     JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "agency": {
            "id": "agency-uuid",
            "name": "Media Agency Co",
            "commission_pct": 15.0,
            "contact_email": "buyer@agency.com"
        },
        "creative": {
            "audio_url": "https://.../creative.mp3",
            "duration_secs": 30,
            "vast_tag_url": "https://.../vast.xml",
            "companion_banner_url": "https://.../banner.jpg",
            "companion_click_url": "https://advertiser.example.com/landing",
            "tracking_pixels": [
                {"event": "impression", "url": "https://tracker.example.com/imp"},
                {"event": "complete", "url": "https://tracker.example.com/complete"}
            ]
        },
        "targeting": {
            "geo_include": ["US", "CA", "GB"],
            "geo_exclude": ["CN"],
            "device_types": ["mobile", "desktop"],
            "app_include": ["Apple Podcasts", "Spotify"],
            "frequency_cap": {"max_per_listener": 3, "window_hours": 168},
            "dayparting": {
                "timezone": "America/New_York",
                "hours": [6, 7, 8, 9, 10, 17, 18, 19, 20]
            }
        },
        "brand_safety": {
            "max_risk_score": 0.3,
            "excluded_categories": ["violence", "adult_content"],
            "require_transcript_review": true
        },
        "exclusions": [
            {"type": "competitor", "advertiser_id": "comp-uuid", "company_name": "Competitor Inc"},
            {"type": "category", "category": "gambling"}
        ],
        "openrtb": {
            "deal_id": "deal_xxx",
            "bid_floor": 20.0,
            "seat_id": "seat_yyy",
            "pmp_enabled": true
        },
        "insertion_order": {
            "io_number": "IO-2026-0042",
            "signed_date": "2026-05-01",
            "document_url": "https://.../io.pdf"
        }
    }
    */

    -- Delivery summary (denormalized for dashboard performance)
    delivery_summary    JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "total_delivered": 450000,
        "total_valid": 425000,
        "delivery_pct": 85.0,
        "revenue_earned": 10625.00,
        "unique_listeners": 180000,
        "by_show": {
            "show-uuid-1": {"delivered": 200000, "valid": 190000},
            "show-uuid-2": {"delivered": 250000, "valid": 235000}
        },
        "by_position": {
            "pre_roll": {"delivered": 150000, "valid": 142000},
            "mid_roll": {"delivered": 300000, "valid": 283000}
        },
        "by_geo": {
            "US": 350000,
            "CA": 50000,
            "GB": 50000
        },
        "last_updated_at": "2026-05-25T10:00:00Z"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ,
    CONSTRAINT valid_flight CHECK (flight_end >= flight_start)
);

CREATE INDEX idx_campaigns_network ON campaigns(network_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_campaigns_status ON campaigns(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_campaigns_flight ON campaigns(flight_start, flight_end);
CREATE INDEX idx_campaigns_config ON campaigns USING GIN (campaign_config);

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

-- ============================================================
-- AD IMPRESSIONS (Relational for joins, JSONB for platform detail)
-- ============================================================

CREATE TABLE ad_impressions (
    id                  BIGSERIAL,
    campaign_id         UUID NOT NULL,
    episode_id          UUID NOT NULL,
    show_id             UUID NOT NULL,
    ad_position         VARCHAR(20) NOT NULL,
    listener_ip_hash    VARCHAR(64) NOT NULL,
    user_agent_hash     VARCHAR(64) NOT NULL,
    country_code        VARCHAR(2),
    device_type         VARCHAR(50),
    is_valid            BOOLEAN NOT NULL DEFAULT TRUE,
    delivered_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Platform-specific impression detail
    impression_detail   JSONB DEFAULT '{}',
    /*
    {
        "region_code": "US-CA",
        "city": "San Francisco",
        "app_name": "Apple Podcasts",
        "app_version": "15.3",
        "os": "iOS",
        "os_version": "18.2",
        "connection_type": "wifi",
        "iab_dedup_window_id": "win_xxx",
        "vast_events": {
            "impression": true,
            "firstQuartile": true,
            "midpoint": true,
            "thirdQuartile": true,
            "complete": true
        },
        "openrtb_response": {
            "bid_id": "bid_xxx",
            "seat_id": "seat_yyy",
            "win_price": 22.50,
            "adomain": ["advertiser.com"]
        }
    }
    */

    PRIMARY KEY (id, delivered_at)
) PARTITION BY RANGE (delivered_at);

-- Monthly partitions
CREATE TABLE ad_impressions_2026_01 PARTITION OF ad_impressions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE ad_impressions_2026_02 PARTITION OF ad_impressions
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... continue for each month

CREATE INDEX idx_impressions_campaign ON ad_impressions(campaign_id, delivered_at);
CREATE INDEX idx_impressions_dedup ON ad_impressions(listener_ip_hash, user_agent_hash, delivered_at);

-- ============================================================
-- CAMPAIGN RECONCILIATION (Relational for financials)
-- ============================================================

CREATE TABLE campaign_reconciliations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    booked_impressions  BIGINT NOT NULL,
    delivered_impressions BIGINT NOT NULL,
    valid_impressions   BIGINT NOT NULL,
    delivery_pct        NUMERIC(7,4),
    revenue_earned      NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL,
    reconciled_by       UUID REFERENCES users(id),
    reconciled_at       TIMESTAMPTZ,

    -- Detailed breakdown stored as JSONB
    reconciliation_detail JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "by_show": [
            {
                "show_id": "show-uuid-1",
                "show_title": "Tech Talk Daily",
                "booked": 100000,
                "delivered": 95000,
                "valid": 90000,
                "revenue": 2250.00
            }
        ],
        "by_position": {"pre_roll": 50000, "mid_roll": 40000},
        "invalid_breakdown": {
            "iab_dedup": 3000,
            "bot_detected": 1500,
            "geo_mismatch": 500
        },
        "adjustments": [],
        "advertiser_report_url": "https://.../report.pdf"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- HOST ROYALTY MANAGEMENT (Fully Normalized — financial data)
-- ============================================================

CREATE TABLE royalty_rules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    effective_from      DATE NOT NULL,
    effective_until     DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- Rule configuration (supports complex split structures)
    rule_config         JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "type": "percentage",
        "network_commission_pct": 20.0,
        "revenue_sources": ["ad_campaign", "subscription", "sponsorship"],
        "tiered_splits": false,
        "tiers": [],
        "notes": "Standard 80/20 split, hosts share the 80%"
    }
    
    Or for tiered splits:
    {
        "type": "tiered",
        "network_commission_pct": 15.0,
        "tiers": [
            {"up_to_revenue": 5000, "host_share_pct": 70},
            {"up_to_revenue": 20000, "host_share_pct": 75},
            {"up_to_revenue": null, "host_share_pct": 80}
        ]
    }
    */

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

    -- Revenue source breakdown
    revenue_breakdown   JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "campaigns": [
            {"campaign_id": "camp-uuid-1", "name": "Acme Q2", "amount": 5000.00},
            {"campaign_id": "camp-uuid-2", "name": "Widget Co", "amount": 3000.00}
        ],
        "subscriptions": {
            "total_subscribers": 250,
            "revenue": 2500.00
        },
        "sponsorships": [
            {"sponsor": "TechCorp", "amount": 1000.00}
        ]
    }
    */

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

    -- Adjustment detail
    adjustment_detail   JSONB DEFAULT '{}',
    /*
    {
        "advances_recouped": -500.00,
        "previous_underpayment": 25.50,
        "tax_withholding": -150.00,
        "notes": "Advance from January recouped"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- PAYMENT DISBURSEMENT (Fully Normalized — financial)
-- ============================================================

CREATE TABLE payment_profiles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    preferred_currency  VARCHAR(3) NOT NULL DEFAULT 'USD',
    payment_method      VARCHAR(50) NOT NULL DEFAULT 'bank_transfer',

    -- Payment provider details and tax forms
    payment_config      JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "stripe_connect_id": "acct_xxx",
        "stripe_onboarded": true,
        "tipalti_payee_id": "payee_xxx",
        "bank_country": "US",
        "tax_forms": [
            {
                "type": "w9",
                "status": "verified",
                "submitted_at": "2026-01-15T...",
                "verified_at": "2026-01-20T...",
                "document_url": "https://..."
            }
        ],
        "payout_threshold": 50.00,
        "payout_schedule": "monthly"
    }
    */

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
    currency            VARCHAR(3) NOT NULL,
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,

    -- Payment processing detail
    processing_detail   JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "exchange_rate": 0.85,
        "amount_local": 850.00,
        "local_currency": "EUR",
        "stripe_transfer_id": "tr_xxx",
        "tipalti_payment_id": "pay_xxx",
        "processed_at": "2026-05-20T14:00:00Z",
        "settlement_date": "2026-05-22",
        "failed_reason": null,
        "retry_count": 0,
        "statement_url": "https://.../statement.pdf",
        "line_items": [
            {"show_id": "show-uuid-1", "show_title": "Tech Talk", "amount": 500.00},
            {"show_id": "show-uuid-2", "show_title": "Code Cast", "amount": 350.00}
        ]
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payouts_recipient ON payouts(recipient_user_id, period_start);
CREATE INDEX idx_payouts_status ON payouts(status);

-- ============================================================
-- CROSS-PLATFORM ANALYTICS (JSONB-heavy — schema varies by platform)
-- ============================================================

CREATE TABLE analytics_sources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id),
    platform            hosting_platform NOT NULL,
    credentials_enc     BYTEA,
    last_sync_at        TIMESTAMPTZ,
    sync_status         VARCHAR(20) DEFAULT 'pending',

    -- Platform-specific sync configuration
    sync_config         JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "sync_interval_minutes": 60,
        "backfill_days": 90,
        "metrics_to_sync": ["downloads", "unique_listeners", "geo", "apps"],
        "api_version": "v3",
        "rate_limit_remaining": 4500,
        "last_error": null
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Unified analytics with platform-specific raw data preserved
CREATE TABLE episode_analytics (
    id                  BIGSERIAL,
    episode_id          UUID NOT NULL,
    show_id             UUID NOT NULL,
    source_platform     hosting_platform NOT NULL,
    metric_date         DATE NOT NULL,

    -- Normalized metrics (common across all platforms)
    downloads           BIGINT NOT NULL DEFAULT 0,
    unique_listeners    BIGINT DEFAULT 0,
    completion_rate     NUMERIC(5,4),
    avg_listen_duration_secs INTEGER,

    -- Platform-specific raw analytics preserved in JSONB
    raw_metrics         JSONB NOT NULL DEFAULT '{}',
    /*
    For Podbean (includes second-by-second engagement):
    {
        "iab_version": "2.1",
        "engagement_data": {
            "total_play_time_secs": 450000,
            "avg_play_depth_pct": 78.5,
            "drop_off_points": [
                {"offset_secs": 120, "drop_pct": 5.2},
                {"offset_secs": 600, "drop_pct": 12.1}
            ]
        },
        "subscriber_plays": 150,
        "non_subscriber_plays": 3200
    }
    
    For Megaphone:
    {
        "iab_version": "2.2",
        "spotify_audience_data": {
            "spotify_listeners": 5000,
            "spotify_starts": 4800,
            "spotify_completions": 3200,
            "spotify_demographics": {
                "age_18_24": 15,
                "age_25_34": 35,
                "age_35_44": 25,
                "age_45_plus": 25
            }
        },
        "metrics_export_batch_id": "batch_xxx"
    }
    
    For Buzzsprout:
    {
        "iab_version": "2.1",
        "total_plays": 3500,
        "top_apps": [
            {"name": "Apple Podcasts", "plays": 1500},
            {"name": "Spotify", "plays": 1200}
        ],
        "top_devices": [
            {"type": "mobile", "plays": 2800},
            {"type": "desktop", "plays": 700}
        ]
    }
    */

    geo_breakdown       JSONB DEFAULT '{}',
    /*
    {
        "US": {"downloads": 2500, "unique_listeners": 2000},
        "CA": {"downloads": 500, "unique_listeners": 400},
        "GB": {"downloads": 300, "unique_listeners": 250},
        "regions": {
            "US-CA": 800,
            "US-NY": 600,
            "US-TX": 400
        }
    }
    */

    app_breakdown       JSONB DEFAULT '{}',
    /*
    {
        "Apple Podcasts": {"downloads": 1500, "unique": 1200},
        "Spotify": {"downloads": 1200, "unique": 1000},
        "Overcast": {"downloads": 300, "unique": 280},
        "Pocket Casts": {"downloads": 200, "unique": 180}
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (episode_id, source_platform, metric_date),
    PRIMARY KEY (id, metric_date)
) PARTITION BY RANGE (metric_date);

CREATE INDEX idx_analytics_episode ON episode_analytics(episode_id, metric_date);
CREATE INDEX idx_analytics_show ON episode_analytics(show_id, metric_date);
CREATE INDEX idx_analytics_raw ON episode_analytics USING GIN (raw_metrics);

-- ============================================================
-- SUBSCRIPTION & MEMBERSHIP
-- ============================================================

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
    /*
    {
        "stripe_price_id": "price_xxx",
        "features": ["ad_free", "bonus_episodes", "early_access", "discord_access"],
        "entitlements": {
            "ad_free_feeds": true,
            "bonus_content": true,
            "early_access_hours": 48,
            "community_access": true
        },
        "trial_days": 7,
        "max_subscribers": null
    }
    */

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
    /*
    {
        "cancel_reason": "too_expensive",
        "private_feed_token": "tkn_xxx",
        "referral_source": "website",
        "lifetime_value": 125.00,
        "payment_history": [
            {"date": "2026-04-01", "amount": 5.00, "status": "paid"},
            {"date": "2026-05-01", "amount": 5.00, "status": "paid"}
        ]
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- REVENUE & P&L (Relational core for accounting)
-- ============================================================

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
    /*
    {
        "campaign_name": "Acme Q2 Campaign",
        "advertiser_name": "Acme Corp",
        "impressions_delivered": 50000,
        "cpm_rate": 25.00
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (revenue_date);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id                  BIGSERIAL,
    network_id          UUID,
    user_id             UUID,
    action              VARCHAR(100) NOT NULL,
    entity_type         VARCHAR(100) NOT NULL,
    entity_id           UUID NOT NULL,
    changes             JSONB,                 -- {field: {old: x, new: y}}
    request_metadata    JSONB DEFAULT '{}',    -- {ip, user_agent, request_id}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Query Examples

### Find shows matching advertiser preferences

```sql
-- Find shows whose category matches advertiser's preferred categories
SELECT s.id, s.title, s.podcast_metadata->'itunes'->>'category' AS category
FROM shows s
WHERE s.network_id = $1
  AND s.is_active = TRUE
  AND s.deleted_at IS NULL
  AND s.podcast_metadata->'itunes'->>'category' = ANY(
      SELECT jsonb_array_elements_text(
          a.advertiser_profile->'content_preferences'->'preferred_categories'
      )
      FROM advertisers a WHERE a.id = $2
  );
```

### Aggregate cross-platform analytics preserving platform detail

```sql
-- Get unified metrics with platform-specific detail for a show
SELECT
    ea.metric_date,
    SUM(ea.downloads) AS total_downloads,
    SUM(ea.unique_listeners) AS total_unique_listeners,
    jsonb_object_agg(
        ea.source_platform::text,
        jsonb_build_object(
            'downloads', ea.downloads,
            'unique_listeners', ea.unique_listeners,
            'raw_metrics', ea.raw_metrics
        )
    ) AS platform_detail
FROM episode_analytics ea
WHERE ea.show_id = $1
  AND ea.metric_date BETWEEN $2 AND $3
GROUP BY ea.metric_date
ORDER BY ea.metric_date DESC;
```

### Query episodes by AI-detected brand safety risk

```sql
-- Find episodes with high brand safety risk
SELECT e.id, e.title, e.published_at,
    e.ai_annotations->'brand_safety'->>'risk_score' AS risk_score,
    e.ai_annotations->'brand_safety'->'flagged_categories' AS flags
FROM episodes e
WHERE e.show_id = $1
  AND (e.ai_annotations->'brand_safety'->>'risk_score')::numeric > 0.5
ORDER BY (e.ai_annotations->'brand_safety'->>'risk_score')::numeric DESC;
```

---

## Pros and Cons

### Pros

1. **Schema evolution without migrations**: When Podcast Namespace adds a new tag (e.g., a future `podcast:recommendations` tag), it goes into the `podcast_metadata` JSONB column without a database migration. Similarly, when a new hosting platform is integrated, its unique analytics fields go into `raw_metrics` without altering the table structure.

2. **Platform-specific data preserved**: Podbean's second-by-second engagement data, Megaphone's Spotify Audience Network demographics, and Buzzsprout's app-level breakdown are all preserved in their original shape in `raw_metrics`, while the normalized columns (`downloads`, `unique_listeners`) provide a unified query surface.

3. **Single database technology**: Unlike approaches that require separate document stores (MongoDB) for flexible data and relational databases for structured data, this model uses a single PostgreSQL instance. This simplifies operations, backup, monitoring, and team skills requirements.

4. **ACID guarantees for financial operations**: Royalty calculations, payout records, and revenue entries use normalized columns with proper constraints and foreign keys. The financial core of the system has the same transactional safety as a fully normalized model.

5. **GIN-indexed JSONB queries**: PostgreSQL's GIN indexes on JSONB columns enable efficient containment queries (`@>` operator), JSON path queries, and partial indexing on extracted JSONB fields. The `idx_episodes_brand_safety` partial index demonstrates filtering episodes by a JSONB-extracted value.

6. **Denormalized delivery summaries**: The `campaigns.delivery_summary` JSONB column provides dashboard-ready delivery breakdowns without expensive joins across the impression table. It is updated asynchronously (via background job or trigger) from the impression data.

7. **Reduced table count**: Where the fully normalized model (Suggestion 1) would need separate `episode_transcripts`, `episode_chapters`, `episode_soundbites`, `episode_persons`, `episode_locations`, and `episode_alternate_enclosures` tables, this model captures all of these in `episode_metadata` JSONB. The schema has roughly 60% fewer tables while retaining the same expressiveness.

### Cons

1. **No referential integrity on JSONB data**: The `campaign_config.agency.id` field references an agency, but PostgreSQL cannot enforce this foreign key within JSONB. If the referenced agency is deleted, the JSONB reference becomes a dangling pointer. Application-level validation is required for all JSONB references.

2. **Query complexity for JSONB operations**: Extracting and filtering on nested JSONB fields requires `->`, `->>`, `#>`, `#>>`, `@>`, or `jsonb_path_query` operators, which are less readable than simple column comparisons and harder for junior developers to maintain.

3. **JSONB storage overhead**: JSONB stores field names with every row (unlike columnar formats). For the `ad_impressions` table at scale, the `impression_detail` JSONB column storing repeated keys like `"region_code"`, `"app_name"`, `"os"` in every row consumes more storage than normalized columns.

4. **Type safety concerns**: JSONB columns are schemaless at the database level. A typo in a JSONB key name (e.g., `"compltion_rate"` instead of `"completion_rate"`) is silently accepted. JSON Schema validation must be enforced at the application layer or via CHECK constraints with `jsonb_typeof()` and `jsonb_path_exists()`.

5. **Reporting complexity**: While the normalized metrics (`downloads`, `unique_listeners`) support straightforward aggregation, cross-platform reports that need platform-specific detail (e.g., "show me Podbean engagement data alongside Megaphone Spotify demographics") require JSONB path extraction in aggregate queries, which is less performant and harder to optimize than column-based aggregation.

6. **Migration testing**: When the JSONB schema evolves (e.g., `advertiser_profile.target_demographics` adds a new field), existing rows may have the old schema. Application code must handle both old and new shapes, and backfilling old rows requires JSON transformation queries that are harder to test than ALTER TABLE migrations.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Database | PostgreSQL 16+ | Native JSONB, GIN indexes, partitioning, pgvector |
| JSONB validation | JSON Schema (application-side, via AJV or Zod) | Enforce JSONB shape before write |
| ORM | Prisma with `Json` type or Drizzle ORM | Both handle JSONB columns natively |
| Migration | Prisma Migrate + custom JSONB migration scripts | Standard migrations for relational, scripts for JSONB backfills |
| Caching | Redis for delivery summaries | Cache frequently-read JSONB aggregates |
| Search | PostgreSQL GIN + optional Meilisearch | GIN for JSONB, Meilisearch for full-text episode search |
| Monitoring | pg_stat_statements + JSONB query analysis | Track JSONB query performance separately |

---

## Migration and Scaling Considerations

### JSONB Schema Governance

Establish a JSONB schema registry (a directory of JSON Schema files in the codebase) for each JSONB column. Use application-layer validation (Zod in TypeScript, Pydantic in Python) to enforce these schemas on write. Version the schemas with a `"schema_version"` field in the JSONB payload to enable backward-compatible reads.

```typescript
// Example: Episode metadata schema versioning
const episodeMetadataV2 = z.object({
    schema_version: z.literal(2),
    itunes: z.object({ ... }),
    podcast_namespace: z.object({
        transcript: z.array(z.object({ ... })),
        chapters: z.object({ ... }).optional(),
        // New in v2: recommendations tag
        recommendations: z.array(z.object({
            feed_guid: z.string(),
            text: z.string().optional()
        })).optional(),
    }),
});
```

### Scaling Path

1. **0-50 shows**: Single PostgreSQL instance. JSONB columns are small. GIN indexes are sufficient.
2. **50-500 shows**: Partition `ad_impressions` and `episode_analytics` by month. Consider moving `delivery_summary` updates to an async worker to avoid write amplification on the campaigns table.
3. **500+ shows**: The `raw_metrics` JSONB in `episode_analytics` becomes the largest storage consumer. Consider compressing old partitions or archiving raw metrics to object storage after the normalized fields have been extracted.

### Migration from Fully Normalized Model

If starting with Suggestion 1 and migrating to this hybrid model:
1. Add JSONB columns to existing tables (non-breaking)
2. Backfill JSONB columns from normalized child tables using `jsonb_build_object()` and `jsonb_agg()`
3. Update application code to read from JSONB columns
4. Run parallel reads (normalized + JSONB) for validation period
5. Drop normalized child tables once JSONB reads are validated
6. This migration is lower-risk than the event-sourcing migration (Suggestion 2) and typically takes 4-8 weeks
