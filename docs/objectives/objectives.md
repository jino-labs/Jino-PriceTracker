# Jino-PriceTracker: Objectives

High-level objective definitions for the Price Tracker application. Each objective is standalone. Technology stack is intentionally undefined except where a structural choice was made (containerization, JWT, message queue).

---

## Objective 1: Data Acquisition & Scraping

**Core Definition:** Build scraping system to extract product prices and stock from target websites, scheduled for high-volume acquisition with fallback deep-search capability.

**Scraping Modes:**
1. **Scheduled (primary):** Run on interval against predefined products list. High-throughput, known URLs/targets.
2. **Deep search (fallback):** Triggered when product not in current dataset. Performs broader scraping to locate/discover the product. Heavier, on-demand.

**Data Extraction (per product):**
- Product name
- Price (raw value)
- Currency (must be captured — part of the equation)
- Stock / availability status (in stock / out of stock / quantity if available)
- Product URL
- Timestamp
- Variant identifier (if applicable)

**Variants Handling:**
- Treated as a **relation** to the parent product (size, color, etc.)
- Variant = child entity linked to base product
- Each variant tracks its own price and stock

**Price / Currency Handling:**
- Capture currency per scrape (USD, EUR, etc.)
- Normalization deferred to Objective 2 (no conversion at scrape)

**Protocol Support:**
- HTTPS preferred
- HTTP allowed when necessary (not excluded)

**Technical Concerns:**
- Rate limiting & polite scraping (respect robots.txt)
- Handle dynamic (JS-rendered) vs static content
- Retry logic, timeout handling
- Detect/handle blocking (user-agent, IP rotation if needed)
- Graceful handling of site structure changes

**Success Criteria:**
- >99% extraction accuracy
- <10% failure rate per scheduled run
- Scheduled runs do not trigger blocks
- Deep search locates products not in the dataset
- Stock changes captured as events feeding Objective 5

---

## Objective 2: Data Processing & Validation

**Core Definition:** Transform raw scraped data into clean, validated, structured records ready for monitoring and storage. Normalize all prices to a base currency.

**1. Parsing & Normalization**
- Parse raw price strings → numeric value
- Normalize decimal/thousands separators (1.000,50 vs 1,000.50)
- Extract & standardize currency code (ISO 4217)
- Store price as **decimal value** (avoids currency multipliers scattered across the app — convert only where needed)
- Trim/clean product names (whitespace, encoding artifacts)

**2. Base Currency & Exchange Rates**
- Normalize all prices to a **base currency** on ingest
- **Exchange rate source = public FX API** (e.g. exchangerate.host, ECB feed) — NOT scraped
- Store: base-currency price + original price + original currency + FX rate used + rate timestamp
- Cache FX rates and refresh on a schedule; pin the rate per conversion for audit

**3. Validation**
- Non-null price & currency
- Valid URL, required fields present
- **Outlier handling:** price jumps >20% → **stored but marked suspicious** (never ignored/dropped)
- Reject only truly invalid data ($0, negative, malformed)

**4. Deduplication & Identity**
- **Identity key = store product ID** (mostly unique per store)
- Fallback: if store ID is not unique → explore store-specific pattern to derive identity
- Match scraped item to existing product via store ID
- Link variants to parent product (relation from Objective 1)

**5. Persistence**
- Store/update product record
- Append price point to history (base price, original price, currency, FX rate, stock, timestamp, suspicious flag)
- **Append-only** price log — never overwrite history

**Data Model (entities):**
- **Product** — store ID (identity), name, metadata
- **Variant** — child of Product, own price/stock track
- **PriceHistory** — append-only (product/variant ref, base price, original price, currency, FX rate, stock, timestamp, suspicious flag)

**Success Criteria:**
- 100% of scraped data passes through validation
- Identity resolved via store ID (zero false duplicates)
- Suspicious outliers flagged, not lost
- Append-only history integrity maintained

---

