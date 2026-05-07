# Podcast Network Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A unified operations layer for podcast networks: multi-show catalogue management, automated host royalty splits, dynamic ad insertion, campaign reconciliation, and cross-platform analytics -- all in one open-source platform.

Podcast networks managing multiple shows are forced to stitch together three or more disconnected tools for hosting, ad management, royalty accounting, and analytics. This project provides a single network operations platform that automates host revenue-share calculations, reconciles ad delivery across programmatic and direct-sold inventory, and aggregates audience data from heterogeneous hosting environments. It is built for network operators, ad sales teams, and show producers who need a consolidated view of their business without enterprise-tier pricing.

---

## Why Podcast Network Management?

- **No incumbent automates host royalty splitting.** Every major hosting platform (Megaphone, Libsyn, Acast, Podbean, Buzzsprout) lacks native multi-contributor revenue-share calculation and payment disbursement. Networks manage this manually in spreadsheets.

- **Cross-platform analytics do not exist.** Networks with shows distributed across multiple hosts have no single pane of glass. Each platform reports only on its own inventory, leaving network operators to manually aggregate data.

- **Campaign reconciliation is fragmented.** Networks selling both programmatic and direct-sold inventory across multiple shows have no unified layer to reconcile booked impressions against delivered impressions.

- **Enterprise-grade tooling is priced out of reach.** Megaphone locks API access and metrics export behind opaque enterprise pricing. AdsWizz has a steep learning curve for smaller networks. Mid-size networks are underserved.

- **No open-source alternative exists.** All ten solutions surveyed -- Megaphone, AdsWizz, Acast, Libsyn, Podbean, RedCircle, Buzzsprout, CoHost, Podscribe, and Podtrac -- are proprietary SaaS platforms with no published open-source components.

---

## Key Features

### Show and Network Catalogue

- Multi-show registry with metadata, hosts, RSS feeds, hosting platform references, and episode archive
- Network-branded page for aggregated show promotion
- Publishing schedule tracking and episode lifecycle management
- Role-based access control: network admin, show admin, host, analyst, advertiser view

### Host Royalty Management

- Configurable per-show revenue-share rules with percentage splits across multiple contributors
- Automated royalty calculation from ad campaign revenue and subscription income
- Payment disbursement integration via Stripe Connect or Tipalti API
- Payout history, statements, and multi-currency international disbursement with tax document collection (W-9/W-8, VAT)

### Ad Insertion and Campaign Management

- Dynamic ad insertion (DAI) for pre-roll, mid-roll, and post-roll with flexible ad point management
- Back-catalogue ad monetisation via DAI without re-editing audio files
- Campaign lifecycle management: advertiser/agency deal records, flight dates, CPM or flat-rate pricing, guaranteed vs. programmatic inventory
- Delivery reconciliation comparing booked impressions against delivered impressions per campaign
- Advertiser-facing delivery reports (PDF/CSV export)

### Cross-Platform Analytics

- Unified listener metrics (downloads, unique listeners, completion rates, geographic distribution) aggregated from multiple hosting platforms via RSS and hosting APIs
- IAB v2.2-compliant download deduplication for credible advertiser reporting
- Network P&L dashboard: revenue by show, host payout obligations, net margin
- Cross-show audience overlap analysis for network packaging proposals

### Subscription and Membership

- Listener subscription management for ad-free feeds and bonus content
- Payment processing and subscriber analytics
- Premium content delivery and entitlement management

---

## AI-Native Advantage

An AI-native approach unlocks capabilities that incumbents do not offer. Automated ad insertion point suggestions based on transcript analysis and engagement drop-off patterns replace manual marker placement. Advertiser-show matching uses content embeddings and audience profile similarity to surface optimal campaign fits, replacing the largely manual sponsorship prospecting process. Anomaly detection in download data flags bot traffic and fraud for IAB-compliant reporting. Transcript-based brand safety pre-screening automatically identifies content risks before episode publication.

---

## Tech Stack and Deployment

The platform is designed for self-hosted and cloud deployment. Podcast download counting follows IAB v2.x deduplication rules for advertiser credibility. A hosting platform normalisation layer abstracts the varying APIs of Megaphone, Libsyn, Acast, Podbean, and Buzzsprout to aggregate analytics across a mixed-hosting network. Programmatic ad exchange integration uses OpenRTB-compliant bid request/response handling (IAB Tech Lab specifications, published under open terms). The Podcast Namespace (podcasting2.org) is available under a permissive open licence for RSS feed extensions. Cross-border payment disbursement integrates with Stripe Connect or Tipalti for multi-currency payouts.

---

## Market Context

Global podcast advertising spend is forecast to reach approximately USD 5.5 billion by 2026, making it one of the fastest-growing advertising channels ([Talks.co](https://talks.co/p/podcast-advertising-platforms/), [Content Allies](https://contentallies.com/learn/top-podcast-advertising-networks)). Despite this scale, the tooling for independent and mid-size networks remains fragmented across hosting, ad management, royalty accounting, and analytics. The primary buyers are professionally operated podcast networks that have outgrown single-show hosting but fall below the scale where Spotify's or iHeart's proprietary infrastructure becomes accessible.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
