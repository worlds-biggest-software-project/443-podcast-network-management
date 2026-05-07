# Podcast Network Management — Feature & Functionality Survey

> Candidate #443 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Megaphone (Spotify) | Hosting + DAI + Ad Network | Commercial / SaaS (enterprise) | https://megaphone.fm/ |
| AdsWizz | Ad tech / DAI / programmatic | Commercial / SaaS | https://www.adswizz.com/ |
| Acast | Hosting + Ad Marketplace | Commercial / SaaS | https://www.acast.com/ |
| Libsyn | Hosting + Monetisation | Commercial / SaaS | https://libsyn.com/ |
| Podbean / PodAds | Hosting + DAI | Commercial / SaaS | https://www.podbean.com/ |
| RedCircle | Hosting + Ad Platform | Commercial / SaaS (free tier) | https://redcircle.com/ |
| Buzzsprout | Hosting + Analytics | Commercial / SaaS | https://www.buzzsprout.com/ |
| CoHost | Analytics + Hosting | Commercial / SaaS | https://www.cohostpodcasting.com/ |
| Podscribe | Attribution + Measurement | Commercial / SaaS | https://podscribe.com/ |
| Podtrac | Audience Measurement | Commercial (free for publishers) | https://podtrac.com/ |

---

## Feature Analysis by Solution

### Megaphone (Spotify)

**Core features**
- Enterprise multi-show hosting with role-based team access
- Dynamic ad insertion (DAI) for pre-roll, mid-roll, and post-roll with flexible ad point management
- Back-catalogue ad monetisation via DAI
- Access to the Spotify Audience Network for programmatic inventory
- Metrics Export Service and API access for enterprise subscribers
- Custom media players and embeddable players
- Analytics dashboard with download statistics and engagement metrics
- 24/7 support and dedicated Technical Account Manager for enterprise plans

**Differentiating features**
- Deep integration with Spotify's listener identity graph for demographic targeting
- Spotify Audience Network reach extends monetisation beyond the host's own audience
- VAST URL acceptance enabling programmatic ad insertion via external ad servers

**UX patterns**
- Web-based CMS with one-click publishing and distribution
- Separate enterprise and self-serve tiers with different feature gates
- Complex features (API, Metrics Export) locked behind enterprise tier

**Integration points**
- REST API at `developers.megaphone.fm` for shows, episodes, and campaign management
- Spotify Audience Network programmatic pipeline
- VAST-compatible ad serving for third-party buyers
- Apple Podcasts, Spotify, and major directory distribution

**Known gaps**
- Host royalty splitting and payment disbursement not natively supported
- Cross-platform analytics (shows hosted elsewhere) not available
- No direct competitor ad clashing / brand-safety controls outside Spotify's own pipeline
- Pricing opaque for mid-tier networks below enterprise threshold

**Licence / IP notes**
- Proprietary SaaS; no open-source components published. Spotify owns the platform following 2020 acquisition.

---

### AdsWizz

**Core features**
- Real-time dynamic ad insertion and ad replacement for live streams and downloads
- Programmatic selling platform (SSP/DSP integration via OpenRTB)
- Contextual targeting by content category, geography, time of day, and weather
- Competitive ad clashing prevention within a single episode
- Streaming vs. download delivery reporting
- AI-powered transcription for content-based targeting and brand safety
- Campaign forecasting and yield optimisation tools
- Offline ad delivery reporting

**Differentiating features**
- Brand-safe inventory certification using AI transcription and third-party verification
- Integration of Simplecast CMS as a distribution companion
- Publisher-agnostic: works across hosting platforms via redirect/pixel prefix

**UX patterns**
- Separate publisher (supply) and advertiser (demand) portals
- Heavy use of targeting rule-builders for campaign configuration
- Reporting exposed via dashboards and CSV export

**Integration points**
- OpenRTB bid request/response for programmatic demand
- VAST 4.1+ for ad serving
- Simplecast CMS integration
- Multiple podcast hosting platforms via prefix redirect

**Known gaps**
- Royalty/revenue-share management not in scope
- Subscription/membership management absent
- Cross-network unified reporting requires custom integration
- Steep learning curve for smaller independent networks

