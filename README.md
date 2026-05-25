# MailRake 🕵️‍♂️

[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Code size](https://img.shields.io/github/languages/code-size/Nixon-H/MailRake)](https://github.com/Nixon-H/MailRake)
[![Last commit](https://img.shields.io/github/last-commit/Nixon-H/MailRake)](https://github.com/Nixon-H/MailRake)

**MailRake** is a full-featured, production-grade email harvesting and verification toolkit. It crawls websites for email addresses, phone numbers, and contact metadata, then SMTP-verifies deliverability — all from a single Python file.

```
git clone https://github.com/Nixon-H/MailRake.git
cd MailRake
pip install -r requirements.txt
python mailrake.py domains.txt
```

---

## ✨ Features

- **Web Crawler** – Multi-threaded, robots.txt–respecting, sitemap-aware crawl engine
- **Email Verifier** – Async SMTP-based verification (direct MX probing or authenticated relay)
- **Stealth TLS** – `curl_cffi` Chrome 120 TLS+JA3 impersonation (Tier-1)
- **Browser Fallback** – `camoufox` stealth-Firefox for JS-rendered / Cloudflare-protected sites (Tier-2)
- **In-house CAPTCHA Bypass** – Auto-resolves Cloudflare challenges & Turnstile, harvests `cf_clearance` cookies
- **Continuous Monitoring** – Polls a Google Sheet on a schedule, crawls only new domains
- **Auto-tuning** – Measures outbound bandwidth, picks optimal thread/concurrency settings
- **Crash-resilient Queue** – SQLite-backed `--persist-queue` survives SIGKILL / power loss
- **Cross-run Dedup** – SQLite history DB tracks all seen emails across runs
- **Multi-format Export** – CSV, JSON, `.xlsx` (with conditional formatting), metadata JSON
- **Webhook Notifications** – POSTs structured summaries to Slack / Discord / Teams

---

## 🚀 Quick Start

```bash
# 1. Clone
git clone https://github.com/Nixon-H/MailRake.git

# 2. Install dependencies
cd MailRake
pip install -r requirements.txt

# 3. Run (basic crawl from a text file)
python mailrake.py domains.txt
```

**Example `domains.txt`:**
```
example.com
github.com
stackoverflow.com
```

Wait for the crawl to complete, then check `output/YYYY-MM-DD/<run_number>/` for results.

---

## 📖 Usage

### CLI Reference

```bash
python mailrake.py <input_file> [options]
python mailrake.py --sheet <google_sheet_url> [options]
python mailrake.py --url <google_sheet_url> [options]
python mailrake.py --self-test
python mailrake.py --diagnose-tls
```

### Examples

```bash
# Basic — 10 threads, default settings
python mailrake.py domains.txt

# High-concurrency with stealth TLS
python mailrake.py domains.txt -t 50 --timeout 30 --delay 1.0

# Google Sheet (one-shot) with browser fallback + pre-screening
python mailrake.py --sheet "https://docs.google.com/spreadsheets/d/..." --browser-fallback --prescreen -t 30

# Continuous monitoring — polls every 5 minutes, posts to Slack
python mailrake.py --url "https://docs.google.com/spreadsheets/d/..." --interval 300 --webhook "https://hooks.slack.com/..."

# Fully unattended — auto-run, auto-tune, xlsx output, re-verify unknowns
python mailrake.py domains.csv --one-shot --auto-tune --xlsx --reverify-unknown

# Dry-run — preview without making any network requests
python mailrake.py domains.txt --dry-run

# Run all validation tests (88 smoke + 581+ unit tests)
python mailrake.py --self-test
```

### All Flags

| Category | Flag | Description |
|----------|------|-------------|
| **Input** | `input_file` | `.txt` (one domain per line) or `.csv` (column A = domains) |
| | `--sheet` | Public Google Sheet URL (one-shot fetch) |
| | `--url` | Public Google Sheet URL (continuous monitoring) |
| **Crawl** | `-t, --threads` | Concurrent threads (default: 10) |
| | `--timeout` | Request timeout in seconds (default: 20) |
| | `--delay` | Delay between requests in seconds (default: 0.5) |
| | `--proxies` | File with proxies (one per line) |
| | `--exclude` | File with domains/TLDs to exclude |
| | `--no-robots` | Ignore `robots.txt` Disallow rules |
| | `--no-sitemap` | Skip sitemap.xml pre-seeding |
| | `--prescreen` | HEAD-probe each domain, skip unreachable/parked |
| **Stealth** | `--no-stealth-tls` | Disable curl_cffi impersonation |
| | `--browser-fallback` | Enable Tier-2 camoufox for blocked domains |
| | `--insecure-tls` | Disable TLS certificate verification |
| **Automation** | `--one-shot` / `--auto-run` | Fully unattended, skip all prompts |
| | `--auto-tune` | Auto-detect optimal thread/concurrency settings |
| | `--dry-run` | Parse + report without network I/O |
| | `--interval` | Seconds between monitoring cycles (default: 300) |
| | `--webhook` | POST JSON summary to URL on completion |
| | `--reverify-unknown` | Re-test unknown results at half concurrency |
| | `--xlsx` | Emit `.xlsx` workbook with conditional formatting |
| | `--persist-queue` | Crash-resilient SQLite-backed crawl queue |
| | `--pid-file` | Write PID to file for systemd/Docker |
| **Verifier** | `--history-db` | Path to SQLite history DB |
| | `--no-history-db` | Disable history DB entirely |
| **Diagnostics** | `--self-test` | Run 88 smoke + 581+ unit tests |
| | `--diagnose-tls` | Print TLS+JA3 fingerprint and exit |
| | `-v, --verbose` | Enable INFO-level console logging |

---

## 📦 Dependencies

### Required

| Package | Purpose |
|---------|---------|
| `requests` | HTTP requests (sitemaps, webhooks) |
| `beautifulsoup4` | HTML parsing + email extraction |
| `lxml` | Fast HTML/XML parser (10x stdlib) |
| `cloudscraper` | Cloudflare JS challenge bypass |
| `tld` | Public Suffix List domain extraction |
| `tqdm` | Progress bars |
| `aiosmtplib` | Async SMTP email verification |
| `aiohttp` | Async HTTP (sitemap fetching) |
| `aiodns` | Async DNS resolution |
| `urllib3` | Lower-level HTTP utilities |

### Optional

| Package | Purpose | Install |
|---------|---------|---------|
| `curl_cffi` | Chrome 120 TLS+JA3 impersonation | `pip install curl_cffi` |
| `camoufox` | Stealth Firefox for JS rendering + CAPTCHA bypass | `pip install camoufox && python -m camoufox fetch` |
| `python-socks[asyncio]` | SOCKS/HTTP proxy for the verifier | `pip install python-socks[asyncio]` |
| `openpyxl` | `.xlsx` workbook output | `pip install openpyxl` |

Missing optional deps don't prevent the tool from running — the corresponding feature is simply disabled.

---

## 🔧 How It Works

### Two-Phase Crawl

1. **Priority Scan** – Hits high-value pages first: `/contact`, `/about`, `/team`, `/security`, `/support`, `/privacy`, security.txt, sitemap.xml. These yield the majority of actionable emails.

2. **Deep Crawl** – Full-site BFS discovery from the homepage. Follows same-domain links, parses every page for emails. Cross-domain links are recorded but not followed.

### Email Extraction

- Regex extraction from HTML body, `mailto:` links, `data-` attributes, JSON-LD, and Cloudflare email protection (`/cdn-cgi/l/email-protection#`)
- Emails classified as **prioritized** (contact, support, info, security, hello…) or **other** (sales, community, feedback…)
- Cross-domain emails tracked separately
- Phone number extraction via regex
- Filters: throwaway domains (Mailinator, GuerrillaMail), placeholder emails (example.com, yourname@), artefact patterns

### Two-Tier Fetching

| Tier | Library | Purpose |
|------|---------|---------|
| 1 | `curl_cffi` | Chrome 120 TLS+JA3 impersonation — indistinguishable from real Chrome |
| 2 | `camoufox` | Stealth Firefox for domains returning 403/429/503; auto-resolves Cloudflare |

### SMTP Verification

Two modes:
- **Direct**: Connects to target MX servers, issues `MAIL FROM`/`RCPT TO` (no email sent)
- **Authenticated**: Routes through a user-configured relay (SendGrid, SES, etc.)

Features: per-domain rate limiting (1/s), greylist detection with retry, accept-all cache (24h TTL, 10k cap), IPv4 forcing.

---

## 📁 Output Structure

```
output/YYYY-MM-DD/<run_number>/
├── JSON/
│   ├── <base>_emails.json          # Per-domain email map
│   └── <base>_crossdomain.json     # Cross-domain emails
├── CSV/
│   ├── <base>_emails.csv           # Domain, Email, Classification, Source_Type
│   ├── <base>_report.csv           # Per-domain crawl report
│   ├── <base>_phones.csv           # Phone numbers (if any)
│   └── <base>_verified_emails.csv  # Verifier deliverability results
├── <base>.xlsx                     # Combined workbook (if --xlsx)
├── metadata.json                   # Run summary: timestamps, counts, args, SHA
└── Logs/
    └── domain_crawler.log          # Structured INFO-level log
```

---

## 🧪 Testing

```bash
python mailrake.py --self-test
```

- **88 smoke checks** — settings persistence, CSV/JSON I/O, extraction, robots.txt, rate limiting, proxy support
- **581+ unit tests** — comprehensive coverage of verifier, crawler, parsers, utilities, DB

Exit code `0` = all pass, `1` = failure.

---

## ⚙️ Configuration

Settings stored in `config/config_settings.json` (auto-created on first run). Key sections:

- **`crawler_settings`** – prioritized/other email prefixes, target keywords, priority crawl paths
- **`verifier_settings`** – SMTP relay credentials, concurrency, greylist retry delay

Edit the JSON file directly or use the interactive menu (run without arguments).

---

## 🛡️ Security

- **No third-party API calls** – all CAPTCHA solving & verification is local
- **TLS verification on by default** – `--insecure-tls` opt-in only
- **CSV injection protection** – formula characters (`= + - @`) prefixed with `'`
- **Webhook SSRF guard** – blocks private/loopback/link-local/metadata IPs
- **Cloud metadata blocked** – 169.254.169.254, metadata.google.internal, etc.
- **Credentials** stored in `config/config_settings.json` — never committed

---

## ⚠️ Limitations

| Constraint | Value |
|------------|-------|
| Output root | `output/` (hardcoded) |
| SMTP concurrency cap | 300 connections (500 in interactive mode) |
| Default threads | 10 (overridden by `--auto-tune` or `-t`) |
| Robots lock cache | 1024 domains (LRU eviction) |
| Accept-all cache | 10,000 entries, 24-hour TTL |
| URL cache | 8,192 entries |
| TLD cache | 10,000 entries |
| Input sources | Exactly one of: file, `--sheet`, `--url` |
| File size | Single 22,207-line `mailrake.py` |

---

## 📚 Documentation

- **README.md** (this file) — user-facing docs: features, usage, examples
- **[TECHNICAL_REFERENCE.md](./TECHNICAL_REFERENCE.md)** — developer reference: data flow, SMTP protocol, crawl lifecycle, 211 test classes, 12 extraction methods, config schema, DB schema, concurrency inventory, version history

---

## 🙏 Acknowledgments

- All open-source library authors whose packages make this possible
- The cybersecurity and OSINT research community

---

## ⚖️ Disclaimer

This tool is intended for **legal security research, penetration testing, and educational purposes only**. Users are solely responsible for ensuring their usage complies with all applicable laws and regulations. The authors assume no liability for misuse or damage caused by this toolkit.

---

*Built with ❤️ by [Nixon-H](https://github.com/Nixon-H)*
