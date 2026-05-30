# Podcast Network Management — Phased Development Plan

> Project: 443-podcast-network-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents into a single, phased implementation specification. The data model adopts **Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the primary schema: a normalised, financially-critical core (networks, shows, royalties, campaigns, payouts, ledger) plus JSONB columns for schema-evolving data (Podcast Namespace tags, per-platform analytics payloads, VAST/OpenRTB creatives, AI annotations). This fits the central challenge — normalising heterogeneous hosting APIs — while keeping a single database for the MVP. A documented growth path toward Suggestion 4's polyglot model (ClickHouse for impressions, Neo4j for overlap graphs) is captured in the scaling notes.

---

## Core Requirements (Synthesis)

**What it does.** A unified operations layer for podcast *networks* (not single shows): a multi-show catalogue, automated host royalty splitting with multi-currency disbursement, dynamic-ad-insertion (DAI) configuration and campaign management, delivery reconciliation, and cross-platform analytics aggregated from heterogeneous hosting providers (Megaphone, Libsyn, Acast, Podbean, Buzzsprout) under IAB v2.2-credible deduplication.

**Who uses it.** Network operators (admins), show admins/producers, hosts/contributors (payout recipients), analysts, and advertiser viewers (read-only delivery reports).

**Key differentiators (the underserved gaps).**
1. Automated multi-contributor royalty splitting + international disbursement — *no incumbent offers this*.
2. Cross-platform analytics single pane of glass across mixed hosting.
3. Unified campaign reconciliation across programmatic + direct-sold inventory.
4. Network-level P&L per show.
5. AI-native: advertiser-show matching (embeddings), AI-suggested DAI markers, download anomaly/bot detection, transcript brand-safety pre-screening.

**Deployment model.** Self-hosted-first, cloud-deployable. Docker + docker-compose. Monolithic API with a worker process for async jobs (analytics sync, royalty calc, payouts, AI). Web dashboard SPA. Optional MCP server exposing catalogue/royalty/campaign/analytics tools.

**Integration surface.** Hosting platform APIs (OAuth 2.0 / token), RSS 2.0 + iTunes + Podcast Namespace feeds, Stripe Connect + Tipalti payouts, OpenRTB 2.6 / VAST 4.1 ad serving, OIDC SSO, optional LLM + embeddings provider.

**Standards to implement.** IAB Podcast Measurement v2.2 (and v2.1 ingestion), IAB OpenRTB 2.6, VAST 4.1+, RSS 2.0 / RFC 4287 Atom / iTunes namespace / Podcast Namespace 1.0 + PSP-1, OAuth 2.0 (RFC 6749/6750), OIDC Core 1.0, TLS 1.3 (RFC 8446), OpenAPI 3.1 + JSON Schema 2020-12, OWASP API Security Top 10 (2023), GDPR (IP truncation/minimisation), PCI DSS scope-reduction via Stripe/Tipalti, MCP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | Project is API/integration/frontend-heavy (5+ hosting APIs, Stripe/Tipalti, OpenRTB/VAST, web dashboard). One language across API, workers, MCP server, and SPA reduces context-switching. Strong typing protects financial code paths. |
| Runtime/package manager | pnpm workspaces (monorepo) | Shares the data model, OpenRTB/VAST, and IAB-dedup packages across api/worker/mcp/web without publishing. Fast, disk-efficient. |
| API framework | Fastify + `@fastify/swagger` | Highest-throughput Node framework (impression callbacks are hot path); first-class JSON Schema validation feeds OpenAPI 3.1 generation directly. |
| Validation / types | Zod + `zod-to-openapi` | Single source of truth for runtime validation and TS types; emits JSON Schema 2020-12 for the OpenAPI doc. |
| Database | PostgreSQL 16 + pgvector | Hybrid relational+JSONB model (Suggestion 3): ACID for money, JSONB+GIN for evolving Podcast Namespace and per-platform payloads, native range partitioning for time-series, GiST exclusion constraints for non-overlapping royalty rules, pgvector for advertiser-show matching. |
| ORM / migrations | Drizzle ORM + `drizzle-kit` | Thin, SQL-first; supports raw partitioning DDL, JSONB columns, GiST/exclusion constraints, and `vector` type via custom column types — things heavier ORMs (Prisma) fight against. Versioned SQL migrations. |
| Task queue | BullMQ on Redis | Async analytics sync, scheduled royalty calculations, payout disbursement state machines, AI jobs, materialised-view refresh. Repeatable (cron) jobs + retries + idempotency keys. |
| Cache / dedup store | Redis 7 | IAB v2.2 24-hour dedup windows (SETEX on `ip_hash:ua_hash`), dashboard caches, rate limiting. |
| Object storage | S3-compatible (MinIO local, S3 prod) | Generated PDF statements/reports, transcript files, chapter JSON, archived Parquet partitions. |
| Frontend | React 19 + Vite + TanStack Query + Tailwind + shadcn/ui + Recharts | Dashboard SPA: catalogue, analytics charts, campaign delivery, royalty/payout review, P&L. Generated typed client from the OpenAPI doc. |
| Auth | Lucia-style session + OIDC (`openid-client`) | Email/password + TOTP MFA (NIST 800-63B) for self-hosted; OIDC SSO for enterprise (Libsyn-style requirement). |
| Payments | `stripe` SDK + Tipalti REST (HMAC) | Stripe Connect transfers for most markets; Tipalti for 200-country / tax-form (W-9/W-8/VAT/DAC7) coverage. Keeps platform out of PCI cardholder scope. |
| Ad serving | Custom OpenRTB 2.6 + VAST 4.1 modules | No suitable OSS lib; thin typed builders/parsers validated against IAB Tech Lab JSON examples. |
| LLM / embeddings | Provider-agnostic via Vercel AI SDK | Brand-safety classification, DAI marker suggestion, advertiser-show match reasoning; embeddings into pgvector. Swappable provider. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | Exposes read-only catalogue/royalty/campaign/analytics tools to Claude/Cursor. |
| Testing | Vitest (unit/integration) + Supertest + Playwright (E2E) + Testcontainers (real PG/Redis) | One runner; Testcontainers gives real-DB integration without external infra. |
| Quality | ESLint + Prettier + `tsc --noEmit` + Drizzle schema check | Enforced in CI and Definition of Done. |
| Containerisation | Docker + docker-compose (api, worker, web, postgres, redis, minio) | Self-hosted one-command bring-up. |
| CI | GitHub Actions | Lint, type-check, test (with service containers), docker build, OpenAPI drift check. |

### Project Structure

```
podcast-network-management/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── docker-compose.yml                # postgres, redis, minio, api, worker, web
├── Dockerfile                        # multi-stage; builds api/worker/web targets
├── .github/workflows/ci.yml
├── openapi.json                      # generated, committed, drift-checked in CI
├── packages/
│   ├── db/                           # Drizzle schema + migrations + seed
│   │   ├── src/schema/               # one file per domain (networks, shows, ...)
│   │   ├── src/client.ts
│   │   ├── migrations/
│   │   └── seed/
│   ├── core/                         # framework-agnostic domain logic (no HTTP)
│   │   ├── src/royalty/              # split engine, calculation, statements
│   │   ├── src/iab/                  # v2.2 dedup + validity rules
│   │   ├── src/reconciliation/
│   │   ├── src/pnl/
│   │   ├── src/money/                # Money type, currency, FX
│   │   └── src/errors.ts
│   ├── integrations/                 # external system adapters
│   │   ├── src/hosting/              # base + megaphone/libsyn/acast/podbean/buzzsprout
│   │   ├── src/feeds/                # RSS/iTunes/Podcast-Namespace parse + generate
│   │   ├── src/payments/             # stripe-connect, tipalti
│   │   ├── src/adtech/               # openrtb, vast
│   │   └── src/ai/                   # embeddings, brand-safety, marker-suggest
│   ├── contracts/                    # Zod schemas + generated OpenAPI types
│   └── config/                       # env parsing (Zod), logging, encryption helpers
├── apps/
│   ├── api/                          # Fastify HTTP API
│   │   ├── src/routes/               # grouped by domain; each registers Zod schemas
│   │   ├── src/plugins/              # auth, rbac, db, redis, error-handler, swagger
│   │   ├── src/ad-serving/           # DAI request + VAST response (hot path)
│   │   └── src/server.ts
│   ├── worker/                       # BullMQ processors + repeatable schedulers
│   │   └── src/jobs/                 # analytics-sync, royalty-calc, payout, ai, refresh-views
│   ├── web/                          # React + Vite SPA
│   │   └── src/{pages,components,api-client,hooks}/
│   └── mcp/                          # MCP server over the platform API
└── tests/
    ├── fixtures/                     # sample RSS feeds, VAST XML, OpenRTB JSON, hosting API responses, server logs
    ├── integration/
    └── e2e/
```