**Licence / IP notes**
- Proprietary SaaS, owned by Pandora/SiriusXM. No open-source components.

---

### Acast

**Core features**
- All-in-one CMS for episode management, scheduling, and one-click distribution
- Dynamic ad insertion across entire back catalogue
- Multi-format monetisation: ads, sponsorships, branded content, memberships, and subscriptions
- Marketplace matching algorithm pairing shows with advertisers at 1,000+ monthly listeners
- Real-time 360-degree analytics dashboard (audience and revenue)
- Conversion-based campaign measurement (not just impressions)
- Programmatic ad delivery with contextual targeting and brand safety
- Webhook support for event-driven integrations

**Differentiating features**
- Conversion-level attribution (not just impression/download metrics)
- Combined editorial and commercial marketplace in one platform
- Acast+ membership and subscription tools for listener revenue

**UX patterns**
- Unified publisher dashboard covering publishing, analytics, and monetisation
- Marketplace approval flow with eligibility thresholds
- Self-serve for smaller shows; dedicated sales team for larger accounts

**Integration points**
- Publishing API at `developers.acast.com` with OAuth-secured show and episode endpoints
- Webhooks on episode publish events
- Distribution to all major directories
- Podscribe partnership for global attribution measurement

**Known gaps**
- Host royalty splitting and multi-party disbursement not present
- Cross-platform analytics limited to Acast-hosted shows
- Network-level reporting (aggregated across multiple shows) available only at the account level
- No OpenRTB-native programmatic for publishers on free tier

**Licence / IP notes**
- Proprietary SaaS. Swedish-founded, publicly listed company.

---

### Libsyn

**Core features**
- Multi-show network hosting with show-level and network-level RSS feeds
- Role-based access control (owner, admin, channel-admin, analyst, contributor) per show or network-wide
- IAB 2.1-certified download analytics
- Geographic performance data, episode-level stats, and YouTube reporting in one dashboard
- Programmatic and dynamic ad insertion campaign support
- Libsyn Audience Network for additional ad inventory
- Apple Podcasts Subscriptions integration for premium/subscriber content
- Podroll cross-promotion monetisation
- Dedicated ad sales team for larger shows
- Hands-on onboarding, show imports, and phone support for enterprise

**Differentiating features**
- One of the oldest platforms (est. 2004) with deep directory relationships
- Libsyn Audience Network launches (2025-2026) for additional programmatic reach
- Per-show or network-wide analytics view with export capabilities

**UX patterns**
- Enterprise-grade administration console with SSO support
- Dedicated account manager and phone support
- Tiered plan structure from individual to enterprise network

**Integration points**
- API available with developer tools for custom integrations (`libsyn.com`)
- Apple Podcasts Subscriptions API
- Directory publishing APIs
- SSO (SAML/OAuth) for enterprise authentication

**Known gaps**
- Host royalty splitting automation not built in; relies on manual processes or external tools
- Real-time bidding SSP exposure limited compared to AdsWizz
- UI is dated and less intuitive than newer competitors
- Cross-platform analytics (shows hosted elsewhere) not supported

**Licence / IP notes**
- Proprietary SaaS. Publicly traded (LSYN on Nasdaq). No open-source components.

---

### Podbean / PodAds

**Core features**
- Network plan supporting multiple shows under one account with network-level analytics
- Team-based role access: owner, admins, channel-admins, analysts, contributors
- Customisable network page to showcase all shows under one brand
- IAB-certified download statistics (network-level and per-show)
- Second-by-second listener engagement reporting (User Engagement Intel)
- PodAds dynamic ad insertion (pre-roll, mid-roll, post-roll) with geo-targeting
- Back-catalogue ad replacement without re-editing audio files
- Ads Marketplace connecting shows with advertisers; ~10% commission on sponsorships
- Subscription and premium content management
- International payment processing for sponsor payments

**Differentiating features**
- User Engagement Intel: second-by-second listening consumption data
- Network-branded page for aggregated show promotion
- Combined hosting + DAI + marketplace in a single mid-market product

**UX patterns**
- Single dashboard for hosting, analytics, and monetisation
- Marketplace approval flow for ad matching
- Network analytics toggle between individual show and aggregate view

