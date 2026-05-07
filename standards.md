# Standards & API Reference

> Project: Podcast Network Management · Generated: 2026-05-07

## Industry Standards & Specifications

### IAB Standards

**IAB Podcast Measurement Technical Guidelines v2.2 (2024)**
- URL: https://iabtechlab.com/standards/podcast-measurement-guidelines/
- PDF: https://iabtechlab.com/wp-content/uploads/2024/02/PodcastMeasurement_v2.2_pc.pdf
- The primary standard for counting podcast downloads and ad delivery. Defines deduplication rules (unique IP + User Agent within a 24-hour window), explicit filters for duplicate Apple Watch downloads, ad delivery validity rules (ad must be part of a valid download), and methodology documentation requirements. Compliance requires a third-party IAB Tech Lab certification audit. Any podcast network management platform producing advertiser-facing reports must implement v2.2 deduplication to be credible with agencies.

**IAB Podcast Measurement Technical Guidelines v2.1 (2021)**
- URL: https://iabtechlab.com/wp-content/uploads/2021/03/PodcastMeasurement_v2.1.pdf
- The predecessor to v2.2; still widely implemented by certified hosting platforms (Libsyn, Podbean, Buzzsprout). Defines the 5-step server-log processing methodology, 60-second play threshold for valid listen counting, and IP/User Agent uniqueness rules. Any normalisation layer aggregating analytics from mixed hosting environments must handle platforms certified at v2.1 as well as v2.2.

**IAB Digital Audio Measurement Guide (2022)**
- URL: https://www.iab.com/wp-content/uploads/2022/05/IAB_Audio_Measurement_Guide_Final.pdf
- Broader guidance covering streaming audio, podcast downloads, and dynamic ad delivery measurement in a unified framework. Relevant for any analytics normalisation layer consuming data from both streaming (Spotify, Apple) and download-based metrics.

**IAB OpenRTB Specification v2.6**
- URL: https://github.com/InteractiveAdvertisingBureau/openrtb2.x/blob/main/2.6.md
- Also: https://iabtechlab.com/standards/openrtb/
- The programmatic advertising protocol used by SSPs and DSPs in the digital audio ecosystem. OpenRTB 2.4 introduced a standalone Audio object with Feed type (Music Service / FM-AM Broadcast / Podcast), Stitched flag, and Volume Normalization. v2.6 clarifies podding rules (multiple ads in a single break) and pricing controls. Required for any platform that exposes podcast ad inventory to programmatic buyers or connects to an exchange like AdsWizz or TritonDigital.

**IAB Video Ad Serving Template (VAST) v4.1+**
- URL: https://www.iab.com/guidelines/vast/
- GitHub: https://github.com/InteractiveAdvertisingBureau/vast
- The ad-serving template standard that supersedes DAAST (Digital Audio Ad Serving Template, now deprecated). VAST 4.1 integrated DAAST audio support, Open Measurement ad verification, and SSAI (server-side ad insertion) signalling. Podcast platforms including Megaphone, Omny Studio, and Whooshka accept VAST URLs for DAI. A podcast network management platform integrating with programmatic demand must generate or consume VAST-compliant ad responses.

**IAB Tech Lab Podcast Measurement Compliance Certification Program**
- URL: https://iabtechlab.com/wp-content/uploads/2024/05/Podcast-Compliance_2024.docx.pdf
- The audit and certification process by which hosting and analytics platforms demonstrate compliance with IAB Podcast Measurement Guidelines. Platforms seeking advertiser trust must be listed on the IAB Tech Lab certified companies register.

---

### W3C & IETF Standards

**RSS 2.0 (RSS Advisory Board)**
- URL: https://www.rssboard.org/rss-specification
- The foundational syndication format for podcast distribution. All podcast hosting platforms generate RSS 2.0 feeds; aggregators and directories consume them. A podcast network management platform must parse and optionally generate valid RSS 2.0 feeds for each show.

**RFC 4287 — The Atom Syndication Format**
- URL: https://datatracker.ietf.org/doc/html/rfc4287
- Atom is referenced in the Podcast Namespace specification as a required namespace (`xmlns:atom`) alongside iTunes and the Podcast namespace. Relevant for feed validation.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The industry-standard protocol for delegated API access. All major podcast hosting platform APIs (Podbean, Acast) use OAuth 2.0 for authentication. A network management platform aggregating data from multiple hosting APIs must implement OAuth 2.0 client credential and authorization code flows.

**RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Defines how Bearer tokens are transmitted in HTTP requests. Relevant for all hosting and analytics API integrations.

**RFC 8446 — TLS 1.3**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- All API communications must use TLS 1.3 (or minimum TLS 1.2). Buzzsprout's API documentation explicitly states all requests must be SSL only.

**OpenID Connect Core 1.0 (OIDC)**
- URL: https://openid.net/developers/how-connect-works/
- Specs: https://openid.net/developers/specs/
- Identity layer built on OAuth 2.0. Relevant for enterprise SSO integration; Libsyn supports SAML/OAuth for enterprise authentication, and a network management platform serving enterprise networks should support OIDC-based SSO.

---

### Podcast Domain Standards

**Podcast Namespace (Podcasting 2.0) — RSS Namespace Extension**
- URL: https://podcasting2.org/docs/podcast-namespace/1.0
- GitHub: https://github.com/Podcastindex-org/podcast-namespace
- An open RSS namespace extension (`xmlns:podcast="https://podcastindex.org/namespace/1.0"`) defining advanced podcast metadata: chapters, transcripts, person credits, season/episode numbering, value streaming (Lightning Network payments), funding links, soundbite markers, and more. Actively maintained through a phased development process (Phase 7 closed July 2024). Chapters, transcripts, and person credits are directly relevant to a network management platform managing show metadata and host attribution.

**PSP-1 Podcast RSS Specification (Podcast Standards Project)**
- URL: https://github.com/Podcast-Standards-Project/PSP-1-Podcast-RSS-Specification
- A consolidated specification for podcast RSS feeds that synthesises iTunes namespace, Podcast namespace, and Atom namespace requirements. Defines mandatory and recommended tags for interoperability with Apple Podcasts, Spotify, and other directories.

**Apple Podcasts Connect Specification (iTunes Namespace)**
- URL: https://podcasters.apple.com/support/823-podcast-requirements
- The de-facto baseline for podcast feed requirements, defining the `itunes:` namespace tags (title, author, category, explicit, image, episode type, duration, etc.). All podcast management platforms must produce iTunes-namespace-compliant feeds.

**Podcast Index — Chapters Specification (Simple Chapters)**
- Described within the Podcast Namespace at: https://podcasting2.org/docs/podcast-namespace/1.0
- Defines the `podcast:chapters` tag pointing to a JSON file specifying chapter start times, titles, images, and URLs. Relevant for managing chapter metadata at the episode level in a network CMS.

---

### Data Model & API Specifications

**OpenAPI Specification v3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing REST APIs. A podcast network management platform should expose its own API using an OpenAPI 3.1 document for developer adoption. Megaphone publishes its API reference via Apiary; Acast and Podbean provide REST APIs with OAuth 2.0.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Used within OpenAPI 3.1 and independently for validating RSS-derived data structures, episode metadata, campaign configuration, and analytics payloads when building a normalisation layer across heterogeneous hosting APIs.

**Podcasting 2.0 Value Tag / Lightning Network (BOLT 11/12)**
- URL: https://github.com/Podcastindex-org/podcast-namespace/blob/main/value/value.md
- Defines value-streaming micropayment tags enabling per-second crypto payments to show contributors. While niche, the value splits model is architecturally similar to the royalty-split problem the network management platform solves, and may be worth monitoring for convergence.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) — see W3C & IETF section above**

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Best-practice guidance for API security covering broken object-level authorisation, authentication weaknesses, and mass assignment risks. Directly applicable to a platform managing multi-tenant network data, royalty calculations, and financial disbursements.

**PCI DSS v4.0 (Payment Card Industry Data Security Standard)**
- URL: https://www.pcisecuritystandards.org/document_library/
- Relevant where the platform processes or facilitates card payments for advertiser billing or listener subscriptions. Delegating payment processing to Stripe or Tipalti (both PCI-compliant) reduces scope, but the platform must understand its own cardholder data environment obligations.

**GDPR (EU Regulation 2016/679) and equivalent privacy frameworks**
- URL: https://gdpr-info.eu/
- Podcast listener analytics involve IP address processing; platforms targeting EU audiences must implement appropriate lawful basis, data minimisation (IP truncation per IAB v2.2), data retention limits, and processor agreements with hosting platforms and attribution vendors. Podscribe's Roqad integration explicitly addresses GDPR-compliant identity resolution.

**NIST SP 800-63B — Digital Identity Guidelines (Authentication)**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- Guidance for authentication assurance levels. Relevant for enterprise network deployments requiring MFA and SSO, consistent with Libsyn's enterprise SSO requirements.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant to an AI-native podcast network management platform for exposing show catalogue, campaign, and royalty data to LLM-powered tooling.