Modules are grouped by concern, not by phase. Every phase adds files/routes/jobs without restructuring.

---

## Phase 1: Foundation — Monorepo, Database Core, Config, Auth Skeleton

### Purpose
Establish the buildable, testable monorepo, the hybrid PostgreSQL schema for the operational core, environment configuration, the encryption helper for credentials, and a running Fastify server with a health check and OpenAPI generation. After this phase, every later phase has a place to put code and a working migration/test harness. No business value ships yet, but the foundation is provably correct (migrations apply, server boots, schema check passes).

### Tasks

#### 1.1 — Monorepo + tooling bootstrap

**What**: Create the pnpm workspace with all packages/apps as empty buildable stubs, shared tsconfig, ESLint/Prettier, Vitest, and a Dockerfile + docker-compose bringing up postgres/redis/minio.

**Design**:
- `pnpm-workspace.yaml` globs `packages/*` and `apps/*`.
- `tsconfig.base.json`: `strict: true`, `noUncheckedIndexedAccess: true`, `target: ES2023`, `moduleResolution: bundler`, project references for incremental builds.
- Root scripts: `build`, `lint`, `typecheck`, `test`, `db:migrate`, `db:seed`, `openapi:gen`, `openapi:check`.
- `docker-compose.yml` services: `postgres:16` (pgvector image `pgvector/pgvector:pg16`), `redis:7`, `minio`, plus `api`/`worker`/`web` referencing the multi-stage Dockerfile (targets: `api`, `worker`, `web`).
- `packages/config`: `loadConfig()` parses `process.env` through a Zod schema and returns a typed, frozen config object. Required keys with defaults:

```ts
const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(8080),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  S3_ENDPOINT: z.string().url(),
  S3_BUCKET: z.string().default('pnm-assets'),
  S3_ACCESS_KEY: z.string(),
  S3_SECRET_KEY: z.string(),
  ENCRYPTION_KEY: z.string().length(64), // 32 bytes hex; AES-256-GCM
  SESSION_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});
export type AppConfig = z.infer<typeof EnvSchema>;
```

- `packages/config/crypto.ts`: `encrypt(plaintext: string): Buffer` / `decrypt(buf: Buffer): string` using AES-256-GCM with `ENCRYPTION_KEY`, 12-byte random IV prepended, 16-byte auth tag appended. Used for all stored OAuth tokens and API credentials.
- Structured logging via `pino`, configured with redaction of `authorization`, `password_hash`, `*_enc`, `set-cookie`.

**Testing**:
- `Unit: loadConfig with all required env vars → frozen AppConfig with correct coerced types`.
- `Unit: loadConfig missing DATABASE_URL → throws ZodError naming DATABASE_URL`.
- `Unit: encrypt then decrypt round-trips arbitrary UTF-8 string → original plaintext`.
- `Unit: decrypt with tampered ciphertext byte → throws (GCM auth failure)`.
- `Smoke: pnpm -r build succeeds for all stub packages`.

#### 1.2 — Database schema: operational core + enums + audit/JSONB conventions

**What**: Drizzle schema + first migration for the financially-critical normalised core and shared conventions (audit columns, soft delete, JSONB metadata, enums), adopting Suggestion 3.

**Design**:
- Enums (PG `CREATE TYPE`): `user_role(network_admin, show_admin, host, analyst, advertiser_viewer)`, `episode_status(draft, scheduled, published, archived, deleted)`, `ad_position(pre_roll, mid_roll, post_roll)`, `campaign_status(draft, proposed, booked, active, paused, completed, cancelled)`, `pricing_model(cpm, flat_rate, revenue_share, hybrid)`, `inventory_type(guaranteed, programmatic, house)`, `payout_status(pending, calculated, approved, processing, completed, failed, reversed)`, `subscription_tier(free, supporter, premium, patron)`, `subscription_status(active, paused, cancelled, expired, past_due)`, `hosting_platform(megaphone, libsyn, acast, podbean, buzzsprout, self_hosted, other)`, `tax_form_type(w9, w8ben, w8bene, vat_registration, dac7)`.
- Currency is stored as ISO-4217 `CHAR(3)` (not an enum — avoids migrations to add currencies).
- Convention helper `auditColumns`: `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `created_by UUID` (nullable, FK→users), `deleted_at TIMESTAMPTZ` (soft delete).
- Core tables this phase (from Suggestion 1/3, money columns `NUMERIC`):

```sql
networks(id uuid pk, name, slug unique, description, logo_url, website_url,
         default_currency char(3) default 'USD', timezone default 'America/New_York',
         iab_certified bool default false, stripe_account_id, tipalti_payer_id,
         settings jsonb default '{}'::jsonb, <audit>);

users(id uuid pk, email citext unique, name, avatar_url, password_hash,
      oidc_subject, oidc_issuer, mfa_enabled bool default false, mfa_secret_enc bytea,
      last_login_at, <audit>);

network_memberships(id uuid pk, network_id fk, user_id fk, role user_role default 'analyst',
                    invited_at, accepted_at, <audit>, unique(network_id, user_id));

shows(id uuid pk, network_id fk, title, slug, description, summary,
      language default 'en', explicit bool default false, cover_image_url, website_url,
      rss_feed_url, hosting_platform hosting_platform default 'self_hosted',
      hosting_external_id, podcast_guid uuid, copyright, is_active bool default true,
      publish_frequency, itunes jsonb default '{}'::jsonb,   -- category/subcategory/owner/type
      metadata jsonb default '{}'::jsonb,                    -- Podcast Namespace evolving tags
      <audit>, unique(network_id, slug));

show_hosts(id uuid pk, show_id fk, user_id fk null, person_name, person_role default 'host',
           person_url, person_image_url, podcast_person_href, is_primary bool default false,
           started_at date, ended_at date, <audit>);

episodes(id uuid pk, show_id fk, title, slug, description, summary,
         status episode_status default 'draft', season_number int, episode_number int,
         episode_type default 'full', audio_url, audio_duration_secs int, audio_size_bytes bigint,
         audio_mime_type default 'audio/mpeg', cover_image_url, explicit bool default false,
         transcript_url, chapters_url, guid text not null, published_at, scheduled_at,
         hosting_external_id, metadata jsonb default '{}'::jsonb, <audit>,
         unique(show_id, slug));
