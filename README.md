# MailRake 🕵️‍♂️

[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Code size](https://img.shields.io/github/languages/code-size/Nixon-H/MailRake)](https://github.com/Nixon-H/MailRake)
[![Last commit](https://img.shields.io/github/last-commit/Nixon-H/MailRake)](https://github.com/Nixon-H/MailRake)
[![Total tests](https://img.shields.io/badge/tests-669%20passing-brightgreen)]()

MailRake is a full-featured, production-grade email harvesting and verification toolkit. It crawls websites for email addresses, phone numbers, and contact metadata; then SMTP-verifies deliverability. Supports stealth TLS fingerprinting, browser-based JavaScript rendering, Cloudflare bypass, continuous Google Sheet monitoring, automated concurrency tuning, webhook notifications, and a cross-run SQLite history database — all from a single Python file.

---

## ✨ Features

- **Web Crawler** – Multi-threaded, robots.txt–respecting, sitemap-aware crawl engine
- **Email Verifier** – Async SMTP-based verification (direct MX probing or authenticated relay)
- **Stealth TLS Fingerprinting** – `curl_cffi` Chrome 120 TLS+JA3 impersonation (Tier-1)
- **Browser Fallback** – `camoufox` stealth-Firefox for JS-rendered / Cloudflare-protected sites (Tier-2)
- **In-house CAPTCHA Bypass** – Auto-resolves Cloudflare challenges & Turnstile, harvests `cf_clearance` cookies
- **Continuous Monitoring** – Polls a Google Sheet on a configurable interval, crawls only new domains
- **Auto-tuning** – Measures outbound bandwidth via Cloudflare speedtest, picks optimal thread/concurrency
- **Crash-resilient Queue** – SQLite-backed `--persist-queue` survives SIGKILL / power loss
- **Cross-run Deduplication** – SQLite history DB tracks all seen emails across runs
- **Multi-format Export** – CSV, JSON, `.xlsx` (with green/red/amber conditional formatting)
- **Webhook Notifications** – POSTs structured summaries to Slack / Discord / Teams
- **Pre-screening** – HEAD-probes domains before crawling, skips unreachable/parked/firewalled

---

## 🚀 Quick Start

### Prerequisites

- **Python 3.10+** required
- **pip** for dependency installation
- **Internet connection** for crawling
- **Optional:** `curl_cffi` for TLS impersonation, `camoufox` for browser fallback

### Installation

```bash
# Clone the repository
git clone https://github.com/Nixon-H/MailRake.git
cd MailRake

# Install dependencies
pip install -r requirements.txt

# Optional: Install stealth TLS support for better evasion
pip install curl_cffi

# Optional: Install browser-based JS rendering + CAPTCHA bypass
pip install camoufox && python -m camoufox fetch
```

### First Run

```bash
# Create a domains file
echo "example.com" > domains.txt
echo "github.com" >> domains.txt

# Run the scraper
python mailrake.py domains.txt
```

You'll see an interactive menu:

```
╔═══════════════════════════════════════════════════════╗
║              MailRake — Advanced Email Scraper         ║
╠═══════════════════════════════════════════════════════╣
║  1. Run Email Scraper                                  ║
║  2. Configure Settings                                 ║
║  3. View/Edit Settings                                 ║
║  4. Manage History Database                            ║
║  5. View Statistics                                    ║
║  6. Exit                                               ║
╚═══════════════════════════════════════════════════════╝
```

Select option `1`, then:
1. Choose input source (file / Google Sheet)
2. Confirm crawl settings
3. Wait for completion — results land in `output/YYYY-MM-DD/<run_number>/`

For fully unattended operation, use CLI flags instead (see below).

---

## 📖 CLI Reference

### Basic Syntax

```bash
python mailrake.py <input_file> [options]
python mailrake.py --sheet <google_sheet_url> [options]
python mailrake.py --url <google_sheet_url> [options]
python mailrake.py --self-test
python mailrake.py --diagnose-tls
python mailrake.py -h, --help          # Show full help
```

### Core Arguments

| Argument | Description |
|----------|-------------|
| `input_file` | `.txt` (one domain per line) or `.csv` (column A holds domains). Google Sheet URL auto-detected |
| `--sheet` | Public Google Sheet URL (one-shot fetch) |
| `--url` | Public Google Sheet URL (continuous monitoring, default 5-minute interval) |
| `-t, --threads` | Number of concurrent threads (default: 10). Range: 1–200 |
| `--timeout` | Request timeout in seconds (default: 20). Increase for slow sites |
| `--delay` | Delay between requests in seconds (default: 0.5). Helps avoid rate limiting |
| `-o, --output` | Base filename for output files (default: "results") |
| `--proxies` | File with proxies (one per line). Format: `http://ip:port` or `socks5://ip:port` |
| `--exclude` | File with domains/TLDs to exclude (one per line). E.g., `example.com`, `.gov` |

### Stealth & Evasion Flags

| Flag | Description |
|------|-------------|
| `--no-stealth-tls` | Disable `curl_cffi` Chrome 120 TLS+JA3 impersonation (fallback to cloudscraper). Use if curl_cffi causes connectivity issues |
| `--browser-fallback` | Enable Tier-2 camoufox stealth-Firefox for domains returning 403/429/503. Required for Cloudflare-protected sites |
| `--selenium` | [DEPRECATED] Use `--browser-fallback` instead |
| `--no-robots` | Ignore `robots.txt` Disallow rules (Crawl-delay still respected) |
| `--no-sitemap` | Skip sitemap.xml pre-seeding. Saves one HTTP request per domain |
| `--prescreen` | HEAD-probe each domain first, skip unreachable/parked/firewalled ones. Typically skips 20–35% of lists |
| `--insecure-tls` | Disable TLS certificate verification on outbound HTTPS. Use only for intranet/self-signed targets |

### Automation & Operations Flags

| Flag | Description |
|------|-------------|
| `--one-shot, --auto-run` | Fully unattended: skip prompts, auto-run deep crawl + verifier. Useful for cron/CI |
| `--auto-tune` | Measure outbound bandwidth via Cloudflare speedtest and auto-pick threads/concurrency |
| `--dry-run` | Parse and report what would be crawled, then exit without network I/O |
| `--self-test` | Run 88 smoke checks + 581+ unittest tests and exit. Exit code 0 = all pass |
| `--diagnose-tls` | Print the TLS+JA3 fingerprint your crawler presents, then exit |
| `--interval` | Seconds between checks in `--url` monitoring mode (default: 300, min: 60) |
| `--history-db` | Path to SQLite history DB (default: `output/email_history.db`) |
| `--no-history-db` | Disable history DB entirely. Runs without cross-run dedup |
| `--webhook` | POST JSON run-summary to this URL on completion. Slack/Discord/Teams compatible |
| `--reverify-unknown` | Re-run "unknown" verification results once at half concurrency |
| `--xlsx` | Emit `.xlsx` workbook with green/red/amber conditional formatting (requires openpyxl) |
| `--pid-file` | Write PID to file (removed on clean shutdown). Useful for systemd/Docker |
| `--persist-queue` | Persist crawl queue to SQLite disk for crash recovery |
| `--no-csv-escape` | Disable CSV-injection escaping of leading `=/-+/@` characters |
| `--captcha-solver` | Only `"in_house"` supported (uses Tier-2 camoufox to auto-resolve Cloudflare). Default: in_house |
| `--captcha-api-key` | [DEPRECATED] No-op; kept for backwards compatibility |
| `-v, --verbose` | Enable INFO-level console logging |

---

## 📝 Examples

### Basic Usage

```bash
# Basic crawl from text file (10 threads)
python mailrake.py domains.txt

# High-concurrency with stealth TLS and longer timeout
python mailrake.py domains.txt -t 50 --timeout 30 --delay 1.0
```

### Google Sheets Integration

```bash
# One-shot: fetch domains from a public sheet and crawl
python mailrake.py --sheet "https://docs.google.com/spreadsheets/d/..." --browser-fallback --prescreen -t 30

# Continuous: monitor sheet every 5 minutes, post results to Slack
python mailrake.py --url "https://docs.google.com/spreadsheets/d/..." --interval 300 --webhook "https://hooks.slack.com/..."
```

### Automated & Production

```bash
# Fully unattended with auto-tuning, xlsx output, and unknown re-verification
python mailrake.py domains.csv --one-shot --auto-tune --xlsx --reverify-unknown

# With proxy rotation + exclude list
python mailrake.py domains.txt --proxies proxies.txt --exclude exclude.txt -t 20

# Crash-resilient: survive power loss and resume
python mailrake.py domains.txt --persist-queue --one-shot

# Dry-run to preview what would be crawled
python mailrake.py domains.txt --dry-run
```

### Diagnostics

```bash
# Run full validation suite
python mailrake.py --self-test

# Check what TLS fingerprint your crawler presents
python mailrake.py --diagnose-tls
```

---

## ⚙️ Advanced Features

### Stealth TLS Fingerprinting (Tier-1)

By default, MailRake uses `curl_cffi` to impersonate a Chrome 120 browser at the TLS+JA3 level. This makes requests indistinguishable from a real Chrome browser to fingerprint-aware bot defences (Cloudflare JS challenge, reCAPTCHA v3, DataDome, PerimeterX).

```bash
# Explicitly enable (default: on)
python mailrake.py domains.txt

# Disable if causing issues
python mailrake.py domains.txt --no-stealth-tls
```

### Browser Fallback (Tier-2)

When Tier-1 returns 403/429/503, Tier-2 camoufox stealth-Firefox is deployed **automatically** per-domain. Camoufox uses a real Firefox binary, fully undetectable by current fingerprinting. All camoufox operations run through a dedicated single thread to avoid thread-safety issues.

```bash
python mailrake.py domains.txt --browser-fallback
```

### In-house Cloudflare / CAPTCHA Bypass

The Tier-2 camoufox browser auto-resolves Cloudflare challenges (JS challenge, Turnstile, reCAPTCHA v2/v3) and harvests `cf_clearance` / `turnstile-token` cookies into a cross-request cookie store. The cookie store is shared across all crawl threads, so only one thread needs to solve the challenge per domain.

No paid third-party CAPTCHA solvers required. No API keys needed.

### Auto-tuning Bandwidth Detection

The `--auto-tune` flag probes outbound bandwidth using Cloudflare's public speedtest endpoint (`https://speed.cloudflare.com/__down?bytes=...`), then selects optimal thread count and verifier concurrency:

- **< 10 Mbps:** 5 threads, 20 verifier concurrency
- **10–50 Mbps:** 15 threads, 50 verifier concurrency
- **50–100 Mbps:** 30 threads, 100 verifier concurrency
- **> 100 Mbps:** 50 threads, 200 verifier concurrency (or user-specified `-t` cap)

```bash
python mailrake.py domains.txt --auto-tune --one-shot
```

### Continuous Sheet Monitoring

The `--url` flag polls a Google Sheet on a configurable interval (`--interval`, default 300s). Each cycle:

1. Fetches the sheet
2. Cross-references against `output/processed_domains.log`
3. Crawls only new (previously unprocessed) domains
4. Writes results to a master folder (`output/Nixon-S master folder/`)
5. Cleans up stale run directories older than 48 hours

```bash
python mailrake.py --url "https://docs.google.com/spreadsheets/d/..." --interval 300 --webhook "https://hooks.slack.com/..."
```

### Crash-resilient Queue

The `--persist-queue` flag backs the crawl frontier to SQLite. If the process is killed (SIGKILL, power loss, OOM), re-running with `--persist-queue` resumes from where it left off instead of re-crawling everything.

```bash
python mailrake.py domains.txt --persist-queue --one-shot
```

### Pre-screening

The `--prescreen` flag HEAD-probes every domain before the main crawl. Domains that are unreachable, parked, firewalled, or return non-HTTP responses are skipped. This typically eliminates 20–35% of domains upfront, saving significant time on large lists.

```bash
python mailrake.py domains.txt --prescreen -t 50
```

### SMTP Email Verification

Two modes available:

**Direct Verification (default):** Connects to the target domain's MX servers, issues SMTP `MAIL FROM`/`RCPT TO` to check deliverability without sending email.

```bash
python mailrake.py domains.txt
```

**Authenticated Relay:** Routes verification through a user-configured SMTP relay (e.g., SendGrid, Amazon SES). Configure via interactive menu or `config/config_settings.json`:

```json
{
  "verifier_settings": {
    "smtp_host": "smtp.sendgrid.net",
    "smtp_port": 587,
    "smtp_user": "apikey",
    "smtp_pass": "SG.xxxxx",
    "smtp_tls": true,
    "default_verification_mode": "authenticated"
  }
}
```

Features:
- Per-domain rate limiting (default 1 probe/second)
- Greylist detection (RFC 6647) with configurable retry delay (default 60s)
- Accept-all cache: 24-hour TTL, capped at 10,000 entries
- IPv4 forcing option for IPv6-problematic servers
- Safety ceiling: 300 simultaneous connections

### Webhook Notifications

On run completion, MailRake POSTs a structured JSON summary to a configurable URL. Compatible with Slack, Discord, Teams, and custom webhook receivers.

```json
{
  "status": "completed",
  "run_id": "20260525_123456_abc123",
  "domains_crawled": 50,
  "emails_found": 142,
  "deliverable": 89,
  "undeliverable": 31,
  "accept_all": 22,
  "duration_seconds": 345,
  "errors": 0
}
```

SSRF protection: rejects URLs pointing to private/loopback/link-local/metadata IPs.

### .xlsx Workbook Output

The `--xlsx` flag combines all CSV sheets into a single Excel workbook with conditional formatting:

- **Green** = deliverable
- **Red** = undeliverable
- **Amber** = accept-all / unknown

Requires `openpyxl`: `pip install openpyxl`

```bash
python mailrake.py domains.txt --xlsx
```

---

## 🔧 How It Works

### Two-Phase Crawl Strategy

1. **Priority Scan** – Quickly visits high-value pages first: `/contact`, `/about`, `/team`, `/security`, `/support`, `/privacy`, security.txt, sitemap.xml, and other known contact-info paths. These typically yield the majority of actionable emails. Runs until the priority URL queue is exhausted.

2. **Deep Crawl** – Full-site breadth-first discovery starting from the homepage. Extracts same-domain links, follows them up to a configurable depth, and parses every page for emails. Cross-domain links are recorded but not followed.

### Email Extraction Heuristics

- Regex-based extraction from HTML body text, `mailto:` links, `data-` attributes, JSON-LD structured data, and `cloudscraper`-decoded Cloudflare email protection (`/cdn-cgi/l/email-protection#`)
- Emails are classified as **prioritized** (local-part matches known prefixes like `contact`, `support`, `info`, `security`, `hello`, `admin`, `help`, `team`, `office`) or **other** (sales, community, feedback, etc.)
- Cross-domain emails (addresses on domain A belonging to domain B) tracked separately
- **Phone number extraction** via regex as a bonus contact signal
- Built-in filtering removes:
  - **Throwaway/disposable domains** (Mailinator, GuerrillaMail, 10minutemail, etc.)
  - **Placeholder emails** (example.com, domain.com, yourname@, you@, etc.)
  - **Artefact patterns** (local parts matching `function`, `var`, `http`, `https`, `www`, `const`, `let`, `this`, `null`, `undefined`, `true`, `false`)

### robots.txt & Sitemap

- **Respected by default** – Parses `robots.txt` Disallow/Allow rules and Crawl-delay. Sitemap XML fetched for queue seeding.
- `--no-robots` – Disables Disallow checking (Crawl-delay always respected).
- `--no-sitemap` – Skips sitemap fetch entirely.

### Two-Tier Fetching Model

| Tier | Library | Technique | Purpose |
|------|---------|-----------|---------|
| 1 | `curl_cffi` | Chrome 120 TLS+JA3 impersonation | Indistinguishable from real Chrome at TLS level |
| 2 | `camoufox` | Stealth Firefox with human-like jitter | For domains returning 403/429/503; auto-resolves Cloudflare |

Tier-2 is only activated per-domain when Tier-1 returns a block/error. The cookie store shares `cf_clearance` tokens across all threads.

---

## 📂 Output Structure

```
output/YYYY-MM-DD/<run_number>/
├── JSON/
│   ├── <base>_emails.json              # Per-domain email map (prioritized + other, target + cross-domain)
│   └── <base>_crossdomain.json         # Cross-domain emails separately
├── CSV/
│   ├── <base>_emails.csv               # Columns: Domain, Email, Classification, Source_Type
│   ├── <base>_report.csv               # Per-domain: Domain, Status (Success/Failed), Details
│   ├── <base>_phones.csv               # Phone numbers (if any)
│   └── <base>_verified_emails.csv      # Verifier results with deliverability classification
├── <base>.xlsx                         # Combined workbook (if --xlsx)
├── metadata.json                       # Run summary: run_id, timestamps, counts, CLI args, git SHA, platform
└── Logs/
    └── domain_crawler.log              # Structured INFO-level log

# In continuous monitoring mode (--url):
output/Nixon-S master folder/           # Aggregate master logs across cycles
output/processed_domains.log            # Tracks processed domains to avoid re-crawls
```

---

## 🧪 Testing

Run the full validation suite:

```bash
python mailrake.py --self-test
```

This executes:
- **88 smoke checks** – Validates settings persistence, output directory creation, CSV/JSON writing, domain normalization, email extraction, robots.txt parsing, rate limiting, proxy support, greylist detection, disposable email detection, role-address detection, and CLI flag wiring. Output written to `_selftest_output/`.
- **581+ extended unittest tests** – Comprehensive unit tests covering all major components (verifier, crawler, parsers, utilities, DB, etc.)

**Exit code:** `0` if all pass, `1` if any fail.

### What the tests cover

- No real network requests during tests (sandboxed DNS/SMTP)
- No filesystem leaks (all test artifacts cleaned up)
- Race conditions via multi-threaded concurrent access patterns
- CSV injection guard end-to-end
- Cloudflare cookie store LRU eviction + TTL
- Persisted queue cross-instance resume
- Adaptive concurrency ramp/back-off

---

## 📦 Dependencies

### Required (auto-installed via `pip install -r requirements.txt`)

| Package | Version | Purpose |
|---------|---------|---------|
| `requests` | >=2.31 | HTTP for sitemaps, webhooks |
| `beautifulsoup4` | >=4.12 | HTML parsing + email extraction |
| `cloudscraper` | >=1.2.71 | Cloudflare JS challenge bypass (Tier-1 fallback) |
| `aiohttp` | >=3.9 | Async HTTP for sitemap fetching |
| `aiodns` | >=3.1 | Async DNS resolution |
| `aiosmtplib` | *latest* | Async SMTP email verification |
| `lxml` | *latest* | Fast HTML/XML parser (10x stdlib) |
| `tld` | *latest* | Public Suffix List domain extraction |
| `tqdm` | *latest* | Progress bars |
| `urllib3` | *latest* | Lower-level HTTP utilities |

### Optional

| Package | Purpose | Install |
|---------|---------|---------|
| `curl_cffi` | Chrome 120 TLS+JA3 impersonation (Tier-1 stealth) | `pip install curl_cffi` |
| `camoufox` | Stealth Firefox for Tier-2 JS rendering + CAPTCHA bypass | `pip install camoufox && python -m camoufox fetch` |
| `python-socks[asyncio]` | SOCKS/HTTP proxy for the async verifier | `pip install python-socks[asyncio]` |
| `openpyxl` | `.xlsx` workbook with conditional formatting | `pip install openpyxl` |
| `selenium` | [DEPRECATED] Legacy JS rendering (use camoufox) | `pip install selenium` |

**Note:** Missing optional deps do not prevent the tool from running. The corresponding feature is simply disabled. On startup, the tool logs detected versions and missing capabilities.

---

## ⚙️ Configuration Reference

Settings are stored in `config/config_settings.json` (auto-created with defaults on first run). Edit directly or via the interactive settings menu.

### Crawler Settings

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
      "permissions", "press", "readers", "reprints", "dispute", "dpa", "terms"
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

### Verifier Settings

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

---

## 🛡️ Security & Privacy

- **No third-party API calls** – All CAPTCHA solving, verification, and crawling is performed locally. No data is sent to external services except (optionally) the user-configured webhook URL.
- **TLS verification on by default** – `--insecure-tls` must be explicitly opted into for intranet/self-signed targets.
- **CSV injection protection** – Formula characters (`=`, `+`, `-`, `@`) are prefixed with a single quote to prevent Excel/LibreOffice code execution on import.
- **Webhook SSRF guard** – Only public HTTPS URLs accepted; private/loopback/link-local/metadata IPs are blocked (169.254.169.254, metadata.google.internal, 100.100.100.200).
- **SMTP credentials** stored in `config/config_settings.json` – user-managed; never committed to version control. Default values are empty placeholders.
- **Paid solver removals** – All third-party CAPTCHA API integrations (2captcha, capmonster, anticaptcha) removed; only free, local in-house bypass remains.
- **Graceful shutdown** – SIGTERM/SIGHUP triggers DB close and partial CSV save. No data corruption.
- **Open-file limit raising** – Automatically raises `RLIMIT_NOFILE` to 4096 on Linux.
- **No environment variables used** – All configuration via CLI flags, interactive menu, or config JSON.

---

## 🔧 Troubleshooting

### Crawl Hangs or Returns No Emails

**Symptom:** Spinner runs for 2+ minutes without progress, or crawl finishes with 0 emails.

**Solutions:**

1. Press `Ctrl+C` to abort
2. Check `output/YYYY-MM-DD/<run_number>/Logs/domain_crawler.log` for errors
3. Common causes:
   - **Network issues** – Try increasing timeout: `--timeout 30`
   - **Cloudflare block** – Add `--browser-fallback` for JS-rendered sites
   - **TLS fingerprint blocked** – Try `--no-stealth-tls`
   - **Too many threads** – Reduce with `-t 5`

### SMTP Verification All "Unknown"

**Symptom:** All emails classified as "unknown" after verification.

**Solutions:**

- Add `--reverify-unknown` to retry with lower concurrency
- Check if port 25 is blocked by your ISP (common for residential connections)
- Configure an authenticated relay in the settings menu
- Try `--insecure-tls` if the MX server has a bad certificate

### "Failed to open input file"

**Symptom:** Error about missing input file or invalid format.

**Solutions:**

- Ensure the file exists and is readable
- Format: one domain per line in `.txt`, or column A in `.csv`
- Remove protocol prefixes (`https://`) – domain names only
- Check for hidden characters or BOM markers

### Rate Limiting / IP Blocked

**Symptom:** All requests return 429 or connection refused.

**Solutions:**

- Add delays: `--delay 2.0`
- Reduce threads: `-t 5`
- Use proxies: `--proxies proxies.txt`
- Enable stealth TLS (default) – rotate JA3 fingerprints

### "Could not get lock" / Database Errors

**Symptom:** History DB errors or crawl queue issues.

**Solutions:**

- Remove the history DB: `rm output/email_history.db`
- Disable history: `--no-history-db`
- The tool never aborts on DB errors – best-effort mode engages

---

## ⚠️ Important Notes

### System Impact

- **Open file descriptors** – Auto-raised to 4096 on Linux for concurrent SMTP + HTTP
- **Memory usage** – ~200–500 MB during full crawl (depends on domain count)
- **Disk space** – Minimal (~10 MB for code + dependencies); output depends on crawl size

### Performance

| Metric | Typical Range |
|--------|---------------|
| Crawl throughput | 5–20 pages/sec/thread |
| SMTP verification | 10–100 probes/sec |
| Pre-screening | 50–200 HEAD probes/sec |
| Memory (idle) | ~80 MB |
| Memory (full crawl, 50 threads) | ~400 MB |

### Hardcoded Constraints

| Constraint | Value |
|------------|-------|
| Output root directory | `output/` (hardcoded) |
| SMTP concurrency ceiling | 300 connections (500 in interactive mode) |
| Default threads | 10 |
| Robots.txt domain lock cache | 1024 entries (LRU) |
| Accept-all cache | 10,000 entries, 24-hour TTL |
| `urlparse()` cache | 8,192 entries |
| TLD cache | 10,000 entries |
| Input sources | Exactly one: file, `--sheet`, or `--url` |
| File size | Single `mailrake.py` file (22,207 lines) |

---

## 🤝 Contributing

Contributions are welcome! Here's how to help:

### Reporting Issues

When reporting issues, please include:

- Python version (`python --version`)
- Operating system
- Full error output or log file
- CLI flags used
- Expected vs actual behavior

### Adding Features / Fixing Bugs

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Make your changes
4. Ensure tests pass: `python mailrake.py --self-test`
5. Submit a pull request

---

## 📚 Documentation

- **README.md** (this file) – User-facing docs: features, quick start, usage, examples, troubleshooting
- **[TECHNICAL_REFERENCE.md](./TECHNICAL_REFERENCE.md)** – Developer reference: data flow diagrams, SMTP protocol details, full crawl lifecycle, 211 test class listings, 12 extraction methods, complete config schema, database schema, concurrency primitives inventory, module-level constants, and full version history

---

## 🙏 Acknowledgments

- All open-source library authors whose packages make this possible
- The OSINT and cybersecurity research community

---

## ⚖️ Disclaimer

This tool is intended for **legal security research, penetration testing, and educational purposes only**. Users are solely responsible for ensuring their usage complies with all applicable laws and regulations. The authors assume no liability for misuse or damage caused by this toolkit.

---

*Built with ❤️ by [Nixon-H](https://github.com/Nixon-H)*