**Model Context Protocol — Official Specification**
- URL: https://modelcontextprotocol.io/specification
- Defines how AI agents (Claude, Cursor, etc.) can connect to structured data sources via MCP servers. A podcast network management MCP server could expose tools for: querying show and episode metadata, retrieving royalty calculations, generating campaign delivery reports, and fetching cross-platform analytics summaries.

**MCP Server SDK (TypeScript / Python)**
- URL: https://github.com/modelcontextprotocol/typescript-sdk
- URL: https://github.com/modelcontextprotocol/python-sdk
- Official SDKs for building MCP servers. An MCP server built on top of the network management platform's API would allow network operators to use Claude or other AI assistants for natural-language reporting and workflow automation without leaving their AI tooling.

---

## Similar Products — Developer Documentation & APIs

### Megaphone

- **Description:** Spotify's enterprise podcast hosting and dynamic ad insertion platform, targeting large publishers and networks. Provides multi-show management, DAI, and access to the Spotify Audience Network.
- **API Documentation:** https://developers.megaphone.fm/
- **Additional Reference:** https://jsapi.apiary.io/apis/megaphoneapi/reference/podcasts.html
- **SDKs/Libraries:** No official SDK. Unofficial Python client: https://github.com/theatlantic/megaphone
- **Developer Guide:** https://support.megaphone.fm/en/articles/2247461-megaphone-api
- **Standards:** REST / JSON; VAST 4.1+ for ad serving
- **Authentication:** API token (enterprise accounts only); token passed in HTTP Authorization header

### Acast

- **Description:** Global podcast hosting and monetisation platform with a combined ad marketplace and publisher analytics dashboard. Supports subscriptions, sponsorships, and programmatic advertising.
- **API Documentation:** https://developers.acast.com/
- **Developer Guide:** https://learn.acast.com/en/articles/5790019-acast-publishing-api
- **Webhooks:** https://learn.acast.com/en/articles/3505461-what-is-a-webhook
- **SDKs/Libraries:** No official SDK. Unofficial client: https://github.com/darbymanning/acastaway
- **Standards:** REST / JSON; OpenAPI-style (documentation not published as machine-readable spec)
- **Authentication:** OAuth 2.0 (API keys set at user level)

### Libsyn

- **Description:** One of the longest-running podcast hosting platforms offering enterprise network management, IAB-certified analytics, ad insertion, and the Libsyn Audience Network.
- **API Documentation:** https://libsyn.com/ (enterprise API access; full docs require account)
- **API Tracker Reference:** https://apitracker.io/a/libsyn
- **Developer Guide:** https://libsyn.com/libsyn-podcast-hosting-features/
- **Standards:** REST / JSON
- **Authentication:** API key; enterprise plans support SAML SSO

### Podbean

- **Description:** Mid-market podcast hosting platform with network management, IAB-certified analytics, PodAds dynamic ad insertion, and an Ads Marketplace for sponsorship matching.
- **API Documentation:** https://developers.podbean.com/podbean-api-docs/
- **Developer Guide:** https://help.podbean.com/support/solutions/articles/25000008051-publishing-a-new-podcast-episode-via-podbean-api
- **Integration Guide:** https://help.podbean.com/support/solutions/articles/25000010443-integrating-your-service-with-podbean-platform
- **Standards:** REST / JSON
- **Authentication:** OAuth 2.0

### Buzzsprout

- **Description:** Consumer-friendly podcast hosting platform with IAB-certified analytics, AI audio processing, and Buzzsprout Ads marketplace. Transparent pricing and strong developer documentation.
- **API Documentation:** https://www.buzzsprout.com/api
- **GitHub:** https://github.com/buzzsprout/buzzsprout-api
- **SDKs/Libraries:** No official SDK. Airbyte source connector: https://docs.airbyte.com/integrations/sources/buzzsprout
- **Standards:** REST / JSON; all endpoints return `.json`; UTF-8 encoded
- **Authentication:** Token-based HTTP Authentication (`Authorization: Token token=<API_KEY>`)

### RedCircle

- **Description:** Free hosting and advertising platform with a self-serve ad marketplace, host-read ad automation, and the OpenRAP open programmatic API for cross-platform ad demand.
- **API Documentation:** https://redcircle.com/features (OpenRAP documented via Simplecast partnership)
- **OpenRAP Reference:** https://blog.simplecast.com/increase-podcast-ad-revenue-with-simplecast-professional-and-redcircles-openrap
- **Standards:** REST / JSON; OpenRAP uses standard ad-tech protocols
- **Authentication:** API key (advertiser portal); OAuth for creator accounts