## Objective 3: User Management & Authentication

**Core Definition:** Enable users to register, authenticate via JWT, and manage profiles, multiple watch lists, and layered alert rules.

**1. Registration & Identity**
- Email-based signup + email verification
- **Social login = stored connection** (link external provider account to the user; persist the connection, not just one-time auth)

**2. Authentication**
- Password hashing (bcrypt/argon2)
- **JWT via cookies** (no server session)
- Login/logout, password reset (email flow)
- Token expiry/refresh strategy
- Cookie security (httpOnly, secure flags)

**3. Authorization & Roles**
- Strict data isolation (user accesses only own data)
- **Role management built in now** (even if only "user" is used today — schema/infra ready for future roles like admin)

**4. Profile Management**
- Edit name, email, password
- Account + data deletion handled under Objective 4 (compliance)

**5. Watch Lists**
- **Multiple watch lists per user**
- Link user → lists → tracked products (relations)

**6. Alert Rules (two layers)**
- **Per-product rules:** specific ("this product under X price")
- **Global / watchlist rules:** apply across a list
- Feeds the monitoring engine (Objective 5) and notification channels (Objective 6)

**Success Criteria:**
- Account creation <30s
- JWT cookie auth secure (httpOnly, secure)
- Password reset functional
- Strict per-user data isolation
- Role system extensible without rework

---

## Objective 4: Compliance (LOPDGDD / GDPR)

**Core Definition:** Meet Spanish data protection law (LOPDGDD, aligned with EU GDPR) for all user data handling. Cross-cutting; kept standalone for auditability.

**User Rights (data subject rights):**
- **Right to erasure** — full account + all user data deletion on request
- **Right to access** — user can view all data held about them
- **Right to portability** — export user data in machine-readable format (JSON/CSV)
- **Right to rectification** — edit/correct personal data
- **Right to object/restrict** — pause processing (e.g. stop tracking, mute without delete)

**Consent Management:**
- Explicit consent at signup (terms, privacy policy)
- Consent tracking (what was consented, when, which version)
- Granular consent (marketing emails separate from service alerts)
- Withdraw-consent mechanism

**Data Handling:**
- Data minimization (store only what is needed)
- Encryption at rest & in transit
- Retention policy (how long inactive data is kept before purge)
- Breach notification process (72h rule)

**Transparency:**
- Privacy policy (what data, why, how long, third parties)
- Cookie consent (JWT cookies + any tracking)
- Record of Processing Activities (RoPA)

**Third-Party / Social Connections:**
- Lawful basis for storing social login connections
- Document data shared with external providers

**Success Criteria:**
- All data subject rights exercisable via the app
- Consent logged & auditable
- Data export/deletion functional & complete
- Privacy policy + cookie consent present
- Breach process documented

---

## Objective 5: Price Monitoring & Alerting Logic

**Core Definition:** Evaluate price/stock history against user-defined rules and trigger alerts via an event-driven pipeline.

**1. Alert Rule Types** (per-product + global, from Objective 3)
- Absolute threshold ("under X price")
- Percentage drop
- Drop vs previous / historical average / lowest
- **Stock / availability change**
- Target reached (lowest ever)

**2. Evaluation Engine**
- Compare latest price/stock point vs rule conditions
- **Suspicious prices (Objective 2 flag): still evaluated, but the alert carries a suspicious annotation**
- Run per-product + watchlist-global rules
- Stock changes evaluated alongside price

**3. Currency**
- Prices stored in base currency (Objective 2)
- **Convert to the user-selected currency** for evaluation/display

**4. Scheduling — Event-Driven**
- **Message queue** drives evaluation (new price/stock event → enqueue → evaluate)
- Decoupled from scraping; reactive, not batch polling

**5. Re-Arm Logic**
- **Alert log** tracks last alert state per rule
- Re-alert **only if the price changed** since the last alert (check the log to suppress repeats on unchanged price)
- Price unchanged → no re-fire

