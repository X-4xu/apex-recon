# APEX Usage Guide & Comprehensive Manual

This guide covers everything you need to run, configure, and extend **APEX (Evidence-First Security Reconnaissance & Bug Bounty Pipeline)**.

---

## 1. Installation & Environment Setup

APEX is packaged as a **single-file distribution script** (`apex.sh`). You do not need compilers, Docker, or complex runtimes to get started.

### Step 1: Install Core Dependencies
The core engine requires standard Linux utilities and `jq` to parse data and generate reports:

* **Debian / Ubuntu / Kali Linux:**
  ```bash
  sudo apt update && sudo apt install -y bash jq curl dnsutils bsdmainutils util-linux
  ```
* **macOS (Homebrew):**
  ```bash
  brew install bash jq curl bind
  ```
* **Arch Linux:**
  ```bash
  sudo pacman -S bash jq curl bind-tools
  ```

### Step 2: Install Optional Security Tools (Recommended)
While APEX includes a fallback **Native Engine** (powered by `curl`, `dig`, and regex heuristics), integrating industry-standard security tools unlocks its full potential:

| Category | Tool | Official Repository | Installation Command |
| :--- | :--- | :--- | :--- |
| **Subdomains** | `subfinder` | [projectdiscovery/subfinder](https://github.com/projectdiscovery/subfinder) | `go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest` |
| **Subdomains** | `amass` | [owasp-amass/amass](https://github.com/owasp-amass/amass) | `go install -v github.com/owasp-amass/amass/v4/...@master` |
| **Subdomains** | `assetfinder` | [tomnomnom/assetfinder](https://github.com/tomnomnom/assetfinder) | `go install -v github.com/tomnomnom/assetfinder@latest` |
| **DNS Resolver**| `dnsx` | [projectdiscovery/dnsx](https://github.com/projectdiscovery/dnsx) | `go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest` |
| **HTTP Probing**| `httpx` | [projectdiscovery/httpx](https://github.com/projectdiscovery/httpx) | `go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest` |
| **Crawling** | `katana` | [projectdiscovery/katana](https://github.com/projectdiscovery/katana) | `go install -v github.com/projectdiscovery/katana/cmd/katana@latest` |
| **Archive URLs**| `gau` | [lc/gau](https://github.com/lc/gau) | `go install -v github.com/lc/gau/v2/cmd/gau@latest` |
| **Archive URLs**| `waybackurls` | [tomnomnom/waybackurls](https://github.com/tomnomnom/waybackurls) | `go install -v github.com/tomnomnom/waybackurls@latest` |
| **Scanners** | `nuclei` | [projectdiscovery/nuclei](https://github.com/projectdiscovery/nuclei) | `go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest` |
| **Fuzzing** | `ffuf` | [ffuf/ffuf](https://github.com/ffuf/ffuf) | `go install -v github.com/ffuf/ffuf/v2@latest` |
| **XSS** | `dalfox` | [hahwul/dalfox](https://github.com/hahwul/dalfox) | `go install -v github.com/hahwul/dalfox/v2@latest` |
| **Parameters** | `arjun` | [s0md3v/Arjun](https://github.com/s0md3v/Arjun) | `pip install arjun` |
| **Takeover** | `subzy` | [LukaSikic/subzy](https://github.com/LukaSikic/subzy) | `go install -v github.com/LukaSikic/subzy@latest` |
| **SQLi** | `sqlmap` | [sqlmapproject/sqlmap](https://github.com/sqlmapproject/sqlmap) | `sudo apt install sqlmap` or `git clone https://github.com/sqlmapproject/sqlmap.git` |

### Step 3: Validate Installation
Run the built-in health check:
```bash
./apex.sh doctor
```

---

## 2. Running Scans

### Mode A: Standard Recon & Signal Discovery (Default Safe Mode)
To run passive discovery, live asset probing, JavaScript secret mining, and non-destructive vulnerability templates:

```bash
./apex.sh scan example.com --authorized=true
```

#### What this mode touches:
* ✅ Queries public OSINT & certificate transparency logs (`crt.sh`, `subfinder`, `amass`).
* ✅ Sends standard HTTP/HTTPS `GET` and `HEAD` probes to discover live hosts, inspect security headers, and fingerprint tech stacks.
* ✅ Crawls and archives public JavaScript files to detect exposed API tokens via Shannon entropy and regex matching.
* ✅ Executes non-intrusive `nuclei` templates (automatically excluding `intrusive`, `dos`, and `fuzz` tags).

#### What this mode does NOT touch:
* ❌ No aggressive fuzzing, parameter pollution, or SQL injection payloads.
* ❌ No active SSRF outbound interaction attempts.
* ❌ No concurrent race-condition bursts.

---

### Mode B: Active & Intrusive Testing (Explicit Opt-In Required)

> ⚠️ **LEGAL WARNING:** Active intrusive testing sends exploit-style probe strings and concurrent bursts that may trigger web application firewalls (WAFs), log alerts, or cause state changes. Ensure you have **explicit written authorization** before using these flags.

To enable deep active testing, specify `--enable_active=true` along with the individual modules you wish to engage:

```bash
./apex.sh scan example.com \
  --authorized=true \
  --enable_active=true \
  --enable_sqlmap=true \
  --enable_ssrf=true \
  --enable_race=true \
  --race_url="https://api.example.com/v1/redeem-voucher"
```

#### Differential Access Control (IDOR Testing):
You can supply two distinct session cookies to automatically test object-referencing endpoints for Broken Object Level Authorization:

```bash
./apex.sh scan example.com \
  --authorized=true \
  --enable_active=true \
  --session_a="session=userA_token_abc" \
  --session_b="session=userB_token_xyz"
```

---

### Mode C: Target Scope Management

APEX enforces strict **Fail-Closed ScopeGuard** rules:

```bash
# Single apex domain (automatically includes all subdomains)
./apex.sh scan example.com --authorized=true

# Wildcard domain
./apex.sh scan "*.example.com" --authorized=true

# Exclude sensitive or out-of-scope subdomains (Exclusions always take precedence)
./apex.sh scan example.com "!dev.example.com" "!internal.example.com" --authorized=true

# Scope from a file (one domain/exclusion per line, supports # comments)
./apex.sh scan targets_scope.txt --authorized=true

# Wildcard VDP / Bug Bounty Allow-All Mode
./apex.sh scan "*" --authorized=true --scope_allow_all=true
```

---

## 3. Understanding the Outputs & Evidence Gate

Every execution creates an isolated run directory (default: `./apex_runs/<scope>_<timestamp>/`):

```
apex_runs/example.com_20260828_224500/
├── db/
│   ├── findings.ndjson      # All deduplicated findings with cryptographic evidence
│   ├── evidence.ndjson      # Raw HTTP requests/responses & SHA-256 hashes
│   ├── hosts.ndjson         # In-scope discovered hostnames
│   ├── web.ndjson           # Live web assets & HTTP status codes
│   └── urls.ndjson          # Extracted URL endpoints
├── reports/
│   ├── report.html          # Self-contained interactive dashboard
│   ├── report.md            # GitHub-flavored markdown report
│   ├── report.json          # Machine-readable JSON summary
│   ├── risk_ranked.json     # Severity x Confidence prioritized matrix
│   └── changes.md           # Run-to-run delta diff (new vs resolved)
├── logs/
│   └── apex.log             # Execution logs (secrets automatically redacted)
└── raw/                     # Tool stdout captures & raw scan dumps
```

### The Evidence Gate (Confidence Scoring)

APEX scores every finding from **0 to 100** based on verifiable proof:

| Tier | Score Range | Meaning & Action Required |
| :--- | :---: | :--- |
| **`CONFIRMED`** | **90 – 100** | Proven with zero doubt via content signatures (e.g. `.env` file containing verified `KEY=VAL` pairs, or reproducible DNS zone transfers). Ready to report. |
| **`HIGH_CONF`** | **75 – 89** | Verified by dedicated testing tools (e.g. `dalfox` confirmed reflected XSS, or `nuclei` verified match). |
| **`MEDIUM_CONF`** | **50 – 74** | Tool-observed weakness (e.g. GraphQL introspection active, missing sensitive headers, or soft differential 200 responses). |
| **`LOW_CONF`** | **25 – 49** | Informational posture weakness (e.g. missing HSTS/CSP or SPF records). |
| **`NEEDS_REVIEW`** | **0 – 24** | **Unverified heuristic surface.** Findings without raw cryptographic evidence are capped here to prevent false alarms. |

---

## 4. Run-to-Run Diffing & Continuous Monitoring

APEX tracks state between sequential runs, allowing automated CI/CD security pipelines to alert only on **new** vulnerabilities:

```bash
# First baseline run
./apex.sh scan example.com --authorized=true --out_dir=./runs/baseline

# Subsequent run with differential comparison
./apex.sh scan example.com --authorized=true --out_dir=./runs/current --baseline=./runs/baseline
```
Inspect `reports/changes.md` to see exactly which findings are **NEW** and which have been **RESOLVED**.

---

## 5. Resume & Dry-Run Modes

* **Plan without scanning (`--dry_run=true`):**
  Inspect the sequence of phases and tool commands without sending a single network packet:
  ```bash
  ./apex.sh scan example.com --authorized=true --dry_run=true
  ```

* **Resume an interrupted scan (`--resume=true`):**
  If a network interruption occurs, re-run with `--resume=true` to skip already completed phases using `.state/` markers:
  ```bash
  ./apex.sh scan example.com --authorized=true --out_dir=./runs/current --resume=true
  ```

---

## 6. Complete Configuration Flags Reference

Pulled directly from `apex.sh config`:

| Option | Default | Description |
| :--- | :--- | :--- |
| `--threads` | `15` | Parallel workers for host-level tasks |
| `--timeout` | `15` | Per-tool timeout in seconds |
| `--rate` | `150` | Global soft cap on requests/sec |
| `--log_level` | `INFO` | Log verbosity (`DEBUG` \| `INFO` \| `WARN` \| `ERROR`) |
| `--log_json` | `0` | Write logs as JSON lines (`1`) or plain text (`0`) |
| `--authorized` | `false` | **Required:** Confirm you are authorized to test the given scope |
| `--enable_active`| `false` | Enable ANY active/intrusive test (requires authorization) |
| `--enable_ssrf` | `false` | Enable SSRF checks (active; needs `enable_active`) |
| `--enable_race` | `false` | Enable race-condition checks (active; needs `enable_active`) |
| `--enable_sqlmap`| `false` | Enable sqlmap validation (active; needs `enable_active`) |
| `--out_dir` | *(auto)* | Output directory (default: `./apex_runs/<scope>_<ts>`) |
| `--local_lab` | `false` | Local-lab mode: only allow RFC1918/localhost targets |
| `--scope_allow_all` | `false` | ALLOW-ALL scope for wildcard/VDP programs (any valid domain) |
| `--katana_depth` | `3` | Crawl depth for katana |
| `--dry_run` | `false` | Plan the phases and exit without touching any target |
| `--resume` | `false` | Skip phases already completed in the output directory |
| `--retries` | `1` | Automatic retries for transient tool failures |
| `--baseline` | `""` | Path to a previous `findings.ndjson` or run directory for diffing |
| `--max_urls` | `2000` | Max URLs analyzed per URL-driven phase |
| `--max_hosts` | `5000` | Max hosts probed per host-driven phase |
| `--race_url` | `""` | Single endpoint to test for race conditions |
| `--race_count` | `20` | Concurrent requests fired by the race harness |
| `--session_a` | `""` | Cookie header for session A (access-control differential test) |
| `--session_b` | `""` | Cookie header for session B (access-control differential test) |
| `--similarity_threshold` | `92` | Soft-404 / body similarity suppression threshold |
| `--bounty_min_confidence`| `75` | Minimum confidence score for bounty candidates |

---

## 7. Links & Community

* Read the primary [README.md](../README.md) for safety architecture and overview.
* Review [CONTRIBUTING.md](../CONTRIBUTING.md) for developer guidelines and testing.
* Check [LICENSE](../LICENSE) for MIT terms.