**Integration points**
- OAuth 2.0-secured REST API at `developers.podbean.com`
- Apple Podcasts and Spotify distribution
- International payment processing integrations
- App Market for third-party service connections

**Known gaps**
- No advanced attribution (funnel tracking, individual listener journeys, paid ad attribution)
- Host royalty splitting not available; only show-level earnings
- Programmatic RTB integration limited compared to AdsWizz or Megaphone
- No SSO for enterprise authentication

**Licence / IP notes**
- Proprietary SaaS. Independent company. No open-source components.

---

### RedCircle

**Core features**
- Free hosting with monetisation-first positioning
- RedCircle Ad Platform (RAP): host-read and programmatic ad marketplace
- Campaign setup with targeting by category, audience size, listener demographics, and CPM
- Advertiser budget/flight date controls and episode-level targeting (new episodes or all episodes)
- Real-time campaign monitoring: budget tracking and performance across shows
- A/B testing of ad formats, messaging, and show placement
- Ad read review and feedback workflow between advertiser and host
- OpenRAP: open programmatic integration allowing Simplecast Professional shows to access RedCircle's ad demand

**Differentiating features**
- First automated podcast advertising platform for scaling host-read ads across thousands of shows
- OpenRAP interoperability enables cross-platform ad demand access
- Launch-in-days campaign setup compared to traditional weeks-long direct-sold process

**UX patterns**
- Streamlined self-serve advertiser portal
- Creator-facing dashboard with campaign tracking
- Budget-first campaign builder with show selection from metrics

**Integration points**
- OpenRAP API for cross-platform programmatic access (Simplecast integration)
- RSS feed distribution
- Stripe Connect for payment disbursement to creators

**Known gaps**
- Analytics depth limited vs. enterprise platforms
- Host royalty splitting (multi-contributor per show) not addressed
- Network-level aggregated reporting not available
- Enterprise features (SSO, dedicated support) absent

**Licence / IP notes**
- Proprietary SaaS. No open-source components disclosed.

---

### Buzzsprout

**Core features**
- Episode hosting with upload-hours-based pricing model
- IAB-certified analytics (downloads, listener apps, device types, geographic data, episode trends)
- One-click distribution to Apple Podcasts, Spotify, YouTube, and 10+ directories
- Magic Mastering: automated audio enhancement and file optimisation
- Power Clean: AI background noise removal
- Filler Killer: AI filler word removal
- Cohost AI: auto-generated episode titles, descriptions, chapter markers, and transcripts
- Three monetisation paths: Listener Support, Premium Content, and Buzzsprout Ads
- Unlimited team members on all paid plans

**Differentiating features**
- AI-powered audio post-processing pipeline (Mastering, Power Clean, Filler Killer) built into hosting
- Cohost AI for automated episode metadata generation
- Transparent pricing (published rates; no enterprise-only tiers for core features)

**UX patterns**
- Consumer-grade, approachable interface for independent podcasters
- AI tools surfaced as optional post-processing steps per episode upload
- Annual plans offered at discount vs. monthly

**Integration points**
- REST API at `buzzsprout.com/api` with token-based authentication; JSON throughout
- Official GitHub documentation at `github.com/buzzsprout/buzzsprout-api`
- Airbyte connector for analytics data pipeline
- Apple Podcasts Connect, Spotify for Podcasters, YouTube distribution

**Known gaps**
- No multi-show network management (no network page, no aggregated network analytics)
- No dynamic ad insertion; ad monetisation limited to the Buzzsprout Ads marketplace
- No host royalty splitting
- Per-hour upload cap limits high-volume network use
- No programmatic or OpenRTB integration

**Licence / IP notes**
- Proprietary SaaS. Independent company.

---

### CoHost

**Core features**
- Podcast hosting with one-click publishing and multi-show management
- Advanced audience demographics: age, income, family, lifestyle, hobbies, and social media habits
- B2B analytics: listener company, industry, size, location, revenue, and job role/seniority
- Consumption insights: new listener growth, listener sources, report-ready data
- AI-powered automatic transcription
- Custom media players
- Multiple user permissions and multi-show management
- Analytics Prefix: works with any hosting provider without migrating (prefix redirect)