**6. Alert State & History**
- Record trigger (rule, product, old/new price, stock, currency, suspicious flag, timestamp)
- Alert log = source of truth for re-arm + audit
- Per-user queryable history

**7. Hand-off**
- Triggered alert → Notification System (Objective 6)

**Success Criteria:**
- Alerts trigger within 5 min via the queue
- Correct evaluation (price + stock)
- Suspicious prices annotated, not dropped
- No duplicate alerts (alert-log re-arm)
- Currency converted to user preference

---

## Objective 6: Notification System

**Core Definition:** Deliver triggered alerts (Objective 5) to users via preferred channels, reliably, respecting consent (Objective 4) and preferences (Objective 3).

**1. Channels**
- Email
- Push (web)
- In-app
- **SMS excluded**

**2. Consumption**
- Consume from the message queue (Objective 5 emits → notification consumes)
- Decoupled delivery

**3. User Preferences** (from Objective 3)
- Per-channel opt-in/out
- Per-watchlist / per-product channel choice
- Consent check (Objective 4) — service vs marketing separation

**4. Batching**
- **Batch alerts** rather than one-per-event
- **Include historic context** in the batch (price trend, prior values)
- Digest groups multiple triggers into a single notification

**5. Throttling**
- **Quiet hours** (user-set mute window)
- **Frequency limit** (max alerts per period, user preference)

