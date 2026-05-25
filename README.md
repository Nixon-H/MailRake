# ADVANCED EMAIL SCRAPPER

A full-featured, production-grade email harvesting and verification toolkit. Crawls websites for email addresses, phone numbers, and contact metadata; then SMTP-verifies deliverability. Supports stealth TLS fingerprinting, browser-based JavaScript rendering, Cloudflare bypass, continuous Google Sheet monitoring, automated concurrency tuning, webhook notifications, and a cross-run SQLite history database.

## 1. OVERVIEW

Single-file Python application (~22,000 lines) combining:

- **Web Crawler** – Multi-threaded, robots.txt–respecting, sitemap-aware crawl engine that discovers and extracts email addresses and phone numbers from target domains.
- **Email Verifier** – Async SMTP-based verifier that classifies emails as deliverable, undeliverable, accept-all, greylisted, or unknown using direct MX probing or authenticated relay.
- **Continuous Monitoring** – Polls a public Google Sheet on a configurable interval, crawling only new (previously unprocessed) domains.
- **Interactive Menu** – TUI-driven mode for users who prefer guided operation over CLI flags.

## 2. ARCHITECTURE

The code is organized top-to-bottom in a single file with clear section boundaries:

| Lines | Component |
|-------|-----------|
| 1–199 | Imports, module-level toggles, HTML parser detection |
| 200–386 | Utility functions (domain normalization, HTML sanitization, CSV safety) |
| 387–1487 | Stealth TLS session factory, robots.txt parser, email extraction heuristics |
| 1488–2556 | Async SMTP email verifier (direct + auth modes, rate limiting, caching) |
| 2556–3697 | Tier-1 stealth HTTP session (`curl_cffi` Chrome 120 impersonation, pre-screening) |
| 3697–6500 | `AdvancedEmailScraper` crawler class (priority scan, deep crawl, page parsing, output) |
| 6500–7232 | History DB, disposable/placeholder domain filters, webhook, xlsx writer |
| 7233–8190 | Output directory setup, settings menu, standalone verifier, cleanup |
| 8190–10900 | `run_scraper_instance`, self-test suite (88 smoke checks) |
| 10901–17156 | Extended unittest test suite (581+ tests) |
| 17157–21583 | More extended tests (continued) |
| 21584–22207 | `main()` CLI parser, continuous monitoring loop, interactive menu |

**Key design patterns:**
- **Tiered fetching** – Tier 1: `curl_cffi` Chrome 120 TLS+JA3 impersonation; Tier 2: `camoufox` stealth-Firefox fallback for blocked domains.
- **In-house CAPTCHA bypass** – No paid third-party solvers; uses existing Tier-2 camoufox browser to resolve Cloudflare challenges and harvest `cf_clearance` cookies into a cross-request cookie store.
- **Async SMTP verifier** – Asyncio-based with semaphore-based concurrency throttling, accept-all cache (24h TTL), greylist retry logic, IPv4 forcing option.
- **Thread-safe output** – RLock-guarded CSV/JSON writing, atomic `os.mkdir` run-directory allocation, crash-resumable queue via `--persist-queue` SQLite backend.

## 3. OUTPUT FILES

All output lives under `output/YYYY-MM-DD/<run_number>/`:

| File | Description |
|------|-------------|
| `JSON/<base>_emails.json` | Per-domain email map (prioritized + other, target-domain + cross-domain) |
| `JSON/<base>_crossdomain.json` | Cross-domain emails separately (if any) |
| `CSV/<base>_emails.csv` | Columns: Domain, Email, Classification (prioritized/other), Source_Type (target-domain/cross-domain) |
| `CSV/<base>_report.csv` | Per-domain crawl report: Domain, Status (Success/Failed), Details |
| `CSV/<base>_phones.csv` | Phone numbers found (only if any extracted). Columns: Domain, Phone |
| `CSV/<base>_verified_emails.csv` | Verifier results (written after verification step). Deliverability classification per email |
| `<base>.xlsx` | Combined workbook (if `--xlsx`), with green/red/amber conditional formatting |
| `metadata.json` | Run summary: run_id, timestamps, counts, CLI args, git SHA, platform |
| `Logs/domain_crawler.log` | Structured INFO-level log for the run |
| `.tmp/` | Temporary files (removed on clean shutdown) |