### AdsWizz

- **Description:** End-to-end digital audio ad technology platform providing DAI, programmatic SSP/DSP integration, brand safety, and yield optimisation for publishers and advertisers.
- **API Documentation:** https://www.adswizz.com/ (requires partner agreement; public API reference not published)
- **Developer Reference:** https://rainnews.com/adswizz-launches-new-podcast-ad-effectiveness-features/
- **Standards:** OpenRTB 2.6 (programmatic); VAST 4.1+ (ad serving); REST API for campaign management
- **Authentication:** Partner-credentialed API key; OAuth 2.0 for publisher integrations

### Stripe Connect

- **Description:** Stripe's platform for marketplaces and multi-party payment flows. Relevant for managing host royalty disbursements across a podcast network with multi-currency international payouts.
- **API Documentation:** https://docs.stripe.com/connect
- **Payout API:** https://docs.stripe.com/api/payouts
- **Connected Accounts:** https://docs.stripe.com/connect/payouts-connected-accounts
- **SDKs/Libraries:** stripe-node, stripe-python, stripe-go, stripe-java, stripe-dotnet (all on GitHub at https://github.com/stripe)
- **Developer Guide:** https://docs.stripe.com/connect/marketplace/tasks/payout
- **Standards:** REST / JSON; OpenAPI 3.0 specification published
- **Authentication:** API secret key + restricted keys; webhook signatures for event verification

### Tipalti

- **Description:** Automated global payables and mass payout platform used for royalty and revenue-share disbursements to creators and contributors in 200+ countries, 120 currencies, 50+ payment methods. KPMG-approved tax engine for W-9/W-8/VAT/DAC7.
- **API Documentation:** https://tipalti.com/mass-payments/payout-api/
- **Developer Guide:** https://tipalti.com/blog/marketplace-payouts-api/
- **Royalty Use Case:** https://tipalti.com/industries/music-royalties/
- **Standards:** REST / JSON; SOAP API also available for legacy integrations
- **Authentication:** API key + HMAC signature for request verification; sandbox environment for testing

### Podscribe

- **Description:** Third-party podcast attribution and measurement platform. Provides multi-touch attribution, incrementality testing, AI-powered brand safety airchecks, and global identity graph integration.
- **API Documentation:** https://podscribe.com/ (publisher and advertiser portals; API access requires account)
- **Help Centre:** https://help.podscribe.com/
- **Standards:** Prefix redirect (standard IAB-compatible tracking URL); REST API; pixel-based web conversion tracking
- **Authentication:** API key (account-level)

---

## Notes

**Emerging and evolving areas:**

1. **IAB Podcast Measurement v3.0**: The IAB Tech Lab Podcast Technical Working Group has signalled future work on streaming metrics (Apple HLS video podcasts introduced in 2026), which will require extending the current download-centric v2.2 framework to address buffered streaming, view-through events, and video completion rates. Any platform built today should design its analytics model to accommodate streaming signals alongside download events.

2. **Podcasting 2.0 value streaming**: The `podcast:value` tag enabling Lightning Network micropayments per second of audio consumed is architecturally analogous to a royalty-split system. As Lightning Network adoption grows, a network management platform may want to natively resolve Podcasting 2.0 value splits to contributor wallets rather than treating them as a separate payment rail.

3. **Podcast namespace chapters and AI insertion points**: The `podcast:chapters` JSON format, combined with transcript data and listener engagement drop-off curves, represents a convergence point where AI can recommend DAI marker placement. The standard is stable enough to build against now.

4. **Cross-platform identity and attribution post-cookie**: With Chartable sunset (December 2025) and increasing restrictions on user-level tracking, prefix-redirect-based attribution and household-level modelling (Podscribe's approach) are becoming the dominant methodology. Platforms should design their analytics normalisation layer around IP-based deduplication (IAB v2.2) rather than cookie-based identity.

5. **OpenRTB 3.0 / AdCOM**: IAB Tech Lab has published OpenRTB 3.0 and AdCOM (Advertising Common Object Model) as the next-generation programmatic standard, decomposing the bid request into reusable objects. Adoption in audio/podcast is currently limited; OpenRTB 2.6 remains the operative standard in 2026, but long-lived platform architectures should plan for eventual migration.