**6. Delivery Reliability**
- **Retry on failure** (backoff)
- Dead-letter for permanent failures
- Track sent / delivered / failed only — **no open tracking** (fire-and-forget; user's data privacy)

**7. Content**
- Payload: product, old/new price, user-selected currency, suspicious flag, historic trend, link
- Templating per channel
- **Localized to the user's preferred locale**

**Success Criteria:**
- >99% email delivery within 1 min
- Channel preferences honored
- Consent respected (no alerts to opted-out users)
- Failed deliveries retried, not lost
- Batched alerts with historic context, in user locale

---

## Objective 7: User Interface & Dashboard

**Core Definition:** Web interface for users to manage watch lists, products, alert rules, view price history, and configure account/preferences. Web only, real-time via websockets.

**1. Auth Screens**
- Register, login, logout, password reset
- Social login connect (Objective 3)
- Email verification flow

**2. Dashboard (home)**
- Overview of watch lists + tracked products
- Current prices (user-selected currency)
- Recent alerts feed
- Stock status indicators

**3. Product Management**
- Add product (by URL or search/deep-scrape trigger, Objective 1)
- Remove product
- Assign product to watch list(s)
- View product detail: current price, stock, variants

**4. Watch Lists**
- Create/edit/delete multiple lists (Objective 3)
- Move products between lists
- Global rules per list

**5. Price History Visualization**
- Chart/graph price over time
- Suspicious points annotated (Objectives 2/5)
- Variant comparison
- Currency in user preference

**6. Alert Rule Config**
- Per-product rules (threshold, % drop, stock)
- Global / watchlist rules
- View/edit/delete rules

**7. Settings**
- Profile (name, email, password)
- Notification preferences (channels, quiet hours, frequency, locale — Objective 6)
- Currency preference
- Consent management (Objective 4)
- Data export + account deletion (Objective 4)

**8. Public Landing Page**
- Marketing entry, value proposition, signup CTA
- Pre-auth public access

**9. Admin UI (stub)**
- Placeholder for future role-based admin (Objective 3)
- Scaffold only, not functional in v1

**Cross-cutting:**
- **Web only** (no native app in v1)
- **Real-time via websockets** (live price/stock/alert updates push to dashboard)
- Responsive (mobile-friendly)
- Accessible (WCAG baseline)
- Localized (user locale)

**Success Criteria:**
- Load time <2s
- Add-product workflow <1 min
- Charts render smoothly
- Real-time updates render without manual refresh
- All user-facing Objective 3/4/6 controls present

---

## Objective 8: Testing & Quality Assurance

**Core Definition:** Comprehensive test coverage across all components ensuring correctness, reliability, security, and compliance.

**1. Unit Tests**
- Scraper parsers (Objective 1)
- Price normalization, currency conversion, FX (Objective 2)
- Validation + outlier/suspicious logic (Objective 2)
- Identity/dedup resolution (Objective 2)
- Alert rule evaluation engine (Objective 5)
- Re-arm / alert-log logic (Objective 5)

**2. Integration Tests**
- Scrape → process → store flow
- Event → queue → evaluate → notify pipeline (Objectives 5/6)
- Auth flow (JWT cookies, Objective 3)
- FX API integration (Objective 2)

**3. End-to-End Tests**
- Full user journeys: register → add product → price drop → alert → notification
- Watch list + rule config flows
- Data export + account deletion (Objective 4)

**4. Specialized Testing**
- **Security:** auth, JWT, data isolation, injection
- **Compliance:** erasure completeness, consent logging (Objective 4)
- **Scraper resilience:** mock fixtures primary + periodic live smoke checks (catch site structure changes)
- **Currency:** conversion correctness across locales

**5. Test Data**
- **Seed fixtures** (deterministic, repeatable)

**6. Performance Testing**
Baselines defined now; full load suite implemented post-MVP (do not block v1). Tooling: k6 or Locust, run nightly in CI (not per-commit).
- **Scrape throughput** — products/min per scheduled run (set baseline)
- **Queue latency** — event → alert evaluated, target <5 min; burst-test (1000 events)
- **API response** — endpoints <500ms p95 under load
- **Websocket** — 1000 concurrent connections without degradation
- **DB query** — price-history lookups stay fast as history grows (test with seeded large dataset)

**7. CI/CD Integration**
- Automated test run on commit/PR
- Block merge on failure
- Coverage reporting

**Success Criteria:**
- **Code coverage: 90% minimum, 100% target**
- All tests pass pre-deploy
- New code requires tests
- E2E covers critical paths
- Security + compliance validated
- Performance baselines established

---

## Objective 9: Deployment & Operations

**Core Definition:** Deploy to production with infrastructure, monitoring, logging, backups, and recovery for reliable operation. Containerized for flexibility.

**1. Infrastructure**
- **Containerized (Docker)** — portable across cloud or self-host
- **Cloud vs self-host: deferred** — containerization keeps it flexible, decide later without rework
- App server(s), database, message queue (Objective 5), websocket server (Objective 7)
- FX rate cache (Objective 2)
- Scraper workers (scheduled + deep-search, Objective 1)

**2. Deployment Pipeline**
- Staging → production environments
- Automated deploy (CI/CD, ties to Objective 8)
- Rollback capability
- DB migrations handling
- Secrets management (FX API keys, JWT secret, social login credentials)

**3. Monitoring & Ops Alerting**
- System health (CPU, memory, disk)
- Scraper success/failure rates (Objective 1)
- Queue depth/lag (Objective 5)
- Notification delivery rates (Objective 6)
- **Ops alerting** — system-health alerts to operators (you/team), distinct from user price alerts. Channel: email/Slack v1, formal paging (PagerDuty) later.

**4. Logging**
- Centralized logs (app, scraper, queue, notifications)
- No PII/credentials in logs (Objective 4 compliance)
- Searchable/queryable
- Audit logs (alert history, data deletions)

**5. Backup & Recovery**
- **Daily DB backup, 30-day retention**
- **Quarterly restore test**
- Disaster recovery procedure
- Price history is irreplaceable (cannot re-scrape the past) → zero-loss priority

**6. Scaling**
- Scraper worker scaling (volume growth)
- Queue consumer scaling
- DB scaling (price history grows continuously)

**Success Criteria:**
- 99.5%+ uptime
- Deploy <30 min
- Error detection <5 min
- Recovery <10 min
- Zero data loss on failure
- No secrets/PII in logs
