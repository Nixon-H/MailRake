# TECHNICAL REFERENCE

This document contains the detailed, implementation-level reference for the Advanced Email Scraper. It is intended for developers maintaining or extending the codebase. For user-facing documentation, see [README.md](./README.md).

1. [Complete Data Flow](#1-complete-data-flow)
2. [SMTP Verification Protocol](#2-smtp-verification-protocol)
3. [Crawl Lifecycle (Per-Domain)](#3-crawl-lifecycle-per-domain)
4. [Complete Test Suite Class Listing](#4-complete-test-suite-class-listing)
5. [Email Deobfuscation & Extraction Methods](#5-email-deobfuscation--extraction-methods)
6. [Full Configuration Reference](#6-full-configuration-reference)
7. [History DB Schema](#7-history-db-schema)
8. [Concurrency & Thread Safety Reference](#8-concurrency--thread-safety-reference)
9. [All Module-Level Constants](#9-all-module-level-constants)
10. [Version History / Changelog](#10-version-history--changelog)

---

## 1. COMPLETE DATA FLOW

```
                      ┌─────────────────────────────────────┐
                      │          INPUT SOURCES              │
                      │  .txt / .csv / Google Sheet / --url │
                      └──────────┬──────────────────────────┘
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │   normalize_domain() :7877    │
                  │   normalize_domain_list()     │
                  │   :7947                       │
                  │   ├── Lowercase + strip       │
                  │   ├── Strip scheme/path/port  │
                  │   ├── Strip trailing dot      │
                  │   ├── Reject IPs / localhost  │
                  │   └── Deduplicate             │
                  └──────────┬───────────────────┘
                             │
                             ▼
            ┌─────────────────────────────────────┐
            │         PER-DOMAIN SETUP             │
            │  ┌──────────────────────────────┐    │
            │  │ _get_robots_rules() :4023     │    │
            │  │   ├── robots.txt fetch       │    │
            │  │   ├── LRU cache (1024 entry) │    │
            │  │   └── Single-flight lock     │    │
            │  ├──────────────────────────────┤    │
            │  │ _seed_sitemap_urls() :4107    │    │
            │  │   ├── sitemap.xml fetch      │    │
            │  │   ├── Parse XML              │    │
            │  │   └── Filter + queue URLs    │    │
            │  └──────────────────────────────┘    │
            └──────────────────┬──────────────────┘
                               │
                               ▼
         ┌───────────────────────────────────────────┐
         │         PHASE 1: PRIORITY SCAN             │
         │  _execute_crawl_phase("Priority Scan")     │
         │  :5942                                     │
         │                                            │
         │  Proactive paths (/contact, /about, ...)   │
         │  + sitemap URLs + homepage crawl           │
         │  ThreadPoolExecutor (max_threads workers)  │
         │  Adaptive concurrency (FIRST_COMPLETED)    │
         └──────────────────┬────────────────────────┘
                            │
                            ▼
         ┌───────────────────────────────────────────┐
         │         PHASE 2: DEEP CRAWL                │
         │  _execute_crawl_phase("Deep Crawl")        │
         │  :5942                                     │
         │                                            │
         │  BFS link extraction from each page        │
         │  Same-domain only, depth-limited           │
         │  Content-hash dedup (skip identical pages)  │
         │  Rate-limiter non-blocking fast-fail       │
         └──────────────────┬────────────────────────┘
                            │
                            ▼
              ┌──────────────────────────┐
              │    PER-PAGE PIPELINE      │
              │  _process_page() :5639    │
              │                          │
              │  1. Rate-limiter probe   │
              │  2. _fetch_url() :5054   │
              │     ├── Tier-1 curl_cffi │
              │     ├── Tier-2 camoufox  │
              │     └── Tier-3 fallback  │
              │  3. Link extraction      │
              │  4. Email extraction     │
              │     _extract_emails()    │
              └──────────────────────────┘
                            │
                            ▼
              ┌──────────────────────────┐
              │   EMAIL EXTRACTION        │
              │  _extract_emails() :4646  │
              │                          │
              │  ├── Regex from HTML     │
              │  ├── mailto: links       │
              │  ├── data- attrs         │
              │  ├── JSON-LD             │
              │  ├── SVG text            │
              │  ├── Base64 decode       │
              │  ├── [at]/[dot] deobfus  │
              │  ├── Cloudflare XOR      │
              │  ├── JS concat           │
              │  └── ROT13 decode        │
              │                          │
              │  Filters:                │
              │  ──────────────────────  │
              │  Throwaway domains      │
              │  Placeholder emails     │
              │  Artefact prefixes      │
              │  Bundle paths           │
              │  APM DSN @-domains      │
              │  Asset extensions TLD   │
              │  Blacklist substrings   │
              └──────────────────────────┘
                            │
                            ▼
         ┌──────────────────────────────────────┐
         │           SAVE RESULTS                │
         │  save_results()                       │
         │  ┌───────────────────────────────┐    │
         │  │ JSON ──► _emails.json         │    │
         │  │         _crossdomain.json     │    │
         │  ├───────────────────────────────┤    │
         │  │ CSV  ──► _emails.csv          │    │
         │  │         _report.csv           │    │
         │  │         _phones.csv           │    │
         │  │         _verified_emails.csv  │    │
         │  ├───────────────────────────────┤    │
         │  │ XLSX ──► combined workbook    │    │
         │  │         (green/red/amber fmt) │    │
         │  ├───────────────────────────────┤    │
         │  │ metadata.json                 │    │
         │  └───────────────────────────────┘    │
         └──────────────────┬───────────────────┘
                            │
                            ▼
         ┌──────────────────────────────────────┐
         │       SMTP VERIFIER                   │
         │  AsyncEmailVerifier :1570             │
         │                                       │
         │  ├── DNS MX lookup                    │
         │  ├── _smtp_handshake_once() :1797     │
         │  ├── Accept-all detection             │
         │  ├── Greylist retry                   │
         │  └── Confidence scoring               │
         └──────────────────┬───────────────────┘
                            │
                            ▼
         ┌──────────────────────────────────────┐
         │       HISTORY DB                      │
         │  EmailHistoryDB :6662                 │
         │                                       │
         │  ├── emails table (INSERT OR BUMP)   │
         │  ├── verifications table (append)     │
         │  └── runs table (start/finish)        │
         └──────────────────────────────────────┘
                            │
                            ▼
         ┌──────────────────────────────────────┐
         │       WEBHOOK (optional)              │
         │  POST JSON run summary to URL         │
         │  SSRF-protected (reject private IPs)  │
         └──────────────────────────────────────┘
```

### Step-by-step walkthrough

| # | Step | Function(s) | Lines |
|---|------|------------|-------|
| 1 | Read input file (.txt/.csv) or Google Sheet | `load_domains_from_path()`, `get_domains_from_google_sheet()` | ~8190–8577 |
| 2 | Normalize each domain (lowercase, strip scheme/path, deduplicate) | `normalize_domain()` | 7877 |
| 3 | Normalize list, remove dupes | `normalize_domain_list()` | 7947 |
| 4 | Setup output directory (`output/YYYY-MM-DD/<run_number>/`) | `setup_output_directory()` | ~7250 |
| 5 | Initialize `AdvancedEmailScraper` with domains + settings | `AdvancedEmailScraper.__init__()` | 3711 |
| 6 | Pre-seed robots.txt rules for all domains | `_get_robots_rules()` | 4023 |
| 7 | Fetch + parse sitemap.xml for each domain | `_seed_sitemap_urls()` | 4107 |
| 8 | Phase 1: Priority Scan — visit high-value pages first | `_execute_crawl_phase("Priority Scan")` | 5942 |
| 9 | For each page: fetch, parse, extract emails/phones | `_process_page()` → `_fetch_url()` → `_extract_emails()` | 5639→5054→4646 |
| 10 | Phase 2: Deep Crawl — BFS from homepage, same-domain only | `_execute_crawl_phase("Deep Crawl")` | 5942 |
| 11 | Per-page: extract links → filter → queue | `_process_page()`, `_is_valid_crawl_url()` | 5639, 4457 |
| 12 | Content-hash dedup (skip identical pages already seen) | `_record_response_seen()` | 4212 |
| 13 | Save crawl results (JSON, CSV, XLSX) | `save_results()` | ~4833 |
| 14 | SMTP verification (async, per-email) | `run_verifier_standalone()` → `AsyncEmailVerifier.run()` | 7820→2246 |
| 15 | Write verification results to CSV | `_log_to_csv_file()` | 4833 |
| 16 | Record to history DB | `EmailHistoryDB.record_email()`, `record_verification()` | 6758, 6785 |
| 17 | Generate metadata.json | `_write_metadata_json()` | ~12559 |
| 18 | POST webhook (if configured) | `send_webhook()` | ~6917 |
| 19 | Continuous monitoring loop (--url mode): wait interval → goto step 1 | `main()` loop | 21584 |

---

## 2. SMTP VERIFICATION PROTOCOL (STEP BY STEP)

### 1. Entry point — `AsyncEmailVerifier.run()` :2246

The verifier accepts a list of email tasks, a concurrency cap, and a mode (`direct` or `auth`). It uses `asyncio.Semaphore` to throttle simultaneous connections.

### 2. `verify_email()` :2088

Dispatches to `_verify_smtp_direct()` or `_verify_smtp_auth()` based on mode.

### 3. DNS MX lookup (`get_mx_records()` :~1630)

```python
resolver = aiodns.DNSResolver()
records = await resolver.query(domain, 'MX')
```
- Sorted by MX preference (lowest priority first)
- TTL = 300s (cached via `_resolver_lock` + per-instance resolver reuse)
- If MX lookup fails → "no MX records"

### 4. Port 25 → 587 fallback (`_verify_smtp_direct()` :~1700)

For each MX host:
1. Try port 25 first (standard SMTP)
2. If 25 fails (timeout/refused), fall back to port 587 (submission)
3. If 587 returns 530 "Authentication required" → skip (not a mailbox verdict)

### 5. `_smtp_handshake_once()` :1797

Performs a single SMTP MAIL/RCPT probe against one `host:port`:

```
1. Connect (TCP) ──► 2. EHLO / HELO ──► 3. STARTTLS (opportunistic)
    ──► 4. EHLO again ──► 5. MAIL FROM ──► 6. RCPT TO ──► 7. QUIT
```

**Step details:**
- **Connection**: TCP connect with 10s timeout. If proxy configured, connects via proxy socket first, rotates up to 3 proxies on failure.
- **EHLO/HELO**: Prefers EHLO (ESMTP), falls back to HELO.
- **STARTTLS**: Always attempted opportunistically. If `smtp_verify_tls` is enabled and STARTTLS fails → "STARTTLS negotiation failed". Otherwise continues in plaintext.
- **MAIL FROM**: `info@example.com` (configurable). 5xx = permanent reject → "undeliverable". 4xx = transient → walks to next MX host. 530 on port 587 = auth-required → non-definitive.
- **RCPT TO**: 2xx/3xx = accepted → "deliverable". 4xx = transient → walks to next. 5xx = rejected → "undeliverable".
- **QUIT**: Always called in inner finally; `.close()` in innermost finally to prevent FD leak.

### 6. Response Classification

| Class | Description | Trigger |
|-------|-------------|---------|
| **deliverable** | Mailbox exists | At least one MX returns 2xx/3xx on RCPT TO |
| **undeliverable** | Mailbox does not exist | All MX hosts return 5xx, or MAIL FROM permanently rejected |
| **accept-all** | Catch-all domain | Domain accepts ALL RCPT TO (incl. random `jkwqheoiqwhr@domain`) |
| **greylisted** | Temporary deferral | RFC 6647 detection via `is_greylist_response()` — matches `450 4.7.1`, `try again later`, `temporarily`, etc. |
| **unknown** | No definitive verdict | All MX returned 4xx on RCPT TO (non-greylist transient) |
| **unreachable** | Connection failed | All hosts unreachable (timeout/refused/firewalled) |
| **no MX records** | DNS failure | MX lookup returned empty |

### 7. Accept-All Detection (`check_accept_all()` :~2119)

1. Generate random email: `jkwqheoiqwhr@domain`
2. Run full SMTP probe
3. If RCPT TO accepted (2xx/3xx) → domain is accept-all
4. Cache result for 24h (LRU, 10k cap)

### 8. Confidence Scoring (`compute_email_confidence()`)

Score 0–100 based on:
- `+40` if deliverable
- `+20` if not disposable domain
- `+15` if not role address
- `+15` if has MX records
- `+10` if not privacy domain

### 9. Greylist Retry Logic

- Detected in `_verify_smtp_direct()` via `is_greylist_response()` checking server text for `450 4.7.1`, `greylist`, `try again later`, etc.
- Email marked as "greylisted" (not "unknown")
- `--reverify-unknown` pass retries greylisted emails after `greylist_retry_delay_seconds` (default 60s) at half concurrency

### 10. Accept-All Cache

- `_ACCEPT_ALL_CACHE_MAX = 10,000` entries
- `_ACCEPT_ALL_CACHE_TTL_SECONDS = 86,400` (24h)
- Backed by `OrderedDict` with LRU eviction
- Thread-safe via `threading.Lock`; async wrappers offload to thread executor via `run_in_executor`
- Per-domain async lock prevents N simultaneous accept-all probes for same domain

### 11. Per-Domain Rate Limiting

- `DomainRateLimiter` :765 — token-bucket per domain at 1/s default
- `rate_limiter.acquire(domain, max_wait=0.5)` — non-blocking fast-fail in `_process_page`

---

## 3. CRAWL LIFECYCLE (PER-DOMAIN)

### Step 1: Domain Normalization
- **Function**: `normalize_domain()` :7877
- Lowercase, strip scheme (`http://`, `https://`), strip WWW, strip path/query/fragment/port, strip trailing dot, reject IP literals, reject single-label (localhost)

### Step 2: robots.txt Fetch + Parse + Cache
- **Function**: `_get_robots_rules()` :4023
- Fetches `https://domain/robots.txt` then `http://domain/robots.txt`
- Parses via `_RobotsRules.parse()` :658 (applies Disallow/Allow rules, extracts Crawl-delay)
- **Cache**: LRU dict (1024 entries), TTL per domain (~300s), single-flight per-domain `RLock` prevents N workers fetching simultaneously

### Step 3: Sitemap Discovery + Parse
- **Function**: `_seed_sitemap_urls()` :4107
- Fetches `sitemap.xml`, `sitemap_index.xml`, `sitemap.xml.gz`
- Parses XML, filters to same-domain URLs, applies eTLD+1 protection
- Prepend URLs to crawl queue (bounded by `_SITEMAP_MAX_RECURSION` and `sitemap_max_urls_per_domain`)

### Step 4: Priority Scan (Proactive Paths)
- **Function**: `_execute_crawl_phase("Priority Scan")` :5942
- Queues proactive paths from `ALL_PROACTIVE_PATHS`:
  - Primary: `/contact`, `/contact-us`, `/about`, `/team`, `/security`, `/bug-bounty`, `/.well-known/security.txt`
  - Secondary: `/privacy`, `/terms`, `/legal`, `/compliance`, `/cookie-policy`

### Step 5: Homepage Fetch + Parse
- **Function**: triggered during priority scan via `_process_page()`
- Each domain's homepage (`https://domain/`) is fetched as part of the priority scan phase
- Single-flight via `homepage_analysis_done` set to prevent N workers fetching homepage simultaneously

### Step 6: Link Extraction + Filtering
- **Function**: `_process_page()` :5639 (link extraction section)
- Extracts all `<a href="...">` links from parsed HTML
- **Filtering via `_is_valid_crawl_url()` :4457**:
  - Same-domain only (`_is_same_domain()` :4410, eTLD+1 comparison)
  - Depth-limited (max_depth increases phase-by-phase)
  - Blocklisted extensions: `.jpg`, `.png`, `.pdf`, `.zip`, `.css`, `.js`, etc.
  - Blocklisted path fragments: `/wp-content/`, `/uploads/`, `/assets/`, `/cdn-cgi/`
  - Not already visited
  - Not already queued (O(1) check via `queued_urls` set)

### Step 7: Deep Crawl Queue Management
- **Function**: `_execute_crawl_phase("Deep Crawl")` :5942
- Continuous-feed `ThreadPoolExecutor` with adaptive concurrency
- Top-up to `target` in-flight futures; wake on `FIRST_COMPLETED`
- Per-domain general_links_discovered cap (5k URLs per domain)
- Rate-limit fast-fail re-queue (max 2 re-queues per URL)

### Step 8: Per-Page Pipeline
- **Function**: `_process_page()` :5639
  1. Rate-limiter non-blocking probe (0.5s max wait)
  2. `_fetch_url()` :5054 — Tier-1 (curl_cffi Chrome 120), Tier-2 (camoufox stealth Firefox), Tier-3 (cloudscraper fallback)
  3. Parse HTML with BeautifulSoup (lxml preferred)
  4. Extract emails via `_extract_emails()` :4646
  5. Extract phone numbers via regex
  6. Extract links → push to crawl queue

### Step 9: Content-Hash Deduplication
- **Function**: `_record_response_seen()` :4212
- Computes SHA-256 hash of response body
- Skips pages with identical content already seen (within same run cycle)
- Maintains `content_hashes` dict + `content_hash_order` deque

### Step 10: Rate Limiter Token Consumption
- **Function**: `DomainRateLimiter.acquire()` :817
- Token-bucket per domain (default 1/s, overridden by robots.txt Crawl-delay)
- Non-blocking 0.5s probe in `_process_page` before fetch
- Full `max_wait=30s` inside `_fetch_url`

### Step 11: Outage Detection
- **Global** (not per-domain):
  - `consecutive_connection_failures` counter increments on every network error
  - When counter ≥ 20 → `_outage_in_progress = True`
  - `_network_available` Event is cleared → all workers pause
  - Recovery: periodic `_probe_network()` checks succeed → event set → workers resume
  - URLs that failed during outage are tracked in `urls_failed_during_outage_check`

### Step 12: Graceful Shutdown Handling
- **Signal handlers**: SIGTERM, SIGHUP set `_SHUTDOWN_REQUESTED = True`
- `_execute_crawl_phase()` checks flag before submitting new URLs
- Pending futures complete naturally
- Partial results saved to disk
- History DB closed, PID file removed, temp files cleaned up

---

## 4. COMPLETE TEST SUITE CLASS LISTING

**211 test classes** across 8 categories + smoke tests:

### Category 1: Core Component Tests

| Class | Line | Methods |
|-------|------|---------|
| `TestPoolSession` | 3142 | 4 |
| `TestRateLimiterDeep` | 15064 | 3 |
| `TestRateLimiterExtraEdgeCases` | 12871 | 8 |
| `TestDomainRateLimiter` | 11349 | 2 |
| `TestDomainRateLimiterReuseEdgeCases` | 13252 | 5 |
| `TestAutotuneConcurrency` | 11424 | 2 |
| `TestAutotuneConcurrencyDeep` | 15276 | 5 |
| `TestAdaptiveDepthPromotion` | 11775 | 2 |
| `TestStealthSession` | 11382 | 2 |
| `TestStealthHeaders` | 11404 | 1 |
| `TestContentHashDedup` | 11682 | 2 |
| `TestContentHashDedupBehaviour` | 15531 | 2 |
| `TestShutdownHandler` | 12008 | 1 |
| `TestInternetConnectionCheck` | 11910 | 3 |
| `TestFormatDuration` | 11981 | 1 |
| `TestFormatDurationDeep` | 15246 | 2 |
| `TestFormatDurationExtraEdgeCases` | 13220 | 1 |
| `TestOutageSingleFlight` | 12602 | 1 |
| `TestMetadataJson` | 12559 | 1 |
| `TestResultDataclass` | 12067 | 1 |
| `TestResultDataclassDeep` | 16123 | 1 |
| `TestResultDataclassMoreEdgeCases` | 16368 | 2 |
| `TestPromptWithTimeout` | 12590 | 1 |
| `TestPromptWithTimeoutDeep` | 16097 | 1 |
| `TestDomainSets` | 12491 | 1 |
| `TestRoleLocalParts` | 12511 | 1 |
| `TestPlaceholderDomainFilter` | 12522 | 2 |
| `TestNetworkAvailableEventBarrier` | 16838 | 1 |

### Category 2: Email Extraction Tests

| Class | Line | Methods |
|-------|------|---------|
| `TestEmailExtraction` | 11007 | 15 |
| `TestCoverageDecoders` | 11139 | 5 |
| `TestEmailExtractionExtraEdgeCases` | 12740 | 3 |
| `TestExtractionEdgeCases` | 12172 | 8 |
| `TestExtractMailtoEmails` | 15869 | 3 |
| `TestBase64DecodeEmails` | 15912 | 2 |
| `TestExtractEmailsFromDataAttrs` | 14699 | 3 |
| `TestExtractEmailsFromSvg` | 14739 | 2 |
| `TestDecodeJsConcatEmailsDeep` | 14644 | 3 |
| `TestDecodeRot13EmailsDeep` | 14682 | 2 |
| `TestDecodeSmtpMsg` | 11487 | 1 |
| `TestDecodeSmtpMsgDeep` | 15195 | 2 |
| `TestDecodeSmtpMsgExtraEdgeCases` | 13236 | 1 |
| `TestPhoneExtraction` | 11606 | 2 |
| `TestPhoneExtractionDeep` | 15099 | 4 |
| `TestPhoneExtractionExtraEdgeCases` | 12823 | 2 |
| `TestExtractJsonldExtraEdgeCases` | 13116 | 3 |
| `TestJsonldSinglePassNotDouble` | 14207 | 1 |
| `TestJsConcatInputCap` | 13433 | 2 |
| `TestEmailValidationRegexModuleLevel` | 14181 | 1 |

### Category 3: Domain/URL Validation Tests

| Class | Line | Methods |
|-------|------|---------|
| `TestDomainNormalisation` | 10941 | 14 |
| `TestUrlNormalisation` | 11824 | 5 |
| `TestUrlNormalisationDeep` | 14400 | 10 |
| `TestNormaliseDomainExtraEdgeCases` | 12691 | 10 |
| `TestExtractDomainDeep` | 14484 | 5 |
| `TestDomainExtraction` | 12353 | 2 |
| `TestSameDomain` | 12391 | 3 |
| `TestIsSameDomainDeep` | 14532 | 3 |
| `TestIsValidCrawlUrlDeep` | 14562 | 5 |
| `TestIsProactiveUrl` | 14620 | 2 |
| `TestCrawlUrlValidation` | 12311 | 3 |
| `TestNormalizeDomainDeep` | 14904 | 2 |
| `TestNormalizeDomainListDeep` | 14933 | 1 |

### Category 4: SMTP Verifier Tests

| Class | Line | Methods |
|-------|------|---------|
| `TestGreylistDetection` | 11459 | 2 |
| `TestGreylistDeep` | 15167 | 2 |
| `TestGreylistExtraEdgeCases` | 13034 | 5 |
| `TestGreylistDefersToReverify` | 13388 | 2 |
| `TestAcceptAllCache` | 11748 | 2 |
| `TestAcceptAllCacheExtraEdgeCases` | 12950 | 3 |
| `TestAcceptAllCacheTtl` | 15569 | 1 |
| `TestAcceptAllPerDomainLock` | 14134 | 2 |
| `TestAcceptAllSkippedInAuthMode` | 14003 | 3 |
| `TestConfidenceScoring` | 11565 | 2 |
| `TestConfidenceScoringExtraEdgeCases` | 13182 | 2 |
| `TestEmailQuality` | 11508 | 3 |
| `TestEmailQualityExtraEdgeCases` | 12799 | 1 |
| `TestVerifierConnectorLimit` | 13327 | 3 |
| `TestVerifierResolverSwap` | 12084 | 2 |
| `TestVerifierWorkerHonoursShutdown` | 13652 | 2 |
| `TestVerifierCsvWriteOffloaded` | 13699 | 1 |
| `TestSmtpHandshakeFdLeak` | 13537 | 1 |
| `TestSmtpForceIpv4Setting` | 13739 | 2 |
| `TestSmtpHandshakeOuterFinallyClosesSmtp` | 16529 | 1 |

### Category 5: Google Sheets Tests

| Class | Line | Methods |
|-------|------|---------|
| `TestGoogleSheetUrl` | 11646 | 3 |
| `TestGoogleSheetUrlDeep` | 15218 | 2 |
| `TestGoogleSheetUrlExtraEdgeCases` | 12991 | 3 |

### Category 6: XLSX Export Tests

| Class | Line | Methods |
|-------|------|---------|
| `TestXlsxReport` | 12468 | 3 |
| `TestXlsxNoEmptyDefaultSheet` | 14234 | 1 |

### Category 7: Audit Regression Tests

| Class | Line | Methods |
|-------|------|---------|
| `_AuditTestBase` (base class) | 17226 | — |
| `TestAuditTlsVerifyDefault` | 17266 | 2 |
| `TestAuditTlsPlumbedToSmtpVerifier` | 17306 | 2 |
| `TestAuditCsvInjectionGuard` | 17394 | 5 |
| `TestAuditEmailRegexAsciiOnly` | 17505 | 1 |
| `TestAuditExtractDomainLowercase` | 17523 | 1 |
| `TestAuditIsTargetDomainEtld` | 17547 | 1 |
| `TestAuditRobotsCachesEmptyOnFailure` | 17569 | 1 |
| `TestAuditCaptchaDetection` | 17593 | 3 |
| `TestAuditAccessDeniedDetection` | 17659 | 1 |
| `TestAuditDeadProxyEviction` | 17690 | 2 |
| `TestAuditAtomicHomepageAnalysis` | 17728 | 1 |
| `TestAuditTier2BodyCap` | 17763 | 1 |
| `TestAuditUrlparseCaching` | 17786 | 1 |
| `TestAuditCrawlQueueSoftCap` | 17816 | 1 |
| `TestAuditRobotsLocksLruBound` | 17834 | 1 |
| `TestAuditCfCookieStoreRoundtrip` | 17854 | 2 |
| `TestAuditPersistedQueueResume` | 17896 | 2 |
| `TestAuditAdaptiveConcurrencyBackoff` | 17926 | 1 |
| `TestAuditSessionPoolBlocked` | 17963 | 1 |
| `TestAuditFingerprintConsistency` | 17998 | 1 |
| `TestAuditConnectionPoolSizing` | 18029 | 1 |
| `TestAuditInHouseCloudflareBypass` | 18241 | 4 |
| `TestAuditDeepCsvInjectionAllWriters` | 18357 | 2 |
| `TestAuditDeepEtldEdgeCases` | 18431 | 3 |
| `TestAuditDeepCfCookieStoreLruAndTtl` | 18478 | 3 |
| `TestAuditDeepPersistedQueueCrossInstance` | 18540 | 2 |
| `TestAuditDeepAdaptiveConcurrency` | 18579 | 2 |
| `TestAuditDeepProxyStrikeRace` | 18619 | 2 |
| `TestAuditDeepHomepageRaceManyThreads` | 18662 | 2 |
| `TestAuditDeepCfBypassRespectsConfig` | 18703 | 2 |
| `TestAuditDeepUrlparseCacheUnderConcurrency` | 18732 | 2 |
| `TestAuditCurlCffiBoundedGetBodyVisibility` | 18777 | 4 |
| `TestAuditEmailJsonEscapeNormalisation` | 18914 | 5 |
| `TestAuditNormalizeDomainResilience` | 19078 | 4 |
| `TestAuditVersionedBundlePathFilter` | 19168 | 3 |
| `TestAuditPlaceholderEmailDeny` | 19235 | 2 |
| `TestAuditApmIngestDsnFilter` | 19265 | 1 |
| `TestAuditPrescreenOneRetry` | 19294 | 1 |
| `TestAuditSheetBareIdAcceptance` | 19314 | 2 |
| `TestAuditPrivateSheetHtmlDetection` | 19369 | 2 |
| `TestAuditRotatedAssetRefAndThrowaway` | 19474 | 4 |
| `TestAuditCamoufoxCrossThreadDispatch` | 19615 | 7 |
| `TestAuditRegressionCriticalFixes` | 20518 | 15 |
| `TestSessionPoolNew` | 18058 | 5 |

### Category 8: Smoke Tests

| Class | Line | Methods |
|-------|------|---------|
| `TestSmokeGetTld` | 19828 | 1 |
| `TestSmokeMeasureInternetSpeed` | 19839 | 2 |
| `TestSmokeSelftestRun` | 19863 | 1 |
| `TestSmokeRegistrable` | 19892 | 2 |
| `TestSmokeProxyMethods` | 19916 | 2 |
| `TestSmokeRequestShutdown` | 19942 | 1 |
| `TestSmokeInstallShutdownHandler` | 19960 | 1 |
| `TestSmokeCleanupPidFile` | 19980 | 2 |
| `TestSmokeInvokeKwargStripping` | 20007 | 1 |
| `TestSmokeLogToCsvFile` | 20029 | 2 |
| `TestSmokePrintDeliverabilitySummary` | 20055 | 2 |
| `TestMultiDomain` | _(monitoring)_ | 3 |
| `TestCaptchaDetection` | 15760 | 1 |
| `TestAccessDeniedDetection` | 15780 | 1 |

---

## 5. EMAIL DEOBFUSCATION & EXTRACTION METHODS

### 1. Regex Extraction from HTML Text
- **Method**: `_extract_emails()` :4646 → `EMAIL_REGEX_CRAWLER.findall(normalised_html)`
- **Pattern**: `[a-zA-Z0-9.!#$%&'*+/=?^_\`{|}~-]+@[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]{2,})+`
- How it works: scans raw HTML text for any string matching the email pattern. Runs first, before any deobfuscation.

### 2. mailto: Link Parsing
- **Method**: `_extract_mailto_emails()` :4500
- How it works: finds all `<a href="mailto:...">` tags, extracts the email from the href, strips any `?subject=...` query params.

### 3. data- Attribute Extraction
- **Method**: `extract_emails_from_data_attrs()` (helper, ~14739)
- How it works: scans all elements with `data-*` attributes whose values contain `@`, uses regex to extract emails from attribute values. Common in JS-driven contact forms.

### 4. JSON-LD Structured Data Extraction
- **Method**: `extract_emails_from_jsonld()` (helper, ~9297)
- How it works: finds `<script type="application/ld+json">` tags, parses JSON, extracts `contactPoint.email`, `author.email`, `Person.email`, etc.

### 5. SVG Text Extraction
- **Method**: `extract_emails_from_svg()` (helper, ~9352)
- How it works: finds `<text>` elements inside SVG tags, collects their text content, applies regex.

### 6. Cloudflare Email Protection Decoding (`_decode_cf_xor_string`)
- **Method**: `_decode_cf_xor_string()` :4539
- How it works:
  1. Cloudflare obfuscates emails by XOR-encrypting each byte with a random key byte
  2. The encoded format is: first 2 hex chars = XOR key, remaining hex pairs = XOR-encrypted bytes
  3. `_decode_cf_xor_string` parses hex string, XOR-decrypts each byte with the key
  4. Two entry points:
     - `_decode_cloudflare_emails()` :4510 — reads `data-cfemail` attribute on tags
     - `_decode_cloudflare_hrefs()` :4524 — reads `/cdn-cgi/l/email-protection#` href fragments
  5. Strict validation: rejects non-printable-ASCII results, validates via `_EMAIL_VALIDATION_RE`

### 7. ROT13 Obfuscation Decoding
- **Method**: `decode_rot13_emails()` :1239
- How it works: applies ROT13 (Caesar cipher, shift 13) to the entire page text, then runs the email regex on the output. ROT13 is self-inverse so both obfuscated and plaintext emails appear — the downstream deduplication removes duplicates.

### 8. Cloudflare Obfuscated mailto Rotation Detection
- **Method**: `_decode_cloudflare_hrefs()` :4524
- How it works: Cloudflare also obfuscates mailto links by rewriting them as `mailto:/cdn-cgi/l/email-protection#<hex>`. This method detects those rewritten links and decodes them via `_decode_cf_xor_string`.

### 9. JS String Concatenation Decoding
- **Method**: `decode_js_concat_emails()` (helper, ~8826)
- How it works: detects patterns like `'sec' + 'urity@' + 'site.com'` inside `<script>` tags by matching adjacent quoted strings joined by `+` operators.

### 10. Base64 Decoding
- **Method**: `_decode_base64_emails()` :4488
- How it works: scans text for Base64-encoded strings that decode to email addresses.

### 11. `[at]` / `[dot]` Obfuscation (`_deobfuscate_emails`)
- **Method**: `_deobfuscate_emails()` :4586
- How it works: replaces common anti-harvesting patterns with the actual symbols:
  - `[at]`, `(at)`, ` @ `, `[@]`, `(@)`, `{at}`, `&#x40;`, `&#64;` → `@`
  - `[dot]`, `(dot)`, ` dot `, `[.]`, `(.)`, `{dot}`, `&#x2E;`, `&#46;` → `.`

### 12. Phone Number Extraction
- **Method**: phone extraction in `_process_page()` / via `_PHONE_PATTERN`
- **Pattern**: configurable regex matching common phone formats
- How it works: runs alongside email extraction, results stored in `self.phones`

---

## 6. FULL CONFIGURATION REFERENCE

### Module-Level Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `EMAIL_REGEX` | `^[a-zA-Z0-9.!#$%&'*+/=?^_\`{\|}~-]+@[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]{2,})*$` | Primary email extraction regex |
| `_EMAIL_VALIDATION_RE` | `^[\w.%+-]{2,64}@[\w.-]+\.[\w]+$` (ASCII flag) | Strict email syntax validator |
| `DEFAULT_TIMEOUT` (CLI default) | `20` | Default HTTP request timeout (seconds) |
| `ACCEPT_ALL_CACHE` | `OrderedDict()` | LRU cache for accept-all domains |
| `_ACCEPT_ALL_CACHE_MAX` | `10_000` | Max accept-all cache entries |
| `_ACCEPT_ALL_CACHE_TTL_SECONDS` | `86_400` (24h) | Accept-all cache TTL |
| `_ACCEPT_ALL_CACHE_LOCK` | `threading.Lock()` | Thread-safe guard for cache |
| `_RESOLVER_TTL` | `300` | DNS resolver cache TTL (seconds) |
| `STEALTH_TLS_DISABLED` | `False` | Module-level flag (set by `--no-stealth-tls`) |
| `BROWSER_FALLBACK_ENABLED` | `False` | Module-level flag (set by `--browser-fallback`) |
| `INSECURE_TLS` | `False` | Module-level flag (set by `--insecure-tls`) |
| `CSV_INJECTION_GUARD` | `True` | Module-level flag (set by `--no-csv-escape`) |
| `PERSIST_QUEUE` | `False` | Module-level flag (set by `--persist-queue`) |
| `CAPTCHA_SOLVER` | `None` | Module-level flag (set by `--captcha-solver`) |
| `CAPTCHA_API_KEY` | `None` | Deprecated no-op |
| `HAS_CURL_CFFI` | `True/False` | Detected at import |
| `HAS_CAMOUFOX` | `False` | Detected at import |
| `_HTML_PARSER` | `"lxml"` or `"html.parser"` | Detected at import |
| `_CSV_FORMULA_PREFIXES` | `("=", "+", "-", "@", "\t", "\r")` | CSV injection formula prefixes |
| `_CONTACT_KEYWORDS` | `(tuple of keywords)` | Contact page detection keywords |
| `_JS_CONCAT_PATTERN` | `re.compile(...)` | JS string concatenation regex |
| `_JS_CONCAT_MAX_INPUT_CHARS` | `50_000` | Max JS concat decoder input |
| `_PHONE_PATTERN` | `re.compile(...)` | Phone number extraction regex |
| `_JSON_ESCAPE_DECODE_RE` | `re.compile(...)` | `\u003e` / `\u003c` etc. JSON escape decoder |
| `_PLACEHOLDER_EMAIL_DENY` | `frozenset{...}` | Exact-placeholder email deny list |
| `_PLACEHOLDER_EMAIL_DOMAINS` | `frozenset{...}` | Placeholder domain deny list |
| `_PLACEHOLDER_LOCAL_PART_PREFIXES` | `(tuple)` | Prefixes for placeholder local-parts |
| `_THROWAWAY_EMAIL_DOMAINS` | `frozenset{...}` | Disposable/temp email domains |
| `_ASSET_EXTENSION_TLDS` | `(tuple)` | Asset extensions treated as TLDs (`.png`, `.jpg`, etc.) |
| `_LABEL_ALL_DIGITS_RE` | `re.compile(r"^\d+$")` | Detects all-digit labels |
| `_LABEL_VERSION_HASH_RE` | `re.compile(...)` | Detects version-hash labels (e.g. `2.1.7-1b23d546`) |
| `_LABEL_SEMVER_RE` | `re.compile(...)` | Detects semver labels (e.g. `2.1.7`) |
| `_APM_INGEST_HOST_FRAGMENTS` | `(tuple)` | APM DSN host fragments (Sentry, Bugsnag, etc.) |
| `_GOOGLE_SHEET_RE` | `re.compile(...)` | Google Sheet URL/id regex |
| `_CF_BYPASS_NETWORK_IDLE_MS` | `12_000` | Max idle ms for CF bypass page load |
| `_CF_BYPASS_CHALLENGE_TIMEOUT` | `25.0` | Max seconds for CF challenge resolve |
| `_CF_BYPASS_HUMANISH_JITTER` | `True` | Enable mouse/scroll noise in CF bypass |
| `_CRAWL_URL_BLOCKLIST_RE` | `re.compile(...)` | Class-level (AdvancedEmailScraper) URL blocklist |
| `_SITEMAP_MAX_RECURSION` | ~`10` | Max sitemap index recursion |
| `_DEAD_PROXY_STRIKES` | `3` | Consecutive failures before proxy eviction |
| `_ROBOTS_LOCKS_MAX` | `1024` | Max LRU size for robots domain lock cache |
| `SMTP_CONCURRENCY_CEILING` | `300` (hardcoded ~line 530) | Max SMTP connections |
| `CF_COOKIE_STORE` | `_CloudflareCookieStore()` | Global singleton CF cookie jar |

### `verifier_settings` Keys

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `direct_check_user` | string | `"info"` | MAIL FROM local-part |
| `direct_check_host` | string | `"example.com"` | MAIL FROM domain |
| `smtp_host` | string | `""` | Authenticated SMTP relay host |
| `smtp_port` | int | `587` | SMTP relay port |
| `smtp_user` | string | `""` | SMTP relay username |
| `smtp_pass` | string | `""` | SMTP relay password |
| `smtp_tls` | bool | `True` | Use TLS for SMTP relay |
| `default_verification_mode` | string | `"direct"` | `"direct"` or `"auth"` |
| `default_email_column` | string | `"B"` | Email column in sheet |
| `default_classification_column` | string | `"C"` | Classification column |
| `default_domain_column` | string | `"A"` | Domain column |
| `prompt_timeout` | int | `5` | Seconds before interactive menu times out |
| `default_concurrency` | int | `100` | Default SMTP concurrency |
| `greylist_retry_delay_seconds` | int | `60` | Delay before greylist retry |
| `smtp_force_ipv4` | bool | `False` | Force A-record only for SMTP |
| `smtp_verify_tls` | bool | `False` | Verify STARTTLS certificate |

### `crawler_settings` Keys

| Key | Type | Default (subset shown) | Description |
|-----|------|------------------------|-------------|
| `prioritized_email_prefixes` | list | `contact, support, info, ...` | Local-part prefixes for prioritized classification |
| `other_email_prefixes` | list | `sales, community, feedback, ...` | Local-part prefixes for other classification |
| `common_providers` | list | `gmail.com, outlook.com, ...` | Common email providers |
| `primary_target_keywords` | list | `contact, support, about, team, ...` | Keywords for primary path detection |
| `secondary_target_keywords` | list | `privacy, legal, terms, ...` | Keywords for secondary path detection |
| `proactive_primary_paths` | list | `/contact, /about, /team, /security, ...` | Priority scan URL paths |
| `proactive_secondary_paths` | list | `/privacy, /terms, /legal, ...` | Secondary scan URL paths |

---

## 7. HISTORY DB SCHEMA

### Table: `emails`
```sql
CREATE TABLE IF NOT EXISTS emails (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    domain        TEXT NOT NULL,
    email         TEXT NOT NULL,
    first_seen    TEXT NOT NULL,        -- ISO timestamp
    last_seen     TEXT NOT NULL,        -- ISO timestamp
    run_count     INTEGER NOT NULL DEFAULT 1,
    prioritized   INTEGER NOT NULL DEFAULT 0,  -- boolean
    source_type   TEXT,                  -- "target-domain" or "cross-domain"
    is_role       INTEGER NOT NULL DEFAULT 0,  -- boolean
    is_disposable INTEGER NOT NULL DEFAULT 0,  -- boolean
    is_privacy    INTEGER NOT NULL DEFAULT 0,  -- boolean
    UNIQUE(domain, email)
);
CREATE INDEX IF NOT EXISTS ix_emails_domain ON emails(domain);
```

### Table: `verifications`
```sql
CREATE TABLE IF NOT EXISTS verifications (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    email       TEXT NOT NULL,
    ts          TEXT NOT NULL,           -- ISO timestamp
    status      TEXT NOT NULL,           -- deliverable|undeliverable|accept-all|greylisted|unknown|unreachable|no MX records
    reason      TEXT,                    -- SMTP response text
    confidence  INTEGER,                 -- 0-100 confidence score
    has_gravatar INTEGER NOT NULL DEFAULT 0,  -- boolean
    run_id      TEXT
);
CREATE INDEX IF NOT EXISTS ix_verify_email  ON verifications(email);
CREATE INDEX IF NOT EXISTS ix_verify_run    ON verifications(run_id);
```

### Table: `runs`
```sql
CREATE TABLE IF NOT EXISTS runs (
    run_id      TEXT PRIMARY KEY,
    started_ts  TEXT NOT NULL,           -- ISO timestamp
    ended_ts    TEXT,                    -- ISO timestamp (NULL until finish_run)
    n_domains   INTEGER NOT NULL DEFAULT 0,
    n_emails    INTEGER NOT NULL DEFAULT 0,
    n_phones    INTEGER NOT NULL DEFAULT 0,
    cli_args    TEXT,                    -- CLI arguments for reproducibility
    python_ver  TEXT,                    -- Python version string
    git_sha     TEXT                     -- Git commit SHA
);
```

### Journal Mode
```sql
PRAGMA journal_mode=WAL;       -- WAL mode for concurrent read/write
PRAGMA synchronous=NORMAL;     -- Balanced durability/performance
```

### Query Patterns

| Query | Purpose | Location |
|-------|---------|----------|
| `INSERT ... ON CONFLICT(domain, email) DO UPDATE SET run_count=run_count+1, ...` | Record or bump email | `record_email()` :6758 |
| `INSERT INTO verifications(...) VALUES(...)` | Append verification result | `record_verification()` :6785 |
| `INSERT OR IGNORE INTO runs(run_id, started_ts, ...) VALUES(...)` | Start run tracking | `start_run()` |
| `UPDATE runs SET ended_ts=?, ... WHERE run_id=?` | End run tracking | `finish_run()` |
| `SELECT COUNT(*) FROM emails` | Stats query | `stats()` :6814 |
| `SELECT COUNT(DISTINCT domain) FROM emails` | Unique domain count | `stats()` :6816 |
| `SELECT COUNT(*) FROM verifications` | Total verifications | `stats()` :6818 |
| `SELECT run_count, is_role FROM emails WHERE email = ?` | Check if email was previously seen | :9852 |
| `SELECT domain, email, prioritized, source_type FROM emails` | Export / cross-run dedup | :12126 |
| `SELECT email, status, reason, confidence FROM verifications WHERE email=?` | Previous verification results | :15727 |
| `SELECT url, depth FROM urls WHERE status='queued'` | Resume persisted queue | :3506 (from `_PersistedRequestQueue`) |

### Persisted Crawl Queue (`._queue.sqlite3`)
```sql
CREATE TABLE IF NOT EXISTS urls(
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    url     TEXT NOT NULL,
    depth   INTEGER NOT NULL DEFAULT 1,
    status  TEXT NOT NULL DEFAULT 'queued'
);
CREATE INDEX IF NOT EXISTS idx_urls_status ON urls(status);
```

---

## 8. CONCURRENCY & THREAD SAFETY REFERENCE

### Primitive Inventory

| Primitive | Line | Protects |
|-----------|------|----------|
| `threading.Lock` | 1508 | `ACCEPT_ALL_CACHE` (accept-all domain cache) |
| `threading.Lock` | 1567 | `_resolver` (aiodns DNS resolver instance) |
| `threading.Lock` | 2968 | `_CloudflareCookieStore` (cf_clearance cookie jar) |
| `threading.Lock` | 3168 | `_SessionPool` (session list + fingerprints) |
| `threading.Lock` | 3429 | `_PersistedRequestQueue` (SQLite queue I/O) |
| `threading.Lock` | 3553 | `_AdaptiveConcurrency` (success/failure counts) |
| `threading.Lock` | 15855 | Test helper lock (thread-safe result collection) |
| `threading.Lock` | 16623, 16650 | Test lock for robots cache race tests |
| `threading.Lock` | 19719 | Camoufox cross-thread dispatch test |
| `threading.RLock` | 780 | `DomainRateLimiter` (per-domain token bucket state) |
| `threading.RLock` | 812 | Per-domain rate limiter token buckets dict |
| `threading.RLock` | 3616 | `_PoolSession` session pool lock |
| `threading.RLock` | 3765 | `_selenium_lock` — serialize all Selenium driver ops |
| `threading.RLock` | 3794 | `AdvancedEmailScraper.lock` — results dict, visited_urls, crawl_queue, etc. |
| `threading.RLock` | 3815 | `_camoufox_lock` — serialize camoufox browser access |
| `threading.RLock` | 3973 | `_robots_fetch_lock` — robots.txt fetch + LRU cache |
| `threading.RLock` | 4058 | Per-domain `RLock` for robots.txt single-flight fetch |
| `threading.RLock` | 6742 | `EmailHistoryDB.lock` — SQLite writes (reentrant) |
| `asyncio.Lock` | 2119 | Per-domain accept-all check (prevents N simultaneous probes) |
| `asyncio.Semaphore` | 2271, 2338 | SMTP verifier concurrency throttle (default 100, half on retry) |
| `threading.Event` | 3877 | `_network_available` — outage barrier (cleared on 20+ failures, set on successful probe) |
| `threading.Event` | 17004, 17738 | Test instrumentation events |
| `ThreadPoolExecutor` | 3679 | Pre-screening pool (`max(20, args.threads)` workers) |
| `ThreadPoolExecutor` | 5957 | Crawler pool (`max_threads` workers) |
| `ThreadPoolExecutor` | 5372 | Camoufox dedicated single-thread pool (greenlet-bound) |

### Lock Ordering (deadlock prevention)

```
1. AdvancedEmailScraper.lock (RLock, outermost)
2. Per-domain robots fetch lock (RLock, acquired inside lock)
3. DomainRateLimiter._lock (RLock, per-domain)
   ├── Never acquire AdvancedEmailScraper.lock while holding this
   └── Never acquire _robots_fetch_lock while holding this
4. _ACCEPT_ALL_CACHE_LOCK (Lock, microsecond-level)
   └── Never acquire any other lock while holding this
5. EmailHistoryDB.lock (RLock, outermost DB lock)
   └── SQLite's own internal lock (WAL mode + timeout=30s)
```

**Invariant**: No lock is ever acquired while holding a lower-numbered lock.

### Single-Flight Patterns

- **Robots.txt fetch**: Per-domain `RLock` in `_get_robots_rules()` :4023 — only one thread fetches; others wait and return cached result
- **Homepage analysis**: `homepage_analysis_done` set checked in `_process_page()` — only one worker per domain analyzes homepage
- **Accept-all probe**: Per-domain `asyncio.Lock` in `check_accept_all()` :2119 — only one coroutine probes; others wait for cached result
- **Camoufox Tier-2**: All camoufox operations funneled through a single dedicated `ThreadPoolExecutor` (greenlet-bound, non-reentrant)

### Thread-Local Storage

- Camoufox browser instance: single-thread bound (an RLock-guarded check prevents cross-thread access)
- `AsyncEmailVerifier._resolver`: per-instance, not per-coroutine; accessed under `_resolver_lock`

### Race Condition History

| Race | Fix | PR/Commit |
|------|-----|-----------|
| robots.txt N simultaneous fetches for same domain | Single-flight per-domain RLock | Batch 1 |
| homepage_analysis_done race (N workers) | Atomic set membership + set | Batch 2 |
| ACCEPT_ALL_CACHE unbounded growth | OrderedDict + LRU + 24h TTL | PR #6 |
| CSV header non-atomic write across processes | Exclusive `open(..., 'x')` create | Perf audit |
| Crawl state leak across monitoring cycles | Explicit `.clear()` on all state in `crawl()` | Batch 5 |
| Selenium driver state corruption (concurrent `.get()`) | Serialize via `_selenium_lock` RLock | Batch 3 |
| `_deobfuscate_emails` case-folding lost original | Preserve case before lowercase for dedup | Audit 2026-05 |

---

## 9. ALL MODULE-LEVEL CONSTANTS

| Constant | Value | Description |
|----------|-------|-------------|
| `EMAIL_REGEX` | `^[a-zA-Z0-9.!#$%&'*+/=?^_\`{\|}~-]+@[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]{2,})*$` | Primary email extraction regex |
| `_EMAIL_VALIDATION_RE` | `^[\w.%+-]{2,64}@[\w.-]+\.[\w]+$` (ASCII flag) | Strict email syntax validator |
| `DEFAULT_TIMEOUT` (CLI default) | `20` | Default HTTP request timeout (seconds) |
| `ACCEPT_ALL_CACHE` | `OrderedDict()` | LRU cache for accept-all domains |
| `_ACCEPT_ALL_CACHE_MAX` | `10_000` | Max accept-all cache entries |
| `_ACCEPT_ALL_CACHE_TTL_SECONDS` | `86_400` (24h) | Accept-all cache TTL |
| `_ACCEPT_ALL_CACHE_LOCK` | `threading.Lock()` | Thread-safe guard for cache |
| `_RESOLVER_TTL` | `300` | DNS resolver cache TTL (seconds) |
| `STEALTH_TLS_DISABLED` | `False` | Module-level flag (set by `--no-stealth-tls`) |
| `BROWSER_FALLBACK_ENABLED` | `False` | Module-level flag (set by `--browser-fallback`) |
| `INSECURE_TLS` | `False` | Module-level flag (set by `--insecure-tls`) |
| `CSV_INJECTION_GUARD` | `True` | Module-level flag (set by `--no-csv-escape`) |
| `PERSIST_QUEUE` | `False` | Module-level flag (set by `--persist-queue`) |
| `CAPTCHA_SOLVER` | `None` | Module-level flag (set by `--captcha-solver`) |
| `CAPTCHA_API_KEY` | `None` | Deprecated no-op |
| `HAS_CURL_CFFI` | `True/False` | Detected at import |
| `HAS_CAMOUFOX` | `False` | Detected at import |
| `_HTML_PARSER` | `"lxml"` or `"html.parser"` | Detected at import |
| `_CSV_FORMULA_PREFIXES` | `("=", "+", "-", "@", "\t", "\r")` | CSV injection formula prefixes |
| `_CONTACT_KEYWORDS` | `(tuple of keywords)` | Contact page detection keywords |
| `_JS_CONCAT_PATTERN` | `re.compile(...)` | JS string concatenation regex |
| `_JS_CONCAT_MAX_INPUT_CHARS` | `50_000` | Max JS concat decoder input |
| `_PHONE_PATTERN` | `re.compile(...)` | Phone number extraction regex |
| `_JSON_ESCAPE_DECODE_RE` | `re.compile(...)` | `\u003e` / `\u003c` etc. JSON escape decoder |
| `_PLACEHOLDER_EMAIL_DENY` | `frozenset{...}` | Exact-placeholder email deny list |
| `_PLACEHOLDER_EMAIL_DOMAINS` | `frozenset{...}` | Placeholder domain deny list |
| `_PLACEHOLDER_LOCAL_PART_PREFIXES` | `(tuple)` | Prefixes for placeholder local-parts |
| `_THROWAWAY_EMAIL_DOMAINS` | `frozenset{...}` | Disposable/temp email domains |
| `_ASSET_EXTENSION_TLDS` | `(tuple)` | Asset extensions treated as TLDs (`.png`, `.jpg`, etc.) |
| `_LABEL_ALL_DIGITS_RE` | `re.compile(r"^\d+$")` | Detects all-digit labels |
| `_LABEL_VERSION_HASH_RE` | `re.compile(...)` | Detects version-hash labels (e.g. `2.1.7-1b23d546`) |
| `_LABEL_SEMVER_RE` | `re.compile(...)` | Detects semver labels (e.g. `2.1.7`) |
| `_APM_INGEST_HOST_FRAGMENTS` | `(tuple)` | APM DSN host fragments (Sentry, Bugsnag, etc.) |
| `_GOOGLE_SHEET_RE` | `re.compile(...)` | Google Sheet URL/id regex |
| `_CF_BYPASS_NETWORK_IDLE_MS` | `12_000` | Max idle ms for CF bypass page load |
| `_CF_BYPASS_CHALLENGE_TIMEOUT` | `25.0` | Max seconds for CF challenge resolve |
| `_CF_BYPASS_HUMANISH_JITTER` | `True` | Enable mouse/scroll noise in CF bypass |
| `_CRAWL_URL_BLOCKLIST_RE` | `re.compile(...)` | Class-level (AdvancedEmailScraper) URL blocklist |
| `_SITEMAP_MAX_RECURSION` | ~`10` | Max sitemap index recursion |
| `_DEAD_PROXY_STRIKES` | `3` | Consecutive failures before proxy eviction |
| `_ROBOTS_LOCKS_MAX` | `1024` | Max LRU size for robots domain lock cache |
| `SMTP_CONCURRENCY_CEILING` | `300` (hardcoded ~line 530) | Max SMTP connections |
| `CF_COOKIE_STORE` | `_CloudflareCookieStore()` | Global singleton CF cookie jar |

---

## 10. VERSION HISTORY / CHANGELOG

```
26a8dbb Merge pull request #15 — Audit FP triage v2 (drop rotated CSS refs, throwaway emails)

  0b3c506 audit FP triage v2: drop rotated CSS-asset refs + throwaway emails + example@*
  2ae479b tier-2: elevate CF cookie harvest log to INFO
  b0e92d0 audit FP triage: drop JS-bundle / placeholder / APM-DSN @-domains + prescreen retry
  adfbad6 audit: --sheet accepts bare sheet ID; +4 NO-MOCK tests (88 smoke + 628 extended)
  b03c031 audit: CRIT — normalize_domain crash on bracketed tokens (Python 3.12);
          private-sheet HTML detection; +8 NO-MOCK tests (88 smoke + 624 extended)
  dcffe91 audit: data-quality — decode \uXXXX JSON escapes BEFORE email regex;
          render.com FP fix; pre-decode <script> bodies; +7 NO-MOCK tests
  b5fc783 audit: CRIT fix — _curl_cffi_bounded_get writes to public .content (not ._content);
          body was silently lost under stream=True; 0 → 37 emails found per real run
  00d87ae audit: deeper NO-MOCK pack (+18 tests, total 606)
  25d647c audit: drop all paid CAPTCHA solvers; implement in-house Cloudflare bypass
  2c38480 audit: CSV injection regression tests use production helpers end-to-end
  da4945b audit: 46 NO-MOCK regression tests for audit fixes Batch 1-5 (581 total)
  264de55 audit: 50-100k/day tuning — bump cloudscraper urllib3 pool, general_links cap (5k)
  4e689a6 audit: Batch 5 — _CloudflareCookieStore, _BrowserFingerprint, _SessionPool,
          CAPTCHA solver hooks (deprecated), _PersistedRequestQueue, _AdaptiveConcurrency

c1d1e85 audit: Batch 3+4 — urlparse() lru_cache, _robots_domain_locks LRU, recovery sleep jitter,
        crawl_queue soft cap
fe0aa79 audit: Batch 1+2 — TLS verify default on, CSV-injection guard, data-cfemail strict ASCII,
        sitemap netloc check, connectivity probe, dead-proxy eviction, atomic homepage_analysis_done,
        Tier-2 body cap, --insecure-tls / --no-csv-escape / --diagnose-tls / --persist-queue flags

7eda245 Merge branch: Purge CustomHttpAdapter + outage barrier
  df4422e test: replace time.sleep(0.2) with instrumented Event.wait() sync
  ab0ea2d fix: post-loop _SHUTDOWN_REQUESTED check + non-spurious test
  23789f4 fix: restore _network_available in call-site finally
  8970866 audit: deep-audit-7 (#1+#2) — purge CustomHttpAdapter, add outage barrier
  8e99ff5 fix: prescreen_domains._probe must close response in finally

cbb2a5e docs: expand README with TL;DR max-coverage command + recipe gallery

5bb8ff7 audit: deep-audit-6 fixes (5 of 7 verified) + 12 regression tests

d273149 test: replace no-op test_uppercase_scheme_passes with real assertions
6b1fb33 test: +218 per-helper / per-method coverage tests (376 → 598 total)

1b3c820 perf: queued_urls set for O(1) link-extraction dedup

0e88951 perf: BS4 decompose + off-load history_db SQLite writes (deep-audit-4)

26888c6 fix: address Devin Review + auth-mode log + PR #5 swallowed exceptions
56beef2 audit: deep-audit-3 fixes (12 of 14 claims) + 12 regression test classes

f495f9c perf: 4 daemon hyper-optimisation fixes (loop blocking, lxml, IPv6 blackhole, ulimit)

8515e37 fix: filter None worker results before append
273c590 audit: 4 daemon-stability fixes from deep audit #2
```

### Grouped by Theme

**Audits & Stability:**
- deep-audit-2: 4 daemon-stability fixes
- deep-audit-3: 12 fixes + 12 regression test classes
- deep-audit-4: BS4 decompose + offload DB writes
- deep-audit-6: 5 fixes + 12 regression tests
- deep-audit-7: Purge CustomHttpAdapter, outage barrier
- Batch 1+2: TLS verify, CSV injection, sitemap, connectivity, dead-proxy eviction
- Batch 3+4: urlparse cache, LRU locks, crawl queue soft cap
- Batch 5: Native Crawlee-inspired features (CF cookie store, session pool, persisted queue, adaptive concurrency)
- FP triage v1: JS-bundle, placeholder, APM-DSN filters, prescreen retry
- FP triage v2: CSS rotated asset refs, throwaway emails

**Critical Bug Fixes:**
- `_curl_cffi_bounded_get` body lost under `stream=True` (body went to `._content` not `.content` — found 0 emails everywhere)
- `normalize_domain` crash on bracketed tokens in Python 3.12
- JSON escape decoder for `\u003e` / `\u003c` before email regex
- CustomHttpAdapter leak → purged

**Performance:**
- queued_urls set for O(1) dedup
- 50-100k/day tuning (urllib3 pool sizing, general_links cap)
- 4 daemon hyper-optimisation fixes (loop blocking, lxml, IPv6 blackhole, ulimit)

**Feature Additions:**
- In-house Cloudflare bypass (free, fully local)
- `--persist-queue` SQLite-backed crash resilience
- `_AdaptiveConcurrency` auto-tuning
- `_CloudflareCookieStore` cross-tier cf_clearance reuse
- Sheet bare ID acceptance

**Testing:**
- +218 per-helper coverage tests (376→598→606→616→624→628)
- 46 NO-MOCK regression tests (Batch 1-5)
- Event.wait() sync replaces time.sleep()