```

- Indexes: `idx_shows_network ON shows(network_id) WHERE deleted_at IS NULL`; `idx_episodes_show_published ON episodes(show_id, published_at DESC) WHERE deleted_at IS NULL`; GIN on `shows.metadata`, `episodes.metadata`.
- `audit_log(id bigserial, network_id, user_id, action, entity_type, entity_id uuid, old_values jsonb, new_values jsonb, ip_address inet, user_agent, created_at)` PARTITION BY RANGE(created_at); create current + next month partitions in migration; index `(entity_type, entity_id)` and `(user_id, created_at)`.
- Drizzle custom types: `vector(n)` and `citext` registered as custom column types. Enable extensions in migration: `CREATE EXTENSION IF NOT EXISTS vector; ... citext;`.

**Testing** (Testcontainers PostgreSQL):
- `Integration: run all migrations on empty DB → succeeds; pgvector + citext extensions present`.
- `Integration: insert network + show with duplicate (network_id, slug) → unique violation`.
- `Integration: soft-delete a show (set deleted_at) → excluded from idx_shows_network partial index path (query returns 0)`.
- `Integration: insert episode metadata JSONB with podcast:chapters key → GIN containment query @> finds it`.
- `Unit: drizzle schema typecheck (tsc) passes`.

#### 1.3 — Fastify server, plugins, health, OpenAPI generation

**What**: Bootable API with db/redis plugins, global error handler, request-id + logging, `/healthz`, `/readyz`, and `openapi.json` generation from Zod route schemas.

**Design**:
- `apps/api/src/server.ts`: `buildServer(config): FastifyInstance` registering plugins: `db` (Drizzle pool, decorates `app.db`), `redis`, `errorHandler`, `swagger` (`@fastify/swagger` + `zod-to-openapi`, OpenAPI 3.1, `info.version` from package.json), `swagger-ui` at `/docs`.
- Error model — every error response conforms to:

```ts
type ApiError = { error: { code: string; message: string; details?: unknown; requestId: string } };
```

Domain errors extend `AppError(code, httpStatus, message)`; the handler maps Zod validation → `422 VALIDATION_ERROR` with field paths, unknown → `500 INTERNAL` (message hidden in production).
- `GET /healthz` → `{ status: 'ok' }` (liveness, no deps). `GET /readyz` → checks `SELECT 1` + Redis `PING`, returns 200/503.
- `openapi:gen` script writes `openapi.json`; `openapi:check` regenerates to a temp file and diffs against committed copy, failing on drift.

**Testing**:
- `Integration: GET /healthz → 200 {status:'ok'}`.
- `Integration (Testcontainers): GET /readyz with PG+Redis up → 200`.
- `Integration: GET /readyz with Redis stopped → 503`.
- `Integration: trigger a route throwing AppError('NOT_FOUND',404) → body matches ApiError shape with requestId`.
- `Unit: openapi:check passes when openapi.json is current; fails when a route schema changes without regen`.

---

## Phase 2: Identity, RBAC, and Network/Show/Episode Catalogue

### Purpose
Deliver the first end-to-end user-facing capability: a network operator can sign up, create a network, invite members with roles, and manage shows/episodes — the catalogue that everything else hangs off. Establishes authentication, the RBAC authorization layer (enforcing OWASP API #1 broken-object-level-authorization protection by scoping all queries to network membership), and full CRUD for the catalogue.

### Tasks

#### 2.1 — Authentication: password + session + TOTP MFA

**What**: Email/password registration, login issuing an HTTP-only session cookie, logout, and optional TOTP MFA enrolment/verification.

**Design**:
- `POST /auth/register {email, name, password}` → creates user (Argon2id hash via `@node-rs/argon2`), 201 `{userId}`.
- `POST /auth/login {email, password, totp?}` → on success sets `Set-Cookie: session=<opaque>; HttpOnly; Secure; SameSite=Lax`; session stored in Redis (`session:<id>` → `{userId, createdAt}`, 30-day TTL, sliding). If `mfa_enabled` and `totp` missing/invalid → `401 MFA_REQUIRED` / `401 INVALID_CREDENTIALS` (uniform timing).
- `POST /auth/mfa/enroll` (authed) → returns `otpauth://` URI + base32 secret; stores `mfa_secret_enc` (encrypted) but `mfa_enabled=false` until `POST /auth/mfa/verify {totp}` confirms one code (using `otplib`, ±1 step skew).
- `POST /auth/logout` → deletes session.
- Auth plugin decorates `req.user` from session; unauthenticated protected routes → `401 UNAUTHENTICATED`.
- Passwords: min length 12 (NIST 800-63B), check against a small common-password blocklist.

**Testing**:
- `Unit: Argon2 hash+verify round-trip; wrong password → false`.
- `Integration: register then login → 200 + session cookie; whoami returns user`.
- `Integration: login with wrong password → 401, no cookie, response time within X of correct-path (timing-safe)`.
- `Integration: enroll MFA, verify with code from secret → mfa_enabled true; subsequent login without totp → 401 MFA_REQUIRED`.
- `Integration: logout → session key gone from Redis; protected route → 401`.

#### 2.2 — RBAC + network scoping (OWASP API #1/#5)

**What**: A reusable authorization layer that resolves the caller's role within a target network and enforces per-action permissions, with all data access scoped to network membership.

**Design**:
- Permission matrix (action → minimum role), e.g.:

```ts
const PERMISSIONS = {
  'network:update':   ['network_admin'],
  'member:invite':    ['network_admin'],
  'show:create':      ['network_admin', 'show_admin'],
  'show:update':      ['network_admin', 'show_admin'],
  'episode:write':    ['network_admin', 'show_admin'],
  'royalty:configure':['network_admin'],
  'payout:approve':   ['network_admin'],
  'campaign:write':   ['network_admin', 'show_admin'],
  'analytics:read':   ['network_admin','show_admin','host','analyst','advertiser_viewer'],
  'report:read':      ['network_admin','show_admin','analyst','advertiser_viewer'],
} as const;
```

- `requirePermission(action)` Fastify preHandler: reads `networkId` from route params, loads the caller's `network_memberships` row, 403 `FORBIDDEN` if role insufficient, 404 if no membership (do not leak existence). Decorates `req.membership`.
- A `scopedShow(showId, networkId)` helper used everywhere: every entity fetch joins/filters on `network_id` so a user of network A can never read/mutate network B objects even with a guessed UUID.
- `POST /networks/:id/members {email, role}` invites; new users get a pending membership (`accepted_at NULL`) and an invite token (Redis, 7-day TTL).

**Testing**:
- `Integration: show_admin calls payout:approve route → 403`.
- `Integration: user with membership in network A requests GET /networks/B/shows/:id → 404 (not 403, no leak)`.
- `Integration: network_admin updates own network → 200`.
- `Unit: requirePermission resolves role from membership and matches matrix for each action`.
- `Integration: invite member → pending membership row; accept via token → accepted_at set`.

#### 2.3 — Catalogue CRUD: networks, shows, episodes, hosts

**What**: Full CRUD for networks, shows, show_hosts, and episodes with validation, soft delete, and audit logging.

**Design**:
- REST resources (all under network scope, all writes audit-logged via an `withAudit` wrapper that diffs old/new and inserts `audit_log`):
  - `POST /networks`, `GET /networks` (memberships of caller), `GET/PATCH/DELETE /networks/:id`.
  - `POST/GET /networks/:nid/shows`, `GET/PATCH/DELETE /networks/:nid/shows/:sid` (DELETE = soft).
  - `POST/GET/PATCH/DELETE /networks/:nid/shows/:sid/hosts`.
  - `POST/GET /networks/:nid/shows/:sid/episodes` (paginated, `?status=&cursor=`), `GET/PATCH/DELETE …/episodes/:eid`.
- Zod request/response schemas in `packages/contracts`; episode `metadata` accepts arbitrary JSON validated as an object (Podcast Namespace passthrough). `slug` auto-generated from title if absent, deduped within parent.
- Royalty-split invariant teaser (enforced in Phase 4) referenced here: deleting a show with active royalty rules → `409 CONFLICT`.
- Cursor pagination: opaque base64 of `(published_at, id)`; default page size 50, max 200.

**Testing**:
- `Integration: create network → create show → create episode → list episodes returns it`.
- `Integration: create two shows same slug in one network → 422`.
- `Integration: PATCH episode metadata adds podcast:transcript key → persisted; audit_log row has old/new diff`.
- `Integration: DELETE show → deleted_at set; subsequent GET → 404; list excludes it`.
- `Integration: pagination cursor returns stable, non-overlapping pages across 120 episodes`.
- `E2E (Playwright): operator signs up, creates network + show + episode via UI → appears in catalogue list`.

---

## Phase 3: Feed Engine + Hosting-Platform Normalisation Layer

### Purpose
Build the data-ingress heart of the cross-platform value proposition: parse and generate standards-compliant RSS feeds, and a hosting-platform adapter abstraction that normalises Megaphone/Libsyn/Acast/Podbean/Buzzsprout into the common schema. After this phase a network can connect external hosting accounts and import shows/episodes, and the platform can emit compliant feeds — enabling the analytics and ad layers that follow.

### Tasks

#### 3.1 — RSS / iTunes / Podcast Namespace parser + generator