Additionally in `--url` continuous-monitoring mode:
- `output/Nixon-S master folder/` – Aggregate master logs across monitoring cycles
- `output/processed_domains.log` – Tracks which domains have already been processed

## 4. CLI USAGE

```
python mailrake.py <input_file> [options]
python mailrake.py --sheet <google_sheet_url> [options]
python mailrake.py --url <google_sheet_url> [options]
python mailrake.py --self-test
python mailrake.py --diagnose-tls
```

### Core arguments

| Argument | Description |
|----------|-------------|
| `input_file` | `.txt` (one domain per line) or `.csv` (column A holds domains). Google Sheet URL auto-detected |
| `--sheet` | Public Google Sheet URL (one-shot fetch) |
| `--url` | Public Google Sheet URL (continuous monitoring, default 5-minute interval) |
| `-t, --threads` | Number of concurrent threads (default: 10) |
| `--timeout` | Request timeout in seconds (default: 20) |
| `--delay` | Delay between requests in seconds (default: 0.5) |
| `-o, --output` | Base filename for output files (default: "results") |
| `--proxies` | File with proxies (one per line) |
| `--exclude` | File with domains/TLDs to exclude (one per line) |

### Stealth & crawl flags

| Argument | Description |
|----------|-------------|
| `--no-stealth-tls` | Disable `curl_cffi` Chrome 120 TLS+JA3 impersonation (fallback to cloudscraper) |
| `--browser-fallback` | Enable Tier-2 camoufox stealth-Firefox for domains returning 403/429/503 |
| `--selenium` | [DEPRECATED] Use `--browser-fallback` instead |
| `--no-robots` | Ignore `robots.txt` Disallow rules (Crawl-delay still respected) |
| `--no-sitemap` | Skip sitemap.xml pre-seeding |
| `--prescreen` | HEAD-probe each domain first, skip unreachable/parked/firewalled ones |
| `--insecure-tls` | Disable TLS certificate verification on outbound HTTPS |

### Automation & ops flags

| Argument | Description |
|----------|-------------|
| `--one-shot, --auto-run` | Fully unattended: skip prompts, auto-run deep crawl + verifier |
| `--auto-tune` | Measure outbound bandwidth and auto-pick threads/concurrency |
| `--dry-run` | Parse and report what would be crawled, then exit without network I/O |
| `--self-test` | Run 88 smoke checks + 581+ unittest tests and exit |
| `--diagnose-tls` | Print the TLS+JA3 fingerprint and exit |
| `--interval` | Seconds between checks in `--url` monitoring mode (default: 300) |
| `--history-db` | Path to SQLite history DB (default: `output/email_history.db`) |
| `--no-history-db` | Disable history DB entirely |
| `--webhook` | POST JSON run-summary to this URL on completion |
| `--reverify-unknown` | Re-run "unknown" results once at half concurrency |
| `--xlsx` | Emit `.xlsx` workbook with conditional formatting (requires `openpyxl`) |
| `--dry-run` | Parse + normalize input and exit before any I/O |
| `--pid-file` | Write PID to file (removed on clean shutdown) |
| `--persist-queue` | Persist crawl queue to disk for crash recovery |
| `--no-csv-escape` | Disable CSV-injection escaping of leading `=/-+/@` |
| `--captcha-solver` | Only "in_house" supported (uses Tier-2 camoufox to auto-resolve) |
| `--captcha-api-key` | [DEPRECATED] No-op; kept for backwards compatibility |
| `-v, --verbose` | Enable INFO-level console logging |

## 5. EXAMPLES

