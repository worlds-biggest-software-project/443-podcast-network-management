# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Podcast Network Management (Candidate #443)
> Generated: 2026-05-25

## Overview

An event-sourced architecture with Command Query Responsibility Segregation (CQRS), where every state change across the system is captured as an immutable event in an append-only event store. Read-optimized projections (read models) are derived from the event stream to serve queries. This approach is particularly well-suited to the podcast network management domain because of three characteristics:

1. **Financial auditability**: Royalty calculations, campaign reconciliations, and payment disbursements require a complete, tamper-proof audit trail. Event sourcing provides this by construction -- every state transition is preserved, and the current state can be reconstructed from the event history at any point.

2. **Multi-system integration**: The platform aggregates data from 5+ hosting platforms, ad exchanges (OpenRTB), payment processors (Stripe Connect, Tipalti), and analytics providers. Each integration produces events that must be reliably captured, even when downstream projections temporarily fail.

3. **Temporal queries**: "What was the royalty split for Show X on March 15?" and "What campaigns were active when this impression was served?" are naturally answered by event sourcing's temporal model, without needing to maintain separate audit tables or slowly-changing-dimension patterns.

The event store uses PostgreSQL for its transactional guarantees and operational familiarity, while read models use a combination of PostgreSQL (for relational queries) and Redis (for real-time dashboards). For very high-throughput impression ingestion, an Apache Kafka topic serves as the entry point with guaranteed ordering.

---

## Architecture Overview

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         Commands        Queries       Webhooks
              │              │         (Stripe, hosting
              ▼              ▼          platforms)
    ┌─────────────────┐  ┌──────────┐     │
    │ Command Handlers│  │  Query   │     │
    │ (Aggregates)    │  │ Service  │     ▼
    └────────┬────────┘  └────┬─────┘  ┌──────────┐
             │                │        │  Event   │
             ▼                │        │ Ingestion│
    ┌─────────────────┐       │        └────┬─────┘
    │   Event Store   │       │             │
    │  (PostgreSQL)   │───────┤             │
    └────────┬────────┘       │             │
             │                │             │
             ▼                ▼             ▼
    ┌─────────────────┐  ┌──────────┐  ┌──────────┐
    │   Event Bus     │  │  Read    │  │  Kafka   │
    │  (internal)     │  │  Models  │  │ (high    │
    └────────┬────────┘  │(Postgres)│  │ volume)  │
             │           └──────────┘  └──────────┘
             ▼
    ┌─────────────────┐
    │   Projections   │
    │  (Event         │
    │   Handlers)     │
    └─────────────────┘
```

---

## Event Store Schema

```sql
-- ============================================================
-- EVENT STORE (Append-Only)
-- ============================================================

-- Core event store table
CREATE TABLE events (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         VARCHAR(100) NOT NULL,   -- 'Show', 'Episode', 'Campaign', 'RoyaltyRule', 'Payout'
    stream_id           UUID NOT NULL,           -- Aggregate root ID
    event_type          VARCHAR(200) NOT NULL,   -- e.g. 'ShowCreated', 'CampaignBooked', 'RoyaltyCalculated'
    event_version       INTEGER NOT NULL,        -- Per-stream sequence number (optimistic concurrency)
    payload             JSONB NOT NULL,          -- Event data
    metadata            JSONB NOT NULL DEFAULT '{}', -- Correlation ID, causation ID, user ID, IP
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_id, event_version)            -- Optimistic concurrency control
) PARTITION BY RANGE (created_at);

-- Monthly partitions for the event store
CREATE TABLE events_2026_01 PARTITION OF events FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... continue per month

CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_stream_type ON events(stream_type, created_at);
CREATE INDEX idx_events_correlation ON events((metadata->>'correlation_id'));