**What**: Parse external RSS 2.0 feeds (iTunes + Podcast Namespace + Atom) into normalised show/episode objects, and generate PSP-1-compliant feeds for self-hosted shows.

**Design**:
- `parseFeed(xml: string): ParsedFeed` using `fast-xml-parser`. Extracts: channel (`title`, `itunes:author`, `itunes:category/subcategory`, `language`, `itunes:explicit`, `itunes:image`, `podcast:guid`, `atom:link[rel=self]`) and items (`title`, `guid` + `isPermaLink`, `enclosure {url,length,type}`, `itunes:duration` parsed from `HH:MM:SS` or seconds, `itunes:episode/season/episodeType`, `pubDate`, `podcast:transcript`, `podcast:chapters`, `podcast:person`). Unknown podcast-namespace tags preserved into `metadata` JSONB.
- `generateFeed(show, episodes, opts): string` produces RSS 2.0 with `xmlns:itunes`, `xmlns:atom`, `xmlns:podcast`; mandatory PSP-1 tags enforced; `<atom:link rel="self">` self-reference; episodes sorted by `published_at` desc; `itunes:duration` formatted; CDATA-wrapped descriptions.
- Feed served at `GET /feeds/:network_slug/:show_slug.xml` (public, ETag + `Cache-Control`).
- Private/premium feed variant (Phase 8) takes a `?token=` and filters entitled episodes.

**Testing** (fixtures in `tests/fixtures/feeds/`):
- `Unit: parse Apple-namespace sample feed → correct channel + N items with durations in seconds`.
- `Unit: parse feed with podcast:transcript and podcast:chapters → URLs captured in episode.metadata`.
- `Unit: parse guid with isPermaLink=false → stored verbatim`.
- `Unit: generateFeed then parseFeed round-trips title/guid/enclosure/duration`.
- `Unit: generated feed validates against PSP-1 mandatory-tag checklist (no missing required tag)`.
- `Unit: malformed XML → ParseError, not crash`.

#### 3.2 — Hosting adapter abstraction + OAuth/token credential store

**What**: A `HostingAdapter` interface plus a credential store (encrypted) and OAuth 2.0 client flows so a network can connect external hosting accounts.

**Design**:

```ts
interface HostingAdapter {
  readonly platform: HostingPlatform;
  authType: 'oauth2' | 'api_token';
  // OAuth platforms (Podbean, Acast):
  getAuthorizeUrl?(state: string): string;
  exchangeCode?(code: string): Promise<TokenSet>;
  refresh?(refreshToken: string): Promise<TokenSet>;
  listShows(creds: Credentials): Promise<NormalisedShow[]>;
  listEpisodes(creds: Credentials, externalShowId: string, since?: Date): Promise<NormalisedEpisode[]>;
  fetchAnalytics(creds: Credentials, externalShowId: string, range: DateRange): Promise<NormalisedDailyMetric[]>;
}
```

- `analytics_sources(id, show_id, platform, api_credential_enc bytea, oauth_token_enc, oauth_refresh_enc, oauth_expires_at, last_sync_at, sync_status, sync_error_message, raw_payload jsonb, <audit>)` — credentials stored via `packages/config/crypto`. `raw_payload` JSONB retains the last raw API response per Suggestion 3 (debuggability + lossless re-normalisation).
- OAuth callback route `GET /networks/:nid/integrations/:platform/callback?code&state` validates `state` (Redis nonce), exchanges code, stores encrypted tokens. Token-auth platforms (Buzzsprout `Authorization: Token token=`, Megaphone enterprise token, Libsyn API key) accept a credential via `POST /networks/:nid/integrations/:platform {apiKey}`.
- A `NormalisedShow`/`NormalisedEpisode`/`NormalisedDailyMetric` set of types is the canonical interchange shape all adapters map to.
- Adapters per platform implement only the endpoints their API exposes; capabilities advertised via `adapter.capabilities` so the UI can grey out unsupported features.