```bash
# Basic crawl from text file (10 threads)
python mailrake.py domains.txt

# High-concurrency crawl with stealth TLS
python mailrake.py domains.txt -t 50 --timeout 30 --delay 1.0

# Google Sheet (one-shot) with browser fallback and pre-screening
python mailrake.py --sheet "https://docs.google.com/spreadsheets/d/..." --browser-fallback --prescreen -t 30

# Continuous monitoring with webhook notification on completion
python mailrake.py --url "https://docs.google.com/spreadsheets/d/..." --interval 300 --webhook "https://hooks.slack.com/..."

# Fully unattended run with auto-tuning and xlsx output
python mailrake.py domains.csv --one-shot --auto-tune --xlsx --reverify-unknown

# Exclude specific domains/TLDs
python mailrake.py domains.txt --exclude exclude.txt -t 20

# Dry-run to preview what would be crawled
python mailrake.py domains.txt --dry-run

# Run all validation tests
python mailrake.py --self-test

# Diagnose the TLS fingerprint your crawler presents
python mailrake.py --diagnose-tls
```

## 6. DEPENDENCIES

### Required (auto-installed via `pip install`)

| Package | Purpose |
|---------|---------|
| `requests` | HTTP requests for sitemap fetching, webhooks |
| `beautifulsoup4` | HTML parsing and email extraction |
| `lxml` | Fast HTML/XML parser (10x faster than stdlib) |
| `cloudscraper` | Cloudflare JS challenge bypass (Tier-1 fallback) |
| `tld` | Public Suffix List domain extraction |
| `tqdm` | Progress bars for CLI operations |
| `aiosmtplib` | Async SMTP client for email verification |
| `aiohttp` | Async HTTP for sitemap fetching |
| `aiodns` | Async DNS resolution |
| `urllib3` | Lower-level HTTP (InsecureRequestWarning suppression) |

### Optional

| Package | Purpose | How to install |
|---------|---------|---------------|
| `curl_cffi` | Chrome 120 TLS+JA3 fingerprint impersonation (Tier-1 stealth) | `pip install curl_cffi` |
| `camoufox` | Stealth Firefox browser for Tier-2 JS rendering and CAPTCHA bypass | `pip install camoufox && python -m camoufox fetch` |
| `python-socks[asyncio]` | SOCKS/HTTP proxy support for the async verifier | `pip install python-socks[asyncio]` |
| `openpyxl` | `.xlsx` workbook output with conditional formatting | `pip install openpyxl` |
| `selenium` | [DEPRECATED] Legacy JS rendering (use camoufox) | `pip install selenium` |

### Startup dependency check
On startup, the tool logs the detected versions of `aiosmtplib`, `aiodns`, `cloudscraper`, and proxy support status. Missing optional dependencies do not prevent the tool from running; the corresponding feature is simply disabled.

## 7. CRAWL STRATEGY

### Two-phase crawl

1. **Priority Scan** – Quickly visits high-value pages first: `/contact`, `/about`, `/team`, `/security`, `/support`, `/privacy`, security.txt, sitemap.xml, and other known contact-info paths. These typically yield the majority of actionable emails. Runs until the priority URL queue is exhausted.

2. **Deep Crawl** – Full-site breadth-first discovery starting from the homepage. Extracts same-domain links, follows them up to a configurable depth, and parses every page for emails. Cross-domain links are recorded but not followed.

### Email extraction heuristics

- Regex-based email extraction from HTML body text, `mailto:` links, `data-` attributes, JSON-LD structured data, and obfuscated Cloudflare email protection (`/cdn-cgi/l/email-protection#`).
- Emails are classified as **prioritized** (local-part matches known prefixes like `contact`, `support`, `info`, `security`, `hello`, `admin`, `help`, `team`, `office`, etc.) or **other** (sales, community, feedback, etc.).
- Cross-domain emails (addresses found on domain A that belong to domain B) are tracked separately in output files.
- **Phone number extraction** via regex as a bonus contact signal.
- Built-in filtering removes **throwaway/disposable** email domains (Mailinator, GuerrillaMail, etc.), **placeholder emails** (example.com, domain.com, yourname@, etc.), and **artefact email addresses** (local parts matching known non-email patterns like `function`, `var`, `http`, `www`, etc.).

### robots.txt
- **Respected by default** – Parses `robots.txt` Disallow/Allow rules and Crawl-delay.
- `--no-robots` flag disables Disallow checking (Crawl-delay always respected).

### Sitemap
- Sitemap XML is fetched and used to seed the crawl queue (pre-discovery of deep pages).
- `--no-sitemap` flag skips this step entirely.

## 8. CAPABILITIES