**Differentiating features**
- B2B firmographic listener insights: unique in the market for identifying the professional profiles of listeners
- Analytics as a standalone layer (Prefix product) usable independently of hosting

**UX patterns**
- Agency and producer-oriented multi-account switcher (no sign-out required between client accounts)
- Report-ready data export for advertiser-facing presentations
- Separate hosting and prefix products allow incremental adoption

**Integration points**
- Prefix redirect compatible with any hosting platform
- REST API for show and episode management
- Publisher directory integrations (Apple, Spotify, etc.)

**Known gaps**
- Ad campaign management and DAI absent
- Host royalty splitting not supported
- Programmatic ad inventory management not in scope
- Subscription/membership management not available

**Licence / IP notes**
- Proprietary SaaS. Canadian company. No open-source components.

---

### Podscribe

**Core features**
- Pixel-based multi-touch attribution tracking exposure-to-conversion at household level
- Incrementality testing to prove causal lift from podcast ad exposure
- AI transcription of episodes for brand safety checks (18+ automated aircheck verifications)
- ChatGPT-powered brand safety monitoring and sponsorship trend analysis
- YouTube SmartModeling attribution framework
- Cross-channel insights combining podcast with other media
- Global attribution via identity graph integration (including Roqad for EU GDPR-compliant attribution)
- Publisher analytics: impressions, delivery verification, engagement data

**Differentiating features**
- Incrementality testing: the only common third-party tool measuring true causal ad lift
- AI-powered airchecks: automated verification of ad read quality, duration, and talking points
- B2B firmographic attribution available alongside consumer demographics

**UX patterns**
- Advertiser-facing dashboard with campaign ROI summaries
- Publisher portal for delivery verification and brand safety reporting
- Automated aircheck alerts for non-compliant ad reads

**Integration points**
- Prefix redirect for download measurement (works with any host)
- Identity graph partnerships (Roqad, LiveRamp)
- Acast preferred global attribution partner
- Pixel-based web conversion tracking

**Known gaps**
- Does not manage hosting, publishing, or DAI
- No royalty or payment management
- Not a campaign management tool; analytics and measurement only
- Pricing not publicly disclosed

**Licence / IP notes**
- Proprietary SaaS. US company. No open-source components.

---

### Podtrac

**Core features**
- Third-party audience measurement with IAB certification (first IAB-certified podcast measurement service)
- Unique monthly audience, episode downloads, and platform-specific performance
- Listener demographics: age, location, and preferences
- Promo Marketing Attribution: tracks new audience growth from audio promos and custom links
- Industry ranking reports (Top Podcasts, Top Publishers)
- Processes over 250 million unique downloads per month

**Differentiating features**
- Industry-standard ranking data used by publishers and advertisers as a credibility benchmark
- First IAB-certified third-party measurement — widely trusted for advertiser reporting

**UX patterns**
- Publisher self-registration via prefix redirect
- Ranking reports published publicly and used in media kits

**Integration points**
- Prefix redirect for all hosting platforms
- No proprietary hosting; pure measurement overlay

**Known gaps**
- No ad management, DAI, royalty, or subscription features
- Limited to download and reach metrics; no attribution or conversion tracking
- No real-time dashboards; reports based on rolling windows

**Licence / IP notes**
- Proprietary SaaS. Free for publishers; revenue from advertiser/agency licensing of data.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Episode hosting with RSS feed generation and multi-directory distribution
- IAB-certified download analytics (v2.1 or v2.2) at show and episode level
- Role-based user access (at minimum: owner, admin, contributor)
- Dynamic ad insertion with pre-roll, mid-roll, and post-roll placement
- Basic campaign management: flight dates, CPM or flat rate, impression targets
- Distribution to Apple Podcasts, Spotify, and major directories

### Differentiating Features
- B2B firmographic listener analytics (CoHost)
- Multi-touch attribution with incrementality testing (Podscribe)
- AI-powered audio post-processing (Buzzsprout)
- Automated AI airchecks for ad delivery verification (Podscribe)
- OpenRAP / cross-platform programmatic demand (RedCircle)
- Second-by-second engagement intel (Podbean)
- Spotify identity graph targeting (Megaphone)
- Brand-safe inventory certification via AI transcription (AdsWizz)