**Testing** (mocked HTTP via `nock`/`msw`, fixtures of each platform's real response shape):
- `Unit (mocked): Buzzsprout listEpisodes → NormalisedEpisode[] with correct duration + audio_url`.
- `Unit (mocked): Podbean OAuth exchangeCode → TokenSet; refresh returns new access token`.
- `Integration: OAuth callback with mismatched state → 400, no token stored`.
- `Integration: store Buzzsprout token → api_credential_enc is ciphertext (not plaintext) in DB`.
- `Unit: each adapter maps its analytics payload to NormalisedDailyMetric with iab_version tag`.

#### 3.3 — Import + sync jobs

**What**: BullMQ jobs that import shows/episodes from a connected source and incrementally sync new episodes; idempotent on `(platform, hosting_external_id)`.

**Design**:
- `importShowJob({sourceId})`: pulls shows+episodes, upserts on `shows.hosting_external_id` / `episodes.hosting_external_id` scoped to network; sets `analytics_sources.last_sync_at`.
- Repeatable scheduler enqueues `syncSourceJob` per active source every 6h (configurable); uses `since = last_sync_at` for incremental episode pulls.
- Idempotency: upsert keyed on external id; episode `guid` collision across re-import updates in place (no duplicate).
- Failures set `sync_status='error'` + `sync_error_message`, retried with exponential backoff (max 5), surfaced in the UI integrations panel.

**Testing**:
- `Integration (mocked adapter): importShowJob → shows+episodes upserted; re-run → no duplicates, updated fields applied`.
- `Integration: syncSourceJob with since-filter → only new episodes inserted`.
- `Integration: adapter throws 429 → job retried with backoff; after max retries sync_status='error'`.

---

## Phase 4: Host Royalty Management (Core Differentiator)

### Purpose
Implement the platform's headline differentiator: configurable multi-contributor revenue-share rules, automated royalty calculation from ad + subscription + other revenue, and generated statements — the capability no incumbent offers. This is intentionally early (phase 4) because it is the product's reason to exist. Disbursement (the payment rail) follows in Phase 5.

### Tasks

#### 4.1 — Revenue ledger + Money type

**What**: A revenue/expense ledger and a precise Money value type that all financial math uses.

**Design**:
- `packages/core/money`: `Money` = `{ amountMinor: bigint; currency: string }` (integer minor units, never floats). Functions: `add/sub` (same-currency guard), `allocate(weights: number[]): Money[]` (largest-remainder distribution so split sums exactly equal the total — no lost/created cents), `fromDecimal(string, currency)`, `toDecimal()`. DB stores `NUMERIC`; mapper converts to/from minor units using currency exponent table (ISO 4217; JPY=0, most=2).
- `revenue_entries(id, network_id, show_id null, source_type ['ad_campaign'|'subscription'|'sponsorship'|'other'], source_id uuid null, amount numeric, currency char(3), revenue_date date, description, <audit>)` PARTITION BY RANGE(revenue_date).
- `expense_entries(id, network_id, show_id null, category ['hosting'|'payout'|'agency_commission'|'platform_fee'], amount numeric, currency, expense_date date, description, reference_id uuid null, <audit>)` PARTITION BY RANGE(expense_date).
- Ad campaign revenue auto-posts here on reconciliation (Phase 7); subscription revenue on Stripe invoice webhook (Phase 8); manual entries via `POST /networks/:nid/revenue`.

**Testing**:
- `Unit: allocate $100.00 across [1/3,1/3,1/3] → [33.34,33.33,33.33] summing to exactly 100.00`.
- `Unit: allocate ¥1000 (exponent 0) across uneven weights → integer yen, exact sum`.
- `Unit: add Money of differing currencies → throws CurrencyMismatch`.
- `Unit: fromDecimal('19.99','USD') → 1999 minor; toDecimal → '19.99'`.

#### 4.2 — Royalty rules + splits with non-overlap invariant

**What**: Per-show revenue-share rules with multiple contributor splits, enforcing that splits sum to ≤100% and that active rules for a show never overlap in time.

**Design**:

```sql
royalty_rules(id, show_id fk, name, description,
              network_commission_pct numeric(7,4) default 0,  -- network share off the top
              effective_from date not null, effective_until date null,
              is_active bool default true, <audit>,
              EXCLUDE USING gist (show_id WITH =,
                 daterange(effective_from, coalesce(effective_until,'infinity'), '[]') WITH &&)
                 WHERE (is_active));
royalty_splits(id, royalty_rule_id fk, recipient_host_id fk→show_hosts,
               split_percentage numeric(7,4) not null,
               minimum_amount numeric(10,2) default 0, cap_amount numeric(10,2) null, <audit>,
               CHECK (split_percentage > 0 AND split_percentage <= 100));
```

- Service-level invariant on create/update: `SUM(split_percentage) <= 100` for a rule (remainder is implicit network retention); reject `422 SPLITS_EXCEED_100` otherwise.
- `POST /networks/:nid/shows/:sid/royalty-rules` (network_admin only) with nested splits; updating an active rule's window that would overlap → `409 RULE_OVERLAP` (caught from the exclusion constraint and translated).
- Endpoints to list/version rules; old rules kept (set `effective_until`, `is_active=false`) for audit.

**Testing**:
- `Unit/Integration: splits summing to 70% (+30% network) → accepted`.
- `Integration: splits summing to 110% → 422 SPLITS_EXCEED_100`.
- `Integration: create rule [2026-01-01, NULL] then [2026-06-01, NULL] same show → 409 RULE_OVERLAP`.
- `Integration: create rule [2026-01-01,2026-05-31] then [2026-06-01,NULL] → both accepted`.

#### 4.3 — Royalty calculation engine + statements

**What**: Given a show and period, compute total revenue, deduct network commission, distribute to contributors per the active rule (honouring minimums/caps), and persist an immutable calculation + line items; generate a PDF statement.

**Design**:
- `calculateRoyalties(showId, period): RoyaltyCalculation`:
  1. Sum `revenue_entries` for the show in `[period_start, period_end]`, grouped into `ad_revenue / subscription_revenue / other_revenue` by `source_type`.
  2. Select the active `royalty_rules` row covering the period (error `409 NO_RULE` if none/ambiguous).
  3. `network_commission = total * network_commission_pct`; `distributable = total - commission`.
  4. `allocate(distributable, splitWeights)` → per-host gross; apply `minimum_amount` floor and `cap_amount` ceiling (overflow returns to network as additional commission to keep books balanced); snapshot `split_percentage` into the line item.
  5. Persist `royalty_calculations` + `royalty_line_items` atomically (single transaction). Recalculation for an already-paid period is blocked (`409 PERIOD_LOCKED` once any payout references its line items).
- Tables:

```sql
royalty_calculations(id, show_id, royalty_rule_id, period_start, period_end,
  total_revenue, ad_revenue, subscription_revenue, other_revenue,
  network_commission, distributable_amount, currency char(3),
  calculated_at, approved_by uuid null, approved_at, <audit>);
royalty_line_items(id, calculation_id fk, recipient_host_id fk,
  split_percentage numeric(7,4), gross_amount, adjustments numeric default 0,
  net_amount, currency char(3), <audit>);
```

- `POST /networks/:nid/shows/:sid/royalties/calculate {period_start, period_end}` → returns calculation; `POST …/royalties/:cid/approve` (network_admin) sets `approved_*`.
- Statement PDF generated by a worker job (`@react-pdf/renderer`) → uploaded to S3 → `statement_url` stored on the eventual payout (Phase 5).
- A repeatable monthly scheduler can auto-create draft calculations for all shows with active rules.

**Testing**:
- `Integration: show with $1000 ad rev, 10% network commission, splits 50/30/20 → line items 450/270/180 summing to 900; commission 100`.
- `Integration: contributor with minimum_amount=200 on a $180 share → paid 200; deficit charged to network`.
- `Integration: contributor with cap_amount=100 on a $180 share → paid 100; 80 returns to network`.
- `Integration: calculate then approve then attempt recalculate after a payout exists → 409 PERIOD_LOCKED`.
- `Integration: period with no active rule → 409 NO_RULE`.
- `Unit: sum of line-item net + commission == total_revenue (no cents lost) across randomised property test`.

---

## Phase 5: Payment Disbursement (Stripe Connect + Tipalti)

### Purpose
Connect the royalty engine to real money movement: collect contributor payment profiles and tax forms, then disburse approved royalties via Stripe Connect (primary) or Tipalti (global/tax coverage), with a robust payout state machine, multi-currency FX, and reconciliation against provider webhooks. After this phase the headline workflow — calculate → approve → pay → statement — is complete.

### Tasks

#### 5.1 — Payment profiles + tax forms + provider onboarding

**What**: Per-user payment profiles with provider account linkage and tax-document status.

**Design**:

```sql
payment_profiles(id, user_id fk, network_id fk, provider ['stripe'|'tipalti'],
  stripe_connect_id, tipalti_payee_id, preferred_currency char(3) default 'USD',
  payment_method default 'bank_transfer', tax_form_type tax_form_type,
  tax_form_status ['pending'|'submitted'|'verified'|'expired'] default 'pending',
  tax_form_verified_at, bank_country char(2), <audit>);
```

- `POST /networks/:nid/payment-profiles` initiates onboarding: Stripe → create Express connected account + return account-link URL; Tipalti → create payee + return iFrame onboarding URL (tax forms collected by provider, keeping the platform out of PII/tax-doc storage scope).
- Webhook ingestion (`account.updated` / Tipalti payee status) updates `tax_form_status` and a `payouts_enabled` flag.

**Testing** (Stripe/Tipalti mocked):
- `Integration (mocked Stripe): create profile → connected account created; account-link URL returned`.
- `Integration: account.updated webhook with charges/payouts enabled → tax_form_status='verified'`.
- `Integration: webhook signature invalid → 400, no state change`.

#### 5.2 — Payout state machine + FX

**What**: Create payouts from approved royalty line items and drive them through a strict lifecycle to a provider transfer.

**Design**:
- State machine: `pending → calculated → approved → processing → completed` with side branches `→ failed` (from processing) and `completed → reversed`. Illegal transitions throw `409 INVALID_TRANSITION`.

```sql
payouts(id, network_id, recipient_user_id, payment_profile_id, status payout_status default 'pending',
  amount numeric, currency char(3), exchange_rate numeric(12,6) null, amount_local numeric null,
  period_start date, period_end date, provider ['stripe'|'tipalti'],
  stripe_transfer_id, tipalti_payment_id, idempotency_key text unique,
  processed_at, failed_reason, statement_url, <audit>);
payout_line_items(id, payout_id fk, royalty_line_item_id fk, show_id fk, amount numeric, description, <audit>);
```

- `createPayoutBatch(networkId, period)`: groups all approved, unpaid `royalty_line_items` by `recipient_user_id`, sums per recipient, applies FX from network default currency → recipient `preferred_currency` (FX provider abstracted; rate snapshotted into `exchange_rate`/`amount_local`), generates an `idempotency_key = hash(networkId, userId, period)`.
- Disbursement job calls provider with the idempotency key (so retries never double-pay); on provider success → `completed` + `processed_at` + posts an `expense_entries` row (category `payout`). Provider webhook confirms terminal status; mismatch → `failed`.
- `POST /networks/:nid/payouts/batch {period}` (network_admin) → returns created payouts in `approved`; `POST …/payouts/:id/process` enqueues disbursement.

**Testing**:
- `Unit: state machine allows approved→processing; rejects pending→completed`.
- `Integration: createPayoutBatch aggregates 3 line items for one host into one payout`.
- `Integration: FX USD→EUR applies snapshotted rate; amount_local computed`.
- `Integration (mocked Stripe): process payout → transfer created with idempotency_key; status completed; expense_entry posted`.
- `Integration: re-run process on same payout (same idempotency_key) → provider not double-charged (single transfer)`.
- `Integration: provider returns failure webhook → status failed, failed_reason set`.

---

## Phase 6: Cross-Platform Analytics + IAB v2.2 Deduplication

### Purpose
Deliver the second core differentiator: a single pane of glass aggregating download/listener metrics across heterogeneous hosting platforms, with IAB v2.2-credible deduplication so advertiser-facing numbers withstand audit. Builds on the adapters (Phase 3) for ingestion and feeds the campaign/reconciliation layer (Phase 7).

### Tasks

#### 6.1 — IAB v2.2 deduplication engine

**What**: A reusable engine that processes raw download/request events into valid, deduplicated download counts per IAB Podcast Measurement Technical Guidelines v2.2 (and v2.1 ingestion).

**Design**:
- `packages/core/iab`: `isValidDownload(req): {valid: boolean; reason?: string}` applying v2.2 filters: GET (or partial 206) request, ≥ a threshold of bytes for the enclosure, exclude known bot/crawler user agents (maintained UA blocklist), exclude duplicate Apple Watch companion requests, exclude HEAD requests.
- `dedupKey(ipHash, uaHash, contentId)`; uniqueness within a rolling 24-hour window enforced via Redis `SET key 1 NX EX 86400` — first occurrence counts, repeats within window are duplicates. IP is hashed with a daily-rotating salt and truncated/anonymised before hashing (GDPR minimisation; IPv4 /24, IPv6 /48).
- Batch path for log-file ingestion (`processLogBatch(rows)`) reproduces the v2.x 5-step methodology deterministically for backfill/audit (sortable, replayable — does not rely on Redis for historical recompute).
- `iab_version` recorded per source so mixed v2.1/v2.2 origins are labelled.

**Testing** (fixtures: sample server-log lines, bot UA list):
- `Unit: HEAD request → invalid (reason 'not_get')`.
- `Unit: request below byte threshold → invalid`.
- `Unit: known bot UA → invalid`.
- `Unit: two identical ip/ua/content within 24h → first valid, second duplicate`.
- `Unit: same ip/ua next day → valid again`.
- `Unit: processLogBatch deterministic — same input twice → identical counts`.
- `Unit: IPv4 anonymised to /24 before hashing (last octet zeroed)`.

#### 6.2 — Normalised analytics store + network aggregation

**What**: Persist per-day, per-source episode metrics (downloads, unique listeners, completion, geo, app/device) and aggregate to show/network level.

**Design**:

```sql
episode_analytics(id bigserial, episode_id fk, show_id fk, source_platform hosting_platform,
  metric_date date, downloads bigint default 0, unique_listeners bigint default 0,
  completion_rate numeric(5,4), avg_listen_duration_secs int, iab_version,
  raw jsonb default '{}'::jsonb,    -- platform-specific extra metrics (Suggestion 3)
  unique(episode_id, source_platform, metric_date)) PARTITION BY RANGE(metric_date);
episode_geo_analytics(... country_code char(2), region_code, downloads, unique_listeners,
  unique(episode_id, metric_date, country_code, region_code)) PARTITION BY RANGE(metric_date);
episode_app_analytics(... app_name, device_type, downloads, unique_listeners,
  unique(episode_id, metric_date, app_name)) PARTITION BY RANGE(metric_date);
```

- `network_analytics_summary` materialised view (network_id, show_id, metric_date, totals/avg completion) refreshed `CONCURRENTLY` by a worker job every 15 min; partition auto-creation via a monthly scheduler.
- Ingestion: adapter `fetchAnalytics` results upserted on the unique key; self-hosted shows feed from the dedup engine directly. Platform-specific extras (e.g. Podbean second-by-second) preserved in `raw`.
- `GET /networks/:nid/analytics?from&to&showIds&groupBy=day|show|network` returns aggregated series; advertiser-viewer scoped to permitted shows.

**Testing**:
- `Integration: upsert same (episode,source,date) twice → single row, latest values`.
- `Integration: aggregate two sources for one episode-day → summed downloads at show level`.
- `Integration: refresh network_analytics_summary → totals match underlying sum`.
- `Integration: analytics query as advertiser_viewer → only permitted shows returned`.

---

## Phase 7: Ad Insertion, Campaign Management, Delivery & Reconciliation

### Purpose
Add the monetisation engine: DAI insertion-point management, advertiser/agency/campaign records, a VAST 4.1 DAI request endpoint that logs impressions, and reconciliation comparing booked vs. IAB-valid delivered impressions — feeding revenue into the ledger (Phase 4) and advertiser-facing reports (Phase 9). OpenRTB 2.6 programmatic demand is included as an optional path.

### Tasks

#### 7.1 — Advertisers, agencies, campaigns, targeting

**What**: CRUD for advertisers/agencies and campaigns with show targeting, exclusions, pricing, flight dates, and inventory type.

**Design**:
- Tables `advertisers`, `agencies`, `campaigns` (status/pricing_model/inventory_type enums; `cpm_rate`/`flat_rate_amount`/`budget_amount` numeric; `booked_impressions`; `flight_start/end` with `CHECK(flight_end>=flight_start)`; `vast_tag_url`; `brand_safety_tags text[]`; `targeting jsonb` for geo/device/category rules — JSONB per Suggestion 3).
- `campaign_show_targets(campaign_id, show_id, ad_position, allocated_impressions, weight, unique(campaign_id,show_id,ad_position))`.
- `campaign_exclusions(campaign_id, excluded_advertiser_id?, excluded_category?)` for competitive separation.
- `ad_insertion_points(id, episode_id, position ad_position, offset_secs numeric(10,3), max_duration_secs default 60, is_auto_detected bool, confidence_score numeric(5,4), <audit>)`.
- Campaign status transitions validated: `draft→proposed→booked→active→(paused↔active)→completed`, `→cancelled` from any non-terminal.

**Testing**:
- `Integration: create campaign with flight_end < flight_start → 422`.
- `Integration: target campaign at show → campaign_show_targets row; duplicate (campaign,show,position) → 422`.
- `Integration: campaign status booked→completed skipping active → 409 INVALID_TRANSITION`.
- `Integration: add manual mid-roll insertion point at 600.0s → persisted`.

#### 7.2 — DAI request endpoint + VAST 4.1 response + impression logging

**What**: A hot-path endpoint that, given an episode + insertion point + listener context, selects an eligible campaign, returns a VAST 4.1 response, and records a (deduplicated) ad impression.

**Design**:
- `GET /ad?episode=&pos=&ip=&ua=&geo=` (called by the audio-stitching layer / player):
  1. Validate the request as an IAB-valid download context (reuse Phase 6 engine).
  2. Candidate campaigns = active, in-flight, targeting this show+position, not excluded by competitive rules, with remaining `booked_impressions`. Rank by inventory_type (guaranteed first), then weight, then pacing (even delivery across flight). If none, optionally fall to OpenRTB programmatic (7.4) or house ads.
  3. Build VAST 4.1 XML (`packages/integrations/adtech/vast`) — `<VAST version="4.1">` with `<Linear>`, `<MediaFiles>`/`<MediaFile delivery="progressive">`, `<TrackingEvents>`, impression URL, and SSAI signalling; or pass through the campaign's `vast_tag_url`.
  4. Insert `ad_impressions` row (partitioned) with hashed ip/ua, geo, device, `is_valid`, `dedup_window_id`.

```sql
ad_impressions(id bigserial, campaign_id fk, episode_id fk, show_id fk,
  insertion_point_id fk null, ad_position ad_position, listener_ip_hash char(64),
  user_agent_hash char(64), country_code char(2), region_code, city, device_type,
  app_name, is_valid bool default true, dedup_window_id, delivered_at timestamptz)
  PARTITION BY RANGE(delivered_at);
```

- `campaign_daily_delivery` materialised view (valid/invalid impressions, unique listeners) refreshed by worker.
- Hot path: no synchronous heavy work — impression insert may be buffered to a Redis stream and flushed in batches by the worker for throughput.

**Testing** (fixtures: IAB Tech Lab VAST examples):
- `Unit: VAST builder output validates against VAST 4.1 XSD/sample structure`.
- `Integration: GET /ad with eligible campaign → 200 VAST XML; ad_impression inserted is_valid=true`.
- `Integration: duplicate ip/ua within 24h → impression inserted is_valid=false (counts toward delivered, not valid)`.
- `Integration: no eligible campaign, no programmatic → 204 (no ad) or house ad`.
- `Integration: campaign with exhausted booked_impressions → not selected`.

#### 7.3 — Delivery reconciliation + revenue posting

**What**: Compare booked vs. delivered (IAB-valid) impressions per campaign per period, compute earned revenue, and post it to the ledger.

**Design**:
- `reconcileCampaign(campaignId, period)`:
  - `delivered = COUNT(*) , valid = COUNT(*) WHERE is_valid` from `ad_impressions` in range.
  - `delivery_pct = valid / booked_impressions * 100`.
  - Revenue: CPM → `valid/1000 * cpm_rate`; flat_rate → `flat_rate_amount` (pro-rated by delivery if under-delivered, per campaign terms flag); minus `agency.commission_pct` if applicable.
  - Persist `campaign_reconciliations(campaign_id, period_start/end, booked_impressions, delivered_impressions, valid_impressions, delivery_pct, revenue_earned, currency, reconciled_by, reconciled_at)`; post `revenue_entries` (source_type `ad_campaign`, source_id=campaign) and `expense_entries` (agency_commission) so royalties (Phase 4) pick it up.
- `POST /networks/:nid/campaigns/:cid/reconcile {period}` (network_admin/show_admin).

**Testing**:
- `Integration: 1.2M valid impressions at $25 CPM → revenue_earned $30,000; revenue_entry posted`.
- `Integration: 15% agency campaign → expense_entry of commission posted`.
- `Integration: under-delivered flat-rate with pro-rate flag → revenue scaled by delivery_pct`.
- `Integration: reconciled revenue flows into next royalty calculation for targeted shows`.

#### 7.4 — OpenRTB 2.6 programmatic path (optional demand)

**What**: Emit OpenRTB 2.6 bid requests (with the Audio object) to a configured exchange and translate winning bids into VAST for insertion.

**Design**:
- `packages/integrations/adtech/openrtb`: typed `BidRequest`/`BidResponse` builders for OpenRTB 2.6 including the `Audio` object (`feed=3` podcast, `stitched`, `nvol`), `pod` fields, `cur`, floor price. POST to exchange endpoint; on `BidResponse` win, use `bid.adm` (VAST) for the DAI response and record the clearing price as campaign revenue (inventory_type `programmatic`).
- Timeout-bounded (`tmax`); on no-bid/timeout, fall back to guaranteed/house ad.

**Testing** (mocked exchange, IAB OpenRTB sample JSON):
- `Unit: BidRequest serialises Audio object with feed=3 and stitched flag → matches IAB 2.6 sample`.
- `Unit: parse BidResponse, extract adm VAST → valid VAST returned`.
- `Integration: exchange timeout > tmax → fallback path selected`.

---

## Phase 8: Subscriptions, Premium Feeds, and Network P&L

### Purpose
Round out monetisation and reporting: listener subscriptions/memberships with private premium RSS feeds and Stripe billing, subscription revenue posted to the ledger, and the network-level P&L dashboard combining ad + subscription revenue against payout obligations and costs per show.

### Tasks

#### 8.1 — Subscription plans, subscriptions, premium entitlements

**What**: Network/show subscription plans, subscriber records billed via Stripe, and token-gated premium feeds.

**Design**:
- Tables `subscription_plans(show_id null, network_id, name, tier, price_amount, currency, billing_interval, features text[], stripe_price_id, is_active)`, `subscriptions(plan_id, subscriber_email, subscriber_name, status, stripe_subscription_id, current_period_*, cancelled_at)`, `premium_content(episode_id, required_tier, private_rss_token unique, ad_free bool, bonus_audio_url)`.
- Stripe Checkout for subscription sign-up; webhook (`invoice.paid`, `customer.subscription.updated/deleted`) updates `subscriptions.status` and posts `revenue_entries` (source_type `subscription`).
- Private feed: `GET /feeds/private/:token.xml` resolves the subscriber's entitlement and emits a feed including premium/bonus episodes and ad-free enclosures (reuses Phase 3 generator with a filter).

**Testing**:
- `Integration (mocked Stripe): invoice.paid webhook → subscription active + revenue_entry posted`.
- `Integration: subscription.deleted → status cancelled`.
- `Integration: private feed with valid token → includes premium episode; invalid token → 404`.
- `Integration: subscription revenue appears in royalty calc subscription_revenue bucket`.

#### 8.2 — Network P&L dashboard

**What**: Per-show and network-level P&L combining revenue, expenses, payout obligations, and net margin over time.

**Design**:
- `network_pnl_monthly` materialised view joining `revenue_entries` and `expense_entries` by `(network_id, show_id, month)` → `total_revenue, total_expenses, net_margin`; payout obligations surfaced from `royalty_calculations.distributable_amount` and `payouts`.
- `GET /networks/:nid/reports/pnl?from&to&groupBy=show|network` returns the series; outstanding (approved-but-unpaid) royalties shown as a liability line.
- Web P&L page: revenue-by-show stacked bars, margin line, payout-obligation table.

**Testing**:
- `Integration: ad + subscription revenue minus payouts + hosting expense → correct net_margin per show`.
- `Integration: approved-unpaid royalties appear as liability, not yet expense`.
- `Integration: P&L groupBy=network sums across shows`.

---

## Phase 9: Advertiser-Facing Reports + Export

### Purpose
Provide the advertiser-trust layer: branded delivery reports (booked vs. valid delivered, pacing, geo/app breakdown) exportable as PDF and CSV, plus a scoped advertiser-viewer portal — the credibility deliverable that lets networks demonstrate ROI.

### Tasks

#### 9.1 — Delivery report generation (PDF/CSV)

**What**: Generate per-campaign delivery reports from reconciliation + impression data.

**Design**:
- `buildDeliveryReport(campaignId, period): ReportModel` aggregating booked/delivered/valid impressions, delivery_pct, daily pacing series, geo and app/device breakdowns, and (IAB-credible) validity note.
- Worker job renders PDF (`@react-pdf/renderer`, network-branded header/logo) → S3; CSV export streams the daily series.
- `GET /networks/:nid/campaigns/:cid/report?format=pdf|csv&period=` returns/redirects to the artefact; advertiser-viewer permitted only for campaigns of their linked advertiser.

**Testing**:
- `Integration: report for a reconciled campaign → PDF generated, stored, URL returned`.
- `Integration: CSV export → header + one row per delivery day with valid/invalid columns`.
- `Integration: advertiser_viewer requests a campaign not linked to them → 403`.
- `Unit: ReportModel delivery_pct matches reconciliation`.

---

## Phase 10: AI-Native Features

### Purpose
Layer on the AI-native advantages that incumbents lack: advertiser-show matching via content embeddings, AI-suggested DAI marker placement, download anomaly/bot detection, and transcript brand-safety pre-screening. Each is additive and degrades gracefully if no LLM provider is configured.

### Tasks

#### 10.1 — Episode embeddings + advertiser-show matching

**What**: Embed episode transcripts and advertiser profiles, store in pgvector, and surface ranked advertiser-show matches.

**Design**:
- `content_embeddings(episode_id, embedding_model, embedding_vector vector(N), transcript_hash char(64), created_at)`; `ivfflat`/`hnsw` cosine index.
- Job embeds new transcripts; advertiser profile (industry, brand-safety prefs, target audience text) embedded similarly.
- `advertiser_show_matches(advertiser_id, show_id, match_score, match_reasons text[], generated_at, expires_at, unique(advertiser_id, show_id))`; score = cosine similarity blended with audience-overlap signal; LLM produces `match_reasons` rationale.
- `GET /networks/:nid/advertisers/:aid/matches` returns ranked shows.

**Testing** (mocked embeddings/LLM):
- `Unit: cosine ranking orders a clearly-aligned show above an unrelated one (fixed fixture vectors)`.
- `Integration: matches endpoint returns sorted results with reasons`.
- `Integration: no LLM configured → matches still returned by score, reasons empty`.

#### 10.2 — AI DAI marker suggestion

**What**: Suggest mid-roll insertion points from transcript structure and engagement drop-off.

**Design**:
- Job analyses transcript (topic-boundary detection) + completion/engagement curve (from analytics) to propose `ad_insertion_points` with `is_auto_detected=true` and `confidence_score`; surfaced for human approval (not auto-published).

**Testing**:
- `Unit: given transcript with clear topic boundaries → suggested offsets near boundaries`.
- `Integration: suggested points stored with is_auto_detected=true, awaiting approval`.

#### 10.3 — Download anomaly / bot detection

**What**: Flag suspicious download spikes / bot patterns beyond the static UA list.

**Design**:
- `brand_safety_scans`-style `download_anomalies(show_id, episode_id, metric_date, anomaly_score, signals jsonb, status)`: statistical baseline (rolling median/MAD) per episode-day flags outliers; clustered ip/ua patterns flagged. Surfaced in analytics UI; excluded counts reported separately to preserve IAB credibility.

**Testing**:
- `Unit: injected 10x spike day → flagged anomaly with high score`.
- `Unit: normal variance → not flagged`.

#### 10.4 — Transcript brand-safety pre-screening

**What**: Classify episode transcripts for brand-safety risk before publication.

**Design**:
- `brand_safety_scans(episode_id, scanned_at, risk_score numeric(5,4), flagged_categories text[], flagged_segments jsonb, model_version)`; LLM classifies against a category taxonomy; high risk warns on publish and is exposed for campaign brand-safety targeting/exclusion.

**Testing** (mocked LLM):
- `Integration: transcript flagged 'adult_content' → scan persisted; publish shows warning`.
- `Integration: clean transcript → low risk_score, no flags`.

---

## Phase 11: MCP Server + Public API Hardening

### Purpose
Expose the platform to AI tooling and finalise the public API for external developers: an MCP server with read-only tools, OpenAPI 3.1 polish, API keys/rate limiting, and an OWASP API Security Top-10 pass. After this phase the platform is integration-ready and security-reviewed.

### Tasks

#### 11.1 — MCP server

**What**: An MCP server exposing read-only tools over the platform API.

**Design**:
- Tools: `list_shows`, `get_show_analytics(showId, range)`, `get_royalty_calculation(showId, period)`, `get_campaign_delivery(campaignId, period)`, `network_pnl(range)`. Auth via a scoped API token mapped to a network membership; all tools enforce the same RBAC scoping as the HTTP API.
- Built with `@modelcontextprotocol/sdk`; stdio + HTTP transports.

**Testing**:
- `Integration: list_shows tool with a network-scoped token → only that network's shows`.
- `Integration: tool call with insufficient role → permission error`.

#### 11.2 — API keys, rate limiting, OpenAPI + security pass

**What**: Programmatic API keys, rate limiting, and an OWASP API Security Top-10 (2023) review.

**Design**:
- `api_keys(id, network_id, name, key_hash, scopes text[], last_used_at, expires_at, revoked_at)`; keys presented as `Authorization: Bearer pnm_…`, hashed at rest.
- `@fastify/rate-limit` (Redis store) per key/IP; sensitive financial endpoints lower limits.
- OWASP checklist enforced: object-level auth (network scoping everywhere — API1), no mass assignment (Zod allow-lists — API3/API6), function-level auth (RBAC matrix — API5), security misconfig (TLS 1.3, security headers via `@fastify/helmet`), inventory (OpenAPI 3.1 doc is the source of truth, `openapi:check` in CI).
- Finalise `openapi.json`: examples, error schemas, auth schemes documented.

**Testing**:
- `Integration: request with revoked API key → 401`.
- `Integration: exceed rate limit → 429 with Retry-After`.
- `Integration: attempt to set created_by/network_id via request body (mass assignment) → ignored, not persisted`.
- `Integration: security headers present (HSTS, X-Content-Type-Options, etc.)`.
- `Unit: openapi:check passes; every route has a documented error response`.

---

## Phase 12: Deployment, Observability, and Hardening

### Purpose
Make the platform operable: container images, one-command self-host bring-up, migrations on boot, backups, metrics/tracing, and the documented scaling path toward the Suggestion-4 polyglot architecture for high-impression-volume networks.

### Tasks

#### 12.1 — Containerisation + compose + CI release

**What**: Production Docker images for api/worker/web/mcp, compose for self-host, GitHub Actions release pipeline.

**Design**:
- Multi-stage Dockerfile with distinct `api`/`worker`/`web`/`mcp` targets; non-root user; healthchecks hitting `/healthz`.
- `docker-compose.yml` (postgres+pgvector, redis, minio, api, worker, web) with `.env.example`; an init container runs `db:migrate` before api starts.
- CI: lint → typecheck → test (PG/Redis service containers) → `openapi:check` → build+push images on tag.

**Testing**:
- `Integration: docker compose up → /healthz 200, /readyz 200, web served`.
- `CI: full pipeline green on a clean checkout`.

#### 12.2 — Observability + backups + scaling guide

**What**: Metrics, tracing, structured logs, financial-data backups, and a documented growth path.

**Design**:
- Prometheus metrics (`/metrics`): request latency/error rate, queue depth/job duration, impression ingest rate, payout success/fail counts. OpenTelemetry traces across api→worker. Logs as JSON with request-id correlation and PII/secret redaction.
- Backups: `pg_basebackup` + WAL archiving to S3 (point-in-time recovery for financial tables); documented 7-year retention for `audit_log`/`payouts`/`revenue_entries` per financial norms.
- Scaling guide (from Suggestion 1 & 4): at 10–100M impressions/month add read replicas + `pg_partman` partition automation + archive cold impression partitions to S3 Parquet; at 100M+ migrate `ad_impressions` to ClickHouse via CDC while PostgreSQL stays source of truth for reconciliation, and move advertiser-show/overlap graph queries to Neo4j (Suggestion 4). pgvector: IVFFlat→HNSW past ~1M embeddings.

**Testing**:
- `Integration: /metrics exposes request and queue gauges`.
- `Integration: restore from a base backup into a fresh PG → row counts match for financial tables`.
- `Smoke: trace context propagates api→worker for a royalty-calc job`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB core, config, server)   ─── required by everything
    │