- **Stealth TLS fingerprinting** – `curl_cffi` Chrome 120 impersonation makes requests indistinguishable from a real Chrome browser to TLS-fingerprint-aware bot defences (Cloudflare JS challenge, reCAPTCHA v3, DataDome).
- **Two-tier browser fallback** – When Tier-1 returns 403/429/503, Tier-2 camoufox stealth-Firefox is deployed automatically for the blocked domain. Camoufox uses a real Firefox binary under the hood, fully undetectable by current fingerprinting.
- **In-house CAPTCHA / Cloudflare bypass** – The Tier-2 camoufox browser auto-resolves Cloudflare challenges and Turnstile, then harvests `cf_clearance` cookies into a cross-request cookie store for reuse across all threads.
- **Continuous Google Sheet monitoring** (`--url`) – Polls a sheet on a configurable interval, identifies new (previously unprocessed) domains, crawls only those, and maintains a master output folder. Includes cleanup of stale run directories older than 48 hours.
- **Auto-tuning** (`--auto-tune`) – Probes outbound bandwidth using Cloudflare's public speedtest endpoint, then selects optimal thread count and verifier concurrency.
- **Pre-screening** (`--prescreen`) – HEAD-probes every domain before crawling; typically skips 20–35% of lists (unreachable, parked, or firewalled domains), saving significant time on large batches.
- **Crash-resilient queue** (`--persist-queue`) – Persists the crawl frontier to SQLite so a SIGKILL / power loss does not lose progress; resumes on restart.
- **Cross-run deduplication** – SQLite history DB (`EmailHistoryDB`) tracks all seen emails and verification results across runs, enabling de-duplication, querying, and cross-run statistics.
- **Webhook notifications** – POSTs a structured JSON summary (Slack/Discord/Teams compatible) to a configurable URL on run completion. Includes SSRF protection (rejects private/loopback/link-local IPs).
- **.xlsx workbook output** (`--xlsx`) – Combines all CSV sheets into a single Excel workbook with conditional formatting (deliverable=green, undeliverable=red, accept-all=amber).
- **PID file management** (`--pid-file`) – Writes PID for systemd/Docker init; cleaned up on exit via `atexit`.
- **Graceful shutdown** – SIGTERM/SIGHUP signal handlers set a shutdown flag, trigger callbacks (DB close, partial CSV save), and exit cleanly. SIGINT uses the existing KeyboardInterrupt path.
- **Open-file limit raising** – Automatically raises `RLIMIT_NOFILE` to 4096 on Linux to handle hundreds of concurrent SMTP and HTTP sockets.
- **SMTP email verification** – Two modes:
  - **Direct**: Connects to the target domain's MX servers, issues SMTP MAIL FROM/RCPT TO to verify deliverability (without sending mail).
  - **Authenticated**: Routes probes through a user-configured SMTP relay (e.g., SendGrid, Amazon SES).
- **SMTP rate limiting** – Per-domain rate limiting (default 1 probe/second), greylist detection (RFC 6647) with configurable retry delay, accept-all cache (24h TTL, capped at 10k entries).
- **User-agent rotation** – Random selection from a curated pool of realistic browser User-Agent strings.

## 9. ERROR HANDLING