### Underserved Areas / Opportunities
- **Host royalty splitting**: No major hosting platform automates multi-contributor revenue-share calculations and payment disbursements. Networks manage this manually in spreadsheets.
- **Cross-platform network analytics**: Networks with shows on multiple hosts (Megaphone, Libsyn, Acast) have no single pane of glass aggregating analytics across all hosting environments.
- **Unified campaign management across programmatic + direct-sold**: Networks selling both programmatic and direct-sold inventory across multiple shows lack a reconciliation layer.
- **Advertiser deal management**: Deal lifecycle (proposal, insertion order, flight management, delivery reconciliation, invoicing) is fragmented across email, spreadsheets, and ad platforms.
- **P&L reporting per show**: No tool produces a network-level P&L combining ad revenue, subscription revenue, hosting costs, and host payout obligations.
- **Multi-currency international disbursement**: Cross-border host payments with tax documentation (W-9/W-8, VAT) require custom integration with Stripe Connect or Tipalti; no podcast-native solution exists.
- **AI-assisted sponsorship prospecting**: Matching advertiser brand profiles against show content and audience demographics for optimal campaign fit is largely manual.

### AI-Augmentation Candidates
- Automated host royalty calculation from ad revenue, subscription revenue, and streaming data
- AI-suggested ad insertion points based on transcript and engagement drop-off analysis
- Automated advertiser-host matching using content embeddings and audience profile similarity
- Anomaly detection in download data for fraud / bot traffic identification
- Automated advertiser delivery reports generated from campaign flight data
- Transcript-based brand safety pre-screening before episode publication
- AI-generated show and episode metadata for SEO and directory optimisation

---

## Legal & IP Summary

All ten solutions surveyed are proprietary SaaS platforms. No open-source licences or published patents relevant to core functionality were identified in publicly available sources. The Podcast Namespace (podcasting2.org, Podcastindex-org/podcast-namespace on GitHub) is published under a permissive open licence and may be freely adopted. The IAB Podcast Measurement Technical Guidelines (v2.1, v2.2) are publicly available standards; implementing them does not require a licence fee, though IAB Tech Lab certification requires passing a third-party audit. OpenRTB specifications (IAB Tech Lab, GitHub: InteractiveAdvertisingBureau/openrtb) are published under open terms. No specific patent filings for podcast-specific DAI, royalty splitting, or attribution workflows were found; however, AdsWizz and Megaphone may hold trade-secret advantages in their targeting and stitching implementations.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Multi-show catalogue: show registry with metadata, hosts, RSS feeds, hosting platform reference, and episode archive
- Host royalty configuration: per-show revenue-share rules (percentage splits across multiple contributors)
- Automated royalty calculation from ad campaign revenue and subscription income
- Payment disbursement integration (Stripe Connect or Tipalti API) with payout history and statements
- Network-level analytics aggregation via RSS and hosting platform APIs (Megaphone, Libsyn, Acast, Podbean, Buzzsprout)
- IAB v2.2-compliant download deduplication for credible advertiser reporting

**Should-have (v1.1)**
- Campaign management: advertiser/agency deal records, flight dates, CPM/flat-rate pricing, guaranteed vs. programmatic inventory
- Delivery reconciliation: compare booked impressions against delivered impressions per campaign
- Advertiser-facing delivery reports (PDF/CSV export)
- Network P&L dashboard: revenue by show, host payout obligations, net margin
- Multi-user role-based access (network admin, show admin, host, analyst, advertiser view)

**Nice-to-have (backlog)**
- AI-assisted advertiser-show matching based on content and audience profiles
- AI-suggested DAI marker placement from episode transcripts and engagement data
- Automated anomaly detection / bot traffic flagging in download data
- Stripe Connect / Tipalti multi-currency payout with W-9/W-8/VAT tax document collection
- Cross-show audience overlap analysis for network packaging proposals
- Subscription and membership management (ad-free feeds, bonus content, subscriber analytics)