-- Snapshot store for performance (avoid replaying full history for long-lived aggregates)
CREATE TABLE snapshots (
    stream_id           UUID NOT NULL,
    stream_type         VARCHAR(100) NOT NULL,
    snapshot_version    INTEGER NOT NULL,        -- Event version at snapshot time
    state               JSONB NOT NULL,          -- Serialized aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Projection checkpoints (tracks which event each projection has processed)
CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(200) PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_event_version  INTEGER NOT NULL,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Dead letter queue for failed event processing
CREATE TABLE dead_letter_events (
    id                  BIGSERIAL PRIMARY KEY,
    event_id            UUID NOT NULL,
    projection_name     VARCHAR(200) NOT NULL,
    error_message       TEXT NOT NULL,
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 5,
    next_retry_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Event Type Catalogue

### Show & Episode Domain

```typescript
// Event type definitions (TypeScript for clarity; stored as JSONB in the event store)

// === Show Aggregate ===

interface ShowCreated {
    type: 'ShowCreated';
    networkId: string;
    title: string;
    slug: string;
    description: string;
    language: string;
    hostingPlatform: 'megaphone' | 'libsyn' | 'acast' | 'podbean' | 'buzzsprout' | 'self_hosted';
    hostingExternalId?: string;
    rssFeedUrl?: string;
    itunesCategory?: string;
    podcastGuid?: string;
}

interface ShowMetadataUpdated {
    type: 'ShowMetadataUpdated';
    changes: {
        title?: string;
        description?: string;
        coverImageUrl?: string;
        websiteUrl?: string;
        language?: string;
        explicit?: boolean;
    };
}

interface ShowHostAdded {
    type: 'ShowHostAdded';
    hostId: string;
    userId?: string;
    personName: string;
    personRole: 'host' | 'co_host' | 'producer' | 'editor';
    isPrimary: boolean;
    startedAt: string;           // ISO date
}

interface ShowHostRemoved {
    type: 'ShowHostRemoved';
    hostId: string;
    endedAt: string;
}

interface ShowDeactivated {
    type: 'ShowDeactivated';
    reason: string;
}

// === Episode Aggregate ===

interface EpisodeDrafted {
    type: 'EpisodeDrafted';
    showId: string;
    title: string;
    slug: string;
    description: string;
    seasonNumber?: number;
    episodeNumber?: number;
    episodeType: 'full' | 'trailer' | 'bonus';
}

interface EpisodeAudioUploaded {
    type: 'EpisodeAudioUploaded';
    audioUrl: string;
    durationSecs: number;
    sizeBytes: number;
    mimeType: string;
}

interface EpisodeTranscriptAttached {
    type: 'EpisodeTranscriptAttached';
    transcriptUrl: string;
    transcriptFormat: 'srt' | 'vtt' | 'json';
    language: string;
}

interface EpisodeChaptersSet {
    type: 'EpisodeChaptersSet';
    chaptersUrl: string;           // podcast:chapters JSON URL
    chapterCount: number;
}

interface AdInsertionPointMarked {
    type: 'AdInsertionPointMarked';
    insertionPointId: string;
    position: 'pre_roll' | 'mid_roll' | 'post_roll';
    offsetSecs: number;
    maxDurationSecs: number;
    isAutoDetected: boolean;
    confidenceScore?: number;
}

interface AdInsertionPointRemoved {
    type: 'AdInsertionPointRemoved';
    insertionPointId: string;
    reason: string;
}

interface EpisodeScheduled {
    type: 'EpisodeScheduled';
    scheduledAt: string;           // ISO timestamp
}

interface EpisodePublished {
    type: 'EpisodePublished';
    publishedAt: string;
    guid: string;
    hostingExternalId?: string;
}

interface EpisodeArchived {
    type: 'EpisodeArchived';
    reason: string;
}

interface BrandSafetyScanCompleted {
    type: 'BrandSafetyScanCompleted';
    riskScore: number;
    flaggedCategories: string[];
    flaggedSegments: Array<{
        startSecs: number;
        endSecs: number;
        reason: string;
    }>;
    modelVersion: string;
}
```

### Campaign & Ad Delivery Domain

```typescript
// === Campaign Aggregate ===

interface CampaignCreated {
    type: 'CampaignCreated';
    networkId: string;
    advertiserId: string;
    agencyId?: string;
    name: string;
    pricingModel: 'cpm' | 'flat_rate' | 'revenue_share' | 'hybrid';
    inventoryType: 'guaranteed' | 'programmatic' | 'house';
    cpmRate?: number;
    flatRateAmount?: number;
    currency: string;
    bookedImpressions?: number;
    budgetAmount?: number;
    flightStart: string;
    flightEnd: string;
}

interface CampaignShowTargeted {
    type: 'CampaignShowTargeted';
    showId: string;
    adPosition: 'pre_roll' | 'mid_roll' | 'post_roll';
    allocatedImpressions?: number;
    weight: number;
}

interface CampaignShowUntargeted {
    type: 'CampaignShowUntargeted';
    showId: string;
    reason: string;
}

interface CampaignCreativeSet {
    type: 'CampaignCreativeSet';
    creativeUrl: string;
    creativeDurationSecs: number;
    vastTagUrl?: string;
}

interface CampaignExclusionAdded {
    type: 'CampaignExclusionAdded';
    excludedAdvertiserId?: string;
    excludedCategory?: string;
}

interface CampaignBooked {
    type: 'CampaignBooked';
    approvedBy: string;
}

interface CampaignActivated {
    type: 'CampaignActivated';
}

interface CampaignPaused {
    type: 'CampaignPaused';
    reason: string;
}

interface CampaignResumed {
    type: 'CampaignResumed';
}

interface CampaignCompleted {
    type: 'CampaignCompleted';
    finalImpressions: number;
    finalRevenue: number;
}

interface CampaignCancelled {
    type: 'CampaignCancelled';
    reason: string;
    cancellationFee?: number;
}

// === Impression Events (high volume — routed via Kafka) ===

interface AdImpressionServed {
    type: 'AdImpressionServed';
    campaignId: string;
    episodeId: string;
    showId: string;
    insertionPointId?: string;
    adPosition: 'pre_roll' | 'mid_roll' | 'post_roll';
    listenerIpHash: string;
    userAgentHash: string;
    countryCode?: string;
    regionCode?: string;
    deviceType?: string;
    appName?: string;
    servedAt: string;
}

interface AdImpressionInvalidated {
    type: 'AdImpressionInvalidated';
    impressionId: string;
    reason: 'iab_dedup' | 'bot_detected' | 'fraud_flagged' | 'manual_override';
}

// === Campaign Reconciliation ===

interface CampaignReconciled {
    type: 'CampaignReconciled';
    periodStart: string;
    periodEnd: string;
    bookedImpressions: number;
    deliveredImpressions: number;
    validImpressions: number;
    deliveryPct: number;
    revenueEarned: number;
    currency: string;
    reconciledBy: string;
}
```

### Royalty & Payment Domain

```typescript
// === Royalty Rule Aggregate ===

interface RoyaltyRuleCreated {
    type: 'RoyaltyRuleCreated';
    showId: string;
    name: string;
    description?: string;
    effectiveFrom: string;
    effectiveUntil?: string;
}

interface RoyaltySplitConfigured {
    type: 'RoyaltySplitConfigured';
    splitId: string;
    recipientHostId: string;
    splitPercentage: number;
    minimumAmount?: number;
    capAmount?: number;
}

interface RoyaltySplitUpdated {
    type: 'RoyaltySplitUpdated';
    splitId: string;
    previousPercentage: number;
    newPercentage: number;
}

interface RoyaltyRuleDeactivated {
    type: 'RoyaltyRuleDeactivated';
    reason: string;
    supersededBy?: string;
}

// === Royalty Calculation Aggregate ===

interface RoyaltyCalculationStarted {
    type: 'RoyaltyCalculationStarted';
    showId: string;
    royaltyRuleId: string;
    periodStart: string;
    periodEnd: string;
    totalRevenue: number;
    adRevenue: number;
    subscriptionRevenue: number;
    otherRevenue: number;
}

interface RoyaltyNetworkCommissionApplied {
    type: 'RoyaltyNetworkCommissionApplied';
    commissionRate: number;
    commissionAmount: number;
    distributableAmount: number;
}

interface RoyaltyLineItemCalculated {
    type: 'RoyaltyLineItemCalculated';
    lineItemId: string;
    recipientHostId: string;
    splitPercentage: number;
    grossAmount: number;
    adjustments: number;
    netAmount: number;
    currency: string;
}

interface RoyaltyCalculationApproved {
    type: 'RoyaltyCalculationApproved';
    approvedBy: string;
}

interface RoyaltyCalculationRejected {
    type: 'RoyaltyCalculationRejected';
    rejectedBy: string;
    reason: string;
}

// === Payout Aggregate ===

interface PayoutInitiated {
    type: 'PayoutInitiated';
    networkId: string;
    recipientUserId: string;
    paymentProfileId: string;
    amount: number;
    currency: string;
    exchangeRate?: number;
    amountLocal?: number;
    periodStart: string;
    periodEnd: string;
    lineItems: Array<{
        royaltyLineItemId: string;
        showId: string;
        amount: number;
    }>;
}

interface PayoutProcessing {
    type: 'PayoutProcessing';
    stripeTransferId?: string;
    tipaltiPaymentId?: string;
}

interface PayoutCompleted {
    type: 'PayoutCompleted';
    processedAt: string;
    statementUrl?: string;
}

interface PayoutFailed {
    type: 'PayoutFailed';
    failedReason: string;
    retryScheduledAt?: string;
}

interface PayoutReversed {
    type: 'PayoutReversed';
    reason: string;
    reversalAmount: number;
}
```

### Analytics & Subscription Domain

```typescript
// === Analytics Sync ===

interface AnalyticsSyncStarted {
    type: 'AnalyticsSyncStarted';
    showId: string;
    platform: string;
    syncPeriodStart: string;
    syncPeriodEnd: string;
}

interface AnalyticsDataIngested {
    type: 'AnalyticsDataIngested';
    episodeId: string;
    showId: string;
    sourcePlatform: string;
    metricDate: string;
    downloads: number;
    uniqueListeners?: number;
    completionRate?: number;
    avgListenDurationSecs?: number;
    iabVersion?: string;
}

interface AnalyticsGeoDataIngested {
    type: 'AnalyticsGeoDataIngested';
    episodeId: string;
    metricDate: string;
    countryCode: string;
    regionCode?: string;
    downloads: number;
    uniqueListeners?: number;
}

interface AnalyticsSyncCompleted {
    type: 'AnalyticsSyncCompleted';
    showId: string;
    platform: string;
    recordsIngested: number;
}

interface AnalyticsSyncFailed {
    type: 'AnalyticsSyncFailed';
    showId: string;
    platform: string;
    errorMessage: string;
}

interface AnomalyDetected {
    type: 'AnomalyDetected';
    episodeId: string;
    metricDate: string;
    anomalyType: 'bot_traffic' | 'download_spike' | 'geo_anomaly';
    details: string;
    severity: 'low' | 'medium' | 'high';
}

// === Subscription ===

interface SubscriptionCreated {
    type: 'SubscriptionCreated';
    planId: string;
    subscriberEmail: string;
    subscriberName?: string;
    tier: string;
    priceAmount: number;
    currency: string;
    stripeSubscriptionId?: string;
}

interface SubscriptionRenewed {
    type: 'SubscriptionRenewed';
    periodStart: string;
    periodEnd: string;
    amount: number;
}

interface SubscriptionCancelled {
    type: 'SubscriptionCancelled';
    cancelReason?: string;
    effectiveDate: string;
}
```

---

## Read Model Projections (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: Show Catalogue
-- ============================================================

CREATE TABLE rm_shows (
    id                  UUID PRIMARY KEY,
    network_id          UUID NOT NULL,
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,
    language            VARCHAR(10),
    explicit            BOOLEAN DEFAULT FALSE,
    cover_image_url     VARCHAR(2048),
    rss_feed_url        VARCHAR(2048),
    hosting_platform    VARCHAR(50),
    is_active           BOOLEAN DEFAULT TRUE,
    episode_count       INTEGER DEFAULT 0,        -- Denormalized counter
    total_downloads     BIGINT DEFAULT 0,         -- Denormalized aggregate
    last_published_at   TIMESTAMPTZ,              -- Denormalized
    host_names          TEXT[],                    -- Denormalized for display
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_episodes (
    id                  UUID PRIMARY KEY,
    show_id             UUID NOT NULL,
    network_id          UUID NOT NULL,
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200),
    status              VARCHAR(20),
    season_number       INTEGER,
    episode_number      INTEGER,
    episode_type        VARCHAR(20),
    audio_url           VARCHAR(2048),
    audio_duration_secs INTEGER,
    published_at        TIMESTAMPTZ,
    total_downloads     BIGINT DEFAULT 0,         -- Denormalized
    ad_point_count      INTEGER DEFAULT 0,        -- Denormalized
    brand_safety_score  NUMERIC(5,4),             -- Latest scan result
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_episodes_show ON rm_episodes(show_id, published_at DESC);

CREATE TABLE rm_show_hosts (
    id                  UUID PRIMARY KEY,
    show_id             UUID NOT NULL,
    user_id             UUID,
    person_name         VARCHAR(255) NOT NULL,
    person_role         VARCHAR(100),
    is_primary          BOOLEAN DEFAULT FALSE,
    is_active           BOOLEAN DEFAULT TRUE,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- READ MODEL: Campaign Dashboard
-- ============================================================

CREATE TABLE rm_campaigns (
    id                  UUID PRIMARY KEY,
    network_id          UUID NOT NULL,
    advertiser_name     VARCHAR(500),             -- Denormalized
    agency_name         VARCHAR(500),             -- Denormalized
    name                VARCHAR(500) NOT NULL,
    status              VARCHAR(30),
    pricing_model       VARCHAR(30),
    inventory_type      VARCHAR(30),
    cpm_rate            NUMERIC(10,4),
    flat_rate_amount    NUMERIC(12,2),
    currency            VARCHAR(3),
    booked_impressions  BIGINT,
    budget_amount       NUMERIC(12,2),
    flight_start        DATE,
    flight_end          DATE,
    -- Delivery summary (denormalized, updated by impression projection)
    delivered_impressions BIGINT DEFAULT 0,
    valid_impressions   BIGINT DEFAULT 0,
    delivery_pct        NUMERIC(7,4) DEFAULT 0,
    revenue_earned      NUMERIC(12,2) DEFAULT 0,
    targeted_show_count INTEGER DEFAULT 0,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_campaigns_status ON rm_campaigns(status);
CREATE INDEX idx_rm_campaigns_flight ON rm_campaigns(flight_start, flight_end);

CREATE TABLE rm_campaign_daily_delivery (
    campaign_id         UUID NOT NULL,
    show_id             UUID NOT NULL,
    ad_position         VARCHAR(20),
    delivery_date       DATE NOT NULL,
    valid_impressions   BIGINT DEFAULT 0,
    invalid_impressions BIGINT DEFAULT 0,
    unique_listeners    BIGINT DEFAULT 0,
    PRIMARY KEY (campaign_id, show_id, delivery_date, ad_position)
);

-- ============================================================
-- READ MODEL: Royalty & Payout Dashboard
-- ============================================================

CREATE TABLE rm_royalty_calculations (
    id                  UUID PRIMARY KEY,
    show_id             UUID NOT NULL,
    show_title          VARCHAR(500),             -- Denormalized
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    total_revenue       NUMERIC(14,2),
    ad_revenue          NUMERIC(14,2),
    subscription_revenue NUMERIC(14,2),
    network_commission  NUMERIC(14,2),
    distributable_amount NUMERIC(14,2),
    currency            VARCHAR(3),
    status              VARCHAR(20),              -- 'pending', 'approved', 'rejected'
    approved_by_name    VARCHAR(255),
    calculated_at       TIMESTAMPTZ,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_royalty_line_items (
    id                  UUID PRIMARY KEY,
    calculation_id      UUID NOT NULL,
    recipient_name      VARCHAR(255),             -- Denormalized host name
    recipient_user_id   UUID,
    split_percentage    NUMERIC(7,4),
    gross_amount        NUMERIC(14,2),
    adjustments         NUMERIC(14,2),
    net_amount          NUMERIC(14,2),
    currency            VARCHAR(3),
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_payouts (
    id                  UUID PRIMARY KEY,
    network_id          UUID NOT NULL,
    recipient_name      VARCHAR(255),             -- Denormalized
    recipient_email     VARCHAR(320),             -- Denormalized
    status              VARCHAR(20),
    amount              NUMERIC(14,2),
    currency            VARCHAR(3),
    period_start        DATE,
    period_end          DATE,
    processed_at        TIMESTAMPTZ,
    show_breakdown      JSONB,                    -- [{showId, showTitle, amount}]
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_payouts_status ON rm_payouts(status);

-- ============================================================
-- READ MODEL: Network Analytics
-- ============================================================

CREATE TABLE rm_episode_analytics_daily (
    episode_id          UUID NOT NULL,
    show_id             UUID NOT NULL,
    network_id          UUID NOT NULL,
    metric_date         DATE NOT NULL,
    total_downloads     BIGINT DEFAULT 0,
    unique_listeners    BIGINT DEFAULT 0,
    completion_rate     NUMERIC(5,4),
    source_platforms    TEXT[],                    -- Which platforms reported data
    PRIMARY KEY (episode_id, metric_date)
);

CREATE TABLE rm_show_analytics_monthly (
    show_id             UUID NOT NULL,
    network_id          UUID NOT NULL,
    month               DATE NOT NULL,            -- First day of month
    total_downloads     BIGINT DEFAULT 0,
    unique_listeners    BIGINT DEFAULT 0,
    avg_completion_rate NUMERIC(5,4),
    episode_count       INTEGER DEFAULT 0,
    new_episodes        INTEGER DEFAULT 0,
    top_countries       JSONB,                    -- [{code, downloads}] top 10
    top_apps            JSONB,                    -- [{name, downloads}] top 10
    PRIMARY KEY (show_id, month)
);

CREATE TABLE rm_network_pnl (
    network_id          UUID NOT NULL,
    month               DATE NOT NULL,
    total_revenue       NUMERIC(14,2) DEFAULT 0,
    ad_revenue          NUMERIC(14,2) DEFAULT 0,
    subscription_revenue NUMERIC(14,2) DEFAULT 0,
    other_revenue       NUMERIC(14,2) DEFAULT 0,
    total_payouts       NUMERIC(14,2) DEFAULT 0,
    agency_commissions  NUMERIC(14,2) DEFAULT 0,
    net_margin          NUMERIC(14,2) DEFAULT 0,
    show_count          INTEGER DEFAULT 0,
    PRIMARY KEY (network_id, month)
);
```

---

## Projection Handlers (Pseudocode)

```typescript
// === Show Catalogue Projection ===

class ShowCatalogueProjection {
    async handle(event: Event): Promise<void> {
        switch (event.type) {
            case 'ShowCreated':
                await db.query(`
                    INSERT INTO rm_shows (id, network_id, title, slug, description,
                        language, hosting_platform, is_active, last_updated_at)
                    VALUES ($1, $2, $3, $4, $5, $6, $7, true, NOW())
                `, [event.streamId, event.payload.networkId, event.payload.title,
                    event.payload.slug, event.payload.description,
                    event.payload.language, event.payload.hostingPlatform]);
                break;

            case 'EpisodePublished':
                await db.query(`
                    UPDATE rm_shows
                    SET episode_count = episode_count + 1,
                        last_published_at = $2,
                        last_updated_at = NOW()
                    WHERE id = (SELECT show_id FROM rm_episodes WHERE id = $1)
                `, [event.streamId, event.payload.publishedAt]);
                break;

            case 'ShowHostAdded':
                await db.query(`
                    UPDATE rm_shows
                    SET host_names = array_append(host_names, $2),
                        last_updated_at = NOW()
                    WHERE id = $1
                `, [event.streamId, event.payload.personName]);
                break;
        }
    }
}

// === Campaign Delivery Projection ===

class CampaignDeliveryProjection {
    async handle(event: Event): Promise<void> {
        switch (event.type) {
            case 'AdImpressionServed':
                // Update daily delivery
                await db.query(`
                    INSERT INTO rm_campaign_daily_delivery
                        (campaign_id, show_id, ad_position, delivery_date,
                         valid_impressions, unique_listeners)
                    VALUES ($1, $2, $3, $4::date, 1, 1)
                    ON CONFLICT (campaign_id, show_id, delivery_date, ad_position)
                    DO UPDATE SET
                        valid_impressions = rm_campaign_daily_delivery.valid_impressions + 1,
                        unique_listeners = rm_campaign_daily_delivery.unique_listeners + 1
                `, [event.payload.campaignId, event.payload.showId,
                    event.payload.adPosition, event.payload.servedAt]);

                // Update campaign totals
                await db.query(`
                    UPDATE rm_campaigns
                    SET delivered_impressions = delivered_impressions + 1,
                        valid_impressions = valid_impressions + 1,
                        delivery_pct = CASE WHEN booked_impressions > 0
                            THEN ((valid_impressions + 1)::numeric / booked_impressions) * 100
                            ELSE 0 END,
                        last_updated_at = NOW()
                    WHERE id = $1
                `, [event.payload.campaignId]);
                break;

            case 'AdImpressionInvalidated':
                await db.query(`
                    UPDATE rm_campaigns
                    SET valid_impressions = valid_impressions - 1,
                        delivery_pct = CASE WHEN booked_impressions > 0
                            THEN ((valid_impressions - 1)::numeric / booked_impressions) * 100
                            ELSE 0 END,
                        last_updated_at = NOW()
                    WHERE id = (
                        SELECT payload->>'campaignId' FROM events WHERE event_id = $1
                    )
                `, [event.payload.impressionId]);
                break;
        }
    }
}

// === Royalty & Payout Projection ===

class RoyaltyProjection {
    async handle(event: Event): Promise<void> {
        switch (event.type) {
            case 'RoyaltyCalculationStarted':
                await db.query(`
                    INSERT INTO rm_royalty_calculations
                        (id, show_id, show_title, period_start, period_end,
                         total_revenue, ad_revenue, subscription_revenue,
                         status, calculated_at, last_updated_at)
                    VALUES ($1, $2,
                        (SELECT title FROM rm_shows WHERE id = $2),
                        $3, $4, $5, $6, $7, 'pending', NOW(), NOW())
                `, [event.streamId, event.payload.showId,
                    event.payload.periodStart, event.payload.periodEnd,
                    event.payload.totalRevenue, event.payload.adRevenue,
                    event.payload.subscriptionRevenue]);
                break;

            case 'RoyaltyLineItemCalculated':
                await db.query(`
                    INSERT INTO rm_royalty_line_items
                        (id, calculation_id, recipient_name, recipient_user_id,
                         split_percentage, gross_amount, adjustments, net_amount,
                         currency, last_updated_at)
                    VALUES ($1, $2,
                        (SELECT person_name FROM rm_show_hosts WHERE id = $3),
                        (SELECT user_id FROM rm_show_hosts WHERE id = $3),
                        $4, $5, $6, $7, $8, NOW())
                `, [event.payload.lineItemId, event.streamId,
                    event.payload.recipientHostId, event.payload.splitPercentage,
                    event.payload.grossAmount, event.payload.adjustments,
                    event.payload.netAmount, event.payload.currency]);
                break;

            case 'PayoutCompleted':
                await db.query(`
                    UPDATE rm_payouts
                    SET status = 'completed',
                        processed_at = $2,
                        last_updated_at = NOW()
                    WHERE id = $1
                `, [event.streamId, event.payload.processedAt]);

                // Update network P&L
                await db.query(`
                    INSERT INTO rm_network_pnl (network_id, month, total_payouts)
                    VALUES ($1, DATE_TRUNC('month', $2::date), $3)
                    ON CONFLICT (network_id, month)
                    DO UPDATE SET total_payouts = rm_network_pnl.total_payouts + $3
                `, [/* derive networkId */, event.payload.processedAt,
                    /* derive amount */]);
                break;
        }
    }
}
```

---

## High-Volume Impression Ingestion (Kafka)

```
Kafka Topics:
├── podcast.impressions.raw         (partitioned by campaignId, 32 partitions)
│   └── AdImpressionServed events
├── podcast.impressions.validated   (after IAB v2.2 dedup)
│   └── Validated impressions forwarded to event store
├── podcast.impressions.invalidated
│   └── AdImpressionInvalidated events
├── podcast.analytics.sync          (partitioned by showId)
│   └── AnalyticsDataIngested events from hosting platforms
└── podcast.dlq                     (dead letter queue)
    └── Failed events for manual review
```

```typescript
// IAB v2.2 Dedup Consumer (processes raw impressions)
class IabDedupConsumer {
    // Uses a Redis sorted set for 24-hour dedup windows
    // Key: `dedup:{campaignId}:{ipHash}:{uaHash}`
    // Score: Unix timestamp of first impression
    // TTL: 86400 seconds (24 hours)

    async processImpression(event: AdImpressionServed): Promise<void> {
        const dedupKey = `dedup:${event.campaignId}:${event.listenerIpHash}:${event.userAgentHash}`;
        const exists = await redis.exists(dedupKey);

        if (exists) {
            // Duplicate within 24-hour window — invalidate
            await kafka.produce('podcast.impressions.invalidated', {
                type: 'AdImpressionInvalidated',
                impressionId: event.eventId,
                reason: 'iab_dedup'
            });
        } else {
            // First impression in window — mark valid
            await redis.set(dedupKey, event.servedAt, 'EX', 86400);
            await kafka.produce('podcast.impressions.validated', event);

            // Also write to event store for persistence
            await eventStore.append('Impression', event.eventId, event);
        }
    }
}
```

---

## Aggregate Root Examples

```typescript
// === Campaign Aggregate ===

class CampaignAggregate {
    private state: CampaignState;
    private uncommittedEvents: Event[] = [];

    constructor(private streamId: string) {
        this.state = { status: 'draft', showTargets: [], exclusions: [] };
    }

    // Rehydrate from event stream
    static async load(streamId: string, eventStore: EventStore): Promise<CampaignAggregate> {
        const agg = new CampaignAggregate(streamId);
        const events = await eventStore.getStream('Campaign', streamId);

        // Check for snapshot first
        const snapshot = await eventStore.getLatestSnapshot(streamId);
        if (snapshot) {
            agg.state = snapshot.state;
            // Only replay events after snapshot
            const recentEvents = events.filter(e => e.event_version > snapshot.snapshot_version);
            for (const event of recentEvents) {
                agg.apply(event.payload);
            }
        } else {
            for (const event of events) {
                agg.apply(event.payload);
            }
        }
        return agg;
    }

    // Command: Book the campaign
    book(approvedBy: string): void {
        // Business rule validation
        if (this.state.status !== 'draft' && this.state.status !== 'proposed') {
            throw new Error(`Cannot book campaign in status: ${this.state.status}`);
        }
        if (this.state.showTargets.length === 0) {
            throw new Error('Campaign must target at least one show');
        }
        if (!this.state.creative) {
            throw new Error('Campaign must have a creative before booking');
        }

        this.raise({ type: 'CampaignBooked', approvedBy });
    }

    // Command: Activate (when flight starts)
    activate(): void {
        if (this.state.status !== 'booked') {
            throw new Error(`Cannot activate campaign in status: ${this.state.status}`);
        }
        this.raise({ type: 'CampaignActivated' });
    }

    // Apply event to state (used during rehydration and after raising)
    private apply(event: any): void {
        switch (event.type) {
            case 'CampaignCreated':
                this.state = {
                    ...this.state,
                    status: 'draft',
                    networkId: event.networkId,
                    advertiserId: event.advertiserId,
                    pricingModel: event.pricingModel,
                    bookedImpressions: event.bookedImpressions,
                    flightStart: event.flightStart,
                    flightEnd: event.flightEnd,
                };
                break;
            case 'CampaignShowTargeted':
                this.state.showTargets.push({
                    showId: event.showId,
                    adPosition: event.adPosition,
                });
                break;
            case 'CampaignBooked':
                this.state.status = 'booked';
                break;
            case 'CampaignActivated':
                this.state.status = 'active';
                break;
            case 'CampaignCompleted':
                this.state.status = 'completed';
                break;
            case 'CampaignCancelled':
                this.state.status = 'cancelled';
                break;
        }
    }

    private raise(event: any): void {
        this.apply(event);
        this.uncommittedEvents.push(event);
    }
}
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by construction**: Every state change is an immutable event. For royalty disputes ("Why did my payout change?"), the full history of rule changes, revenue entries, and calculation adjustments is preserved without separate audit tables. This is significantly stronger than trigger-based audit logging in a CRUD model.

2. **Temporal queries are natural**: "What were the active royalty splits for Show X on February 14?" is answered by replaying events up to that date. Campaign reconciliation disputes can be resolved by reconstructing the exact state at any point in time.

3. **Decoupled read and write models**: Write performance (event appending) is independent of read performance (projection queries). The campaign delivery dashboard can be served from a denormalized read model without impacting the event store's write throughput.

4. **Resilient integration**: When a hosting platform API sync fails, the events already captured are preserved. When the projection handler resumes, it processes events from its checkpoint without data loss. This is critical for aggregating analytics from 5+ platforms with varying reliability.

5. **Multiple projections from the same events**: The same `AdImpressionServed` event feeds the campaign delivery projection, the show analytics projection, the network P&L projection, and the IAB compliance report. Adding a new reporting dimension (e.g., geographic breakdown) means adding a new projection handler, not changing the write model.

6. **Event replay for bug fixes**: If a projection handler has a bug in royalty calculation logic, fixing the handler and replaying events from the event store regenerates corrected read models without data loss.

7. **Natural fit for Kafka integration**: High-volume impression events flow through Kafka with guaranteed ordering per partition (partitioned by campaignId), enabling horizontal scaling of impression processing without changing the core event model.

### Cons

1. **Eventual consistency in read models**: After a campaign is booked (event written), the campaign dashboard (read model) may not reflect the change for milliseconds to seconds. For real-time dashboards, this is usually acceptable; for payout approval workflows where a user expects immediate feedback, it requires careful UX handling (optimistic updates, loading states).

2. **Significant implementation complexity**: Event sourcing requires building event store infrastructure, projection handlers, snapshot management, dead letter queues, and checkpoint tracking. The team must understand aggregate design, event versioning, and idempotent projections. This is substantially more complex than a CRUD repository pattern.

3. **Event schema evolution is hard**: When the `CampaignCreated` event needs a new field (e.g., `brandSafetyTags`), old events in the store don't have it. The system must handle upcasting (transforming old event shapes to new ones during replay) or maintain backward-compatible readers. Over years of operation, event schema debt accumulates.

4. **Debugging difficulty**: When a read model shows incorrect data, the root cause could be in the event store (wrong event written), the projection handler (wrong logic), or the checkpoint (events skipped). Tracing through event chains is harder than inspecting a single row in a CRUD database.

5. **Storage growth**: The event store is append-only and grows indefinitely. A network serving 50M impressions/month produces 50M+ events per month in the impression stream alone. Snapshots reduce replay time but don't reduce storage. Event archiving (moving old partitions to cold storage) is necessary but adds operational complexity.

6. **Projection rebuild time**: If a projection needs to be rebuilt from scratch (due to a bug fix or schema change), replaying millions of events takes time. For the impression projection with hundreds of millions of events, a full rebuild could take hours or days unless parallelized.

7. **Testing complexity**: Unit testing aggregate behavior requires setting up event streams. Integration testing projections requires producing events and verifying read models. The test infrastructure is substantially heavier than testing CRUD operations.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Event store | PostgreSQL 16+ with JSONB payloads | ACID guarantees, familiar ops, partitioning for retention |
| Message broker | Apache Kafka (Confluent Cloud or self-hosted) | Guaranteed ordering, horizontal scaling for impression volume |
| Read model DB | PostgreSQL (same cluster or separate) | Consistent technology, relational queries for projections |
| Real-time cache | Redis 7+ | IAB dedup windows (24hr TTL), real-time dashboard counters |
| Event bus (internal) | In-process (for low-volume domains) or Kafka (for high-volume) | Avoid over-engineering low-volume domains |
| Snapshot serialization | JSONB | Same format as events, queryable for debugging |
| Monitoring | OpenTelemetry + Grafana | Trace event processing latency, projection lag |
| Framework | Axon Framework (Java/Kotlin) or custom (TypeScript/Node.js) | Axon provides event store + saga + projection abstractions |

---

## Migration and Scaling Considerations

### Initial Deployment

Start with PostgreSQL for both event store and read models. Use in-process event bus for low-volume domains (shows, campaigns, royalties). Use Kafka only for impression events. A single PostgreSQL instance handles the event store, read models, and all projections.

### Scaling Event Store

- **Partition management**: Monthly partitions with pg_partman. Archive partitions older than 12 months to S3 as Parquet files. Maintain index-only partitions for the most recent 3 months.
- **Snapshot frequency**: Create snapshots every 100 events per aggregate. Campaign aggregates with frequent impression-related events benefit most from snapshotting.
- **Connection management**: Projection handlers should use connection pooling to avoid exhausting connections during high-throughput event processing.

### Scaling Projections

- **Parallel projection processing**: Different projections can process events independently. The campaign delivery projection and the analytics projection can run on separate consumers.
- **Projection partitioning**: The impression projection can be partitioned by campaign ID, allowing multiple instances to process impressions for different campaigns concurrently.
- **Eventual consistency SLA**: Monitor projection lag (difference between latest event timestamp and latest processed timestamp per projection). Alert if lag exceeds 30 seconds for financial projections or 5 minutes for analytics projections.

### Scaling Kafka

- **Partition count**: Start with 32 partitions for the impression topic. Increase as throughput grows.
- **Consumer groups**: One consumer group per projection. Each projection independently tracks its offset.
- **Retention**: 7 days on Kafka (events are persisted in the event store; Kafka is for distribution, not permanent storage).

### Migration from CRUD

If starting with the normalized PostgreSQL model (Suggestion 1) and later migrating to event sourcing:
1. Implement dual-write: write to both CRUD tables and event store during transition
2. Build projections that produce read models identical to existing CRUD tables
3. Validate projection output against CRUD data for 30 days
4. Switch reads to projection-based read models
5. Retire CRUD write path
6. This migration is complex and typically takes 3-6 months for a team of 3-5 engineers