### Crawler errors
- Network failures (timeout, connection refused, DNS failure): logged, domain marked as failed in report, crawl continues to next domain.
- HTTP errors (4xx, 5xx): 403/429/503 trigger Tier-2 browser fallback (if enabled); other errors are logged and skipped.
- Parse errors: malformed HTML is handled gracefully (BeautifulSoup's `html.parser` fallback).

### Verifier errors
- **Greylisting** (RFC 6647 4xx codes): automatically retried after a configurable delay (default 60 seconds) by the `--reverify-unknown` pass.
- **Connection failures**: SMTP timeout/refused/blocked → classified as "unreachable", logged, not retried.
- **Rate limiting**: per-domain rate limiter prevents blacklisting by target MX servers.

### File I/O errors
- Missing input/proxy/exclusion files: logged as ERROR, run aborts early.
- Disk-full / permission errors during output: caught and logged; best-effort partial save attempted.
- History DB errors: never abort the run; failures are logged and verification continues without history tracking.

### Monitoring loop errors
- Sheet fetch failures: logged, cycle skipped, retry up to 5 times with 60-second backoff, then exit.
- Crawl exceptions in monitoring mode: domain batch marked as processed to prevent infinite retry loops.

### Security guards
- **Webhook SSRF protection**: rejects URLs pointing to private/loopback/link-local/metadata IPs.
- **CSV injection protection**: by default, prepends single quote to cells starting with `=`, `+`, `-`, or `@`.
- **TLS verification**: enabled by default for all outbound HTTPS (crawler + SMTP STARTTLS); `--insecure-tls` opt-out only.
- **No paid API calls**: all third-party CAPTCHA/verification services removed; every operation is self-contained and free.

## 10. PERFORMANCE

### Threading model
- **Crawler** uses `ThreadPoolExecutor` (default 10 threads, configurable via `-t/-threads`).
- **Pre-screening** uses a separate `ThreadPoolExecutor` with `max(20, args.threads)` workers.
- **Camoufox Tier-2** uses a dedicated single-thread `ThreadPoolExecutor` (greenlet-bound; all camoufox operations funneled through it to avoid thread-safety issues).
- **SMTP Verifier** uses `asyncio` with `Semaphore`-based concurrency throttling (default 100, adjustable; capped at 500 in interactive mode).

### Throughput
- Typical crawl throughput: 5–20 pages/second per thread (varies by network latency and server response time).
- SMTP verification throughput: 10–100 probes/second (depends on MX server responsiveness and concurrency setting).
- Pre-screening throughput: ~50–200 HEAD probes/second (parallel, short timeout of 5s).

### Resource limits
- Open file descriptors: auto-raised to 4096 soft limit (Linux).
- Accept-all cache: 10,000 entries with 24-hour TTL.
- Robots.txt domain lock cache: 1024 entries (LRU eviction).
- Per-domain SMTP rate: 1 probe/second default.
- SMTP concurrency ceiling: 300 simultaneous connections (safety cap from the code at line ~530).

## 11. TESTING

Run the full validation suite:

```bash
python mailrake.py --self-test
```

This executes:

- **88 smoke checks** – Validates settings persistence, output directory creation, CSV/JSON writing, domain normalization, email extraction, robots.txt parsing, rate limiting, proxy support, greylist detection, disposable email detection, role-address detection, and CLI flag wiring. Output is written to a sandboxed `_selftest_output/` directory.
- **581+ extended unittest tests** – Comprehensive unit tests covering all major components (verifier, crawler, parsers, utilities, DB, etc.) in a separate test harness.

Exit code: `0` if all tests pass, `1` if any fail.

The self-test also validates critical invariants:
- No real network requests during tests (sandboxed DNS/SMTP).
- No filesystem leaks (all test artifacts cleaned up).
- Race conditions tested via multi-threaded concurrent access patterns.

## 12. SETTINGS CONFIGURATION

Settings are stored in `config/config_settings.json` (auto-created with defaults on first run).

### Crawler settings

```json
{
  "crawler_settings": {
    "prioritized_email_prefixes": [
      "contact", "support", "info", "hello", "help", "admin",
      "office", "team", "security", "trust", "vulnerability",
      "disclosure", "bugbounty", "bbp", "vdp", "connect", "getintouch"
    ],
    "other_email_prefixes": [
      "sales", "community", "feedback", "partners", "postmaster",
      "webmaster", "hostmaster", "abuse", "sysadmin", "dev",
      "developer", "privacy", "dpo", "dataprotection", "compliance",
      "legal", "lawenforcement", "copyright", "corrections",
      "permissions", "press", "readers", "reprints", "dispute",
      "dpa", "terms"
    ],
    "common_providers": [
      "gmail.com", "outlook.com", "hotmail.com", "yahoo.com", "aol.com"
    ],
    "primary_target_keywords": [
      "contact", "support", "help", "about", "team", "people",
      "staff", "leadership", "management", "founders", "company",
      "office", "locations", "connect", "get-in-touch", "reach-us",
      "security", "trust", "vulnerability", "disclosure", "bug-bounty",
      "bbp", "vdp"
    ],
    "secondary_target_keywords": [
      "privacy", "gdpr", "data-protection", "terms", "term",
      "terms-of-service", "terms-and-conditions", "compliance",
      "legal", "legal-notice", "law-enforcement", "privacy-policy-short",
      "terms-conditions", "security-and-privacy", "cookie-policy", "ccpa"
    ],
    "proactive_primary_paths": [
      "/contact", "/contact-us", "/support", "/help", "/about",
      "/about-us", "/team", "/our-team", "/company", "/security",
      "/vulnerability-disclosure", "/disclosure-policy", "/bug-bounty",
      "/security.txt", "/.well-known/security.txt", "/bbp", "/vdp"
    ],
    "proactive_secondary_paths": [
      "/privacy", "/privacy-policy", "/terms", "/terms-of-service",
      "/terms-and-conditions", "/legal", "/legal-notice", "/compliance",
      "/privacy-policy-short", "/terms-conditions",
      "/security-and-privacy", "/cookie-policy", "/ccpa"
    ]
  }
}
```

### Verifier settings

```json
{
  "verifier_settings": {
    "direct_check_user": "info",
    "direct_check_host": "example.com",
    "smtp_host": "",
    "smtp_port": 587,
    "smtp_user": "",
    "smtp_pass": "",
    "smtp_tls": true,
    "default_verification_mode": "direct",
    "default_email_column": "B",
    "default_classification_column": "C",
    "default_domain_column": "A",
    "prompt_timeout": 5,
    "default_concurrency": 100,
    "greylist_retry_delay_seconds": 60,
    "smtp_force_ipv4": false
  }
}
```

## 13. SECURITY & PRIVACY

- **No third-party API calls** – All CAPTCHA solving, verification, and crawling is performed locally. No data is sent to external services except (optionally) the user-configured webhook URL.
- **TLS verification on by default** – `--insecure-tls` must be explicitly opted into for intranet/self-signed targets.
- **CSV injection protection** – Formula characters (`=`, `+`, `-`, `@`) are prefixed with a single quote to prevent Excel/LibreOffice code execution on import.
- **Webhook SSRF guard** – Only public HTTPS URLs accepted; private/loopback/link-local/metadata IPs are blocked.
- **SMTP credentials** stored in `config/config_settings.json` – file is user-managed; never committed to version control. Default values are empty placeholders.
- **Cloud metadata endpoint blocking** – 169.254.169.254, metadata.google.internal, 100.100.100.200, etc. are blocked from webhook URLs.
- **Paid solver removals** – All third-party CAPTCHA API integrations (2captcha, capmonster, anticaptcha) were removed; only the free, local in-house bypass exists.
- **No environment variables used** – All configuration is via CLI flags, interactive settings menu, or config JSON file.

## 14. LIMITATIONS & HARDCODED CONSTRAINTS

- **Output naming**: output directory root is hardcoded as `"output"`. Run subdirectories are `output/YYYY-MM-DD/<run_number>/`.
- **Master folder**: monitoring mode writes to `"output/Nixon-S master folder"`.
- **SMTP concurrency ceiling**: 300 simultaneous connections; interactive mode additionally caps at 500.
- **Default threads**: 10; overridden by `--auto-tune` or explicit `-t` flag.
- **Robots.txt domain lock cache**: max 1024 domains; LRU eviction beyond that.
- **TLD cache**: 10,000 entries; `urlparse` cache: 8192 entries.
- **Accept-all cache**: 10,000 entries, 24-hour TTL.
- **HTML parser**: prefers `lxml` if installed, falls back to `html.parser`.
- **Allowed webhook scheme**: `https` only; SSRF guard enforced.
- **Monitoring cleanup interval**: every 48 hours.
- **Stale run cleanup**: directories older than 30 minutes with no output files are removed.
- **Input source exclusivity**: exactly one of `input_file`, `--sheet`, or `--url` must be provided.
- **Single file**: the entire application is a single 22,207-line `mailrake.py` file.

---

> **For detailed technical reference** including data flow diagrams, SMTP protocol, crawl lifecycle, test class listings, email extraction methods, config schema, DB schema, concurrency primitives, module-level constants, and version history — see [`TECHNICAL_REFERENCE.md`](./TECHNICAL_REFERENCE.md).