Phase 2: Identity, RBAC, Catalogue                        ─── requires 1
    │
Phase 3: Feeds + Hosting Normalisation                    ─── requires 2
    │
    ├── Phase 4: Royalty Management ───────── requires 2,3 ─┐
    │       │                                               │
    │   Phase 5: Payment Disbursement ─────── requires 4    │
    │                                                       │
    └── Phase 6: Analytics + IAB Dedup ────── requires 3    │
            │                                               │
        Phase 7: Ad Insertion / Campaigns / Reconciliation ─┘ requires 6 (and posts revenue → 4)
            │
        Phase 8: Subscriptions + P&L ──────── requires 4,7
            │
        Phase 9: Advertiser Reports ───────── requires 7
            │
        Phase 10: AI Features ─────────────── requires 6,7 (embeddings, markers, brand-safety, anomaly)
            │
        Phase 11: MCP + API Hardening ─────── requires 2–9
            │
        Phase 12: Deployment + Observability ─ requires all (incremental from Phase 1)
```

**Parallelism opportunities:**
- After Phase 3: **Phase 4 (Royalty)** and **Phase 6 (Analytics/IAB)** can be developed concurrently — independent domains.
- **Phase 5 (Payments)** parallels **Phase 6** once Phase 4 is underway.
- After Phase 7: **Phase 8 (Subscriptions/P&L)**, **Phase 9 (Reports)**, and **Phase 10 (AI)** can be developed concurrently.
- **Phase 12** is incremental — Docker/compose/CI scaffolding should be stood up during Phase 1 and extended each phase, with full observability/scaling work landing last.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pnpm test`), including the listed happy-path and edge-case scenarios.
3. Linting and formatting pass (`pnpm lint`, Prettier check).
4. Type checking passes (`pnpm typecheck` / `tsc --noEmit`) with no `any` introduced in financial code paths.
5. Drizzle migrations are created, apply cleanly on an empty database, and the schema check passes.
6. The feature works end-to-end (verified by an integration or Playwright E2E test where user-facing).
7. New configuration options are added to `.env.example` and documented.
8. New/changed API endpoints appear in the regenerated `openapi.json`, and `openapi:check` passes in CI (no drift).
9. New write paths emit `audit_log` entries where they mutate financial or catalogue state.
10. All external-credential and PII storage uses the encryption helper; no secrets logged (redaction verified).
11. Docker build succeeds for affected targets; `docker compose up` yields healthy `/healthz` and `/readyz`.
12. OWASP-relevant endpoints enforce network scoping (object-level auth) and the RBAC permission matrix.
```
