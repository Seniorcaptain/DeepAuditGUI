# DeepAuditGUI
# AuditForensics Advanced v10.5

Full-platform security audit, forensics, and agent management tool with a modern Streamlit UI.

**Authorised use only.** Scan and deploy agents only on systems you own or have written permission to test.

---

## Table of contents

1. [What it is](#what-it-is)
2. [Features](#features)
3. [How it works](#how-it-works)
4. [Requirements](#requirements)
5. [Install](#install)
6. [First-time setup](#first-time-setup)
7. [How to use](#how-to-use)
8. [Web application scans](#web-application-scans)
9. [Results, threat model & exports](#results-threat-model--exports)
10. [Project layout](#project-layout)
11. [Optional modules](#optional-modules)
12. [Environment variables](#environment-variables)
13. [Troubleshooting](#troubleshooting)
14. [Security notes](#security-notes)

---

## What it is

AuditForensics is a local (or self-hosted) security platform that combines:

- Host and configuration audits  
- Network discovery and service mapping  
- Active Directory posture checks  
- Web application checks (headers, TLS, path probing, subdomains)  
- Threat modeling and hunt / detect / triage guidance  
- Optional endpoint agents and a manager API  
- Compliance mapping, history, and export formats  

The main entry point is a **Streamlit** web UI: `auditforensics_web.py`.

---

## Features

| Area | What you get |
|------|----------------|
| **Local machine** | Host hardening checks, packages, common misconfigurations |
| **Network Map** | IP-range discovery, ports, banners, topology, CVE hints |
| **Active Directory** | Kerberoastable / AS-REP accounts, delegation, domain posture |
| **Web Application** | Security headers, TLS, path checks, JS endpoints, optional subdomains |
| **Path / directory checks** | Built-in high-value paths; optional SecLists wordlists with time limits |
| **Threat model** | Separate stream: actors, attack paths, MITRE, detect / hunt / triage |
| **Risk score** | 0–100 score and priority mix from open findings |
| **Remediation checklist** | Prioritised actions from audit + threat model |
| **ATT&CK Navigator** | Export a Navigator layer JSON from techniques on findings |
| **Agents** | Optional endpoint agents + manager API |
| **Auth** | Local users, roles (admin / analyst / auditor), MFA, idle timeout |
| **Exports** | CSV, JSON, SARIF, HTML; threat-model Markdown/CSV |

---

## How it works

```
┌─────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│  Streamlit  │────▶│  Scanners & modules  │────▶│  Results store  │
│  UI + auth  │     │  (web, host, AD…)    │     │  session + DB   │
└─────────────┘     └──────────────────────┘     └────────┬────────┘
                                                          │
                    ┌──────────────────────┐              │
                    │ Threat model / hunt  │◀─────────────┘
                    │ Risk score / exports │
                    └──────────────────────┘
```

1. **You sign in** (`af_auth.py`). Idle sessions expire (default 3 hours; overridable).
2. **You pick a scan type** and a target you are allowed to test.
3. **Scope checks** (`af_scope.py`) enforce allow/deny lists, size caps, and authorization for external targets.
4. **Scanners run** in-process (or via agents). Path probing is **time-budgeted** so long wordlists do not hang the UI or expire your login.
5. **Findings** are normalized (`make_result`), enriched (MITRE, optional CVE intel), and saved to local history.
6. **Threat analysis** (`af_threatmodel.py`) builds a **separate** stream of threat cards and hunt/triage steps from open issues.
7. **Results tab** shows metrics, charts, audit table, threat model, remediation checklist, and downloads.

Web path checks use soft-404 / baseline comparison so generic “everything is 200” sites produce fewer false positives.

---

## Requirements

- **Python 3.10+** (3.11 or 3.12 recommended)
- Network access only as needed for the targets you scan
- Optional: SecLists tree next to the app (or under `~/.auditforensics/SecLists`) for larger wordlists

---

## Install

### 1. Unpack

```bash
unzip auditforensics_v10.5_seclists_fuzz.zip -d auditforensics
cd auditforensics
```

### 2. Virtual environment

```bash
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
python -m pip install --upgrade pip
```

### 3. Core dependencies

On macOS, install `cryptography` from a **binary wheel** first if pip tries to compile and fails (Rust / LLVM errors):

```bash
pip install --only-binary=:all: "cryptography>=42,<51"
pip install -r requirements.txt
```

Optional extras (LDAP, YARA, cloud SDKs, report libraries, etc.):

```bash
pip install -r requirements-optional.txt
```

Skip any optional package that fails; the UI still starts without it.

### 4. Run

```bash
streamlit run auditforensics_web.py
```

Or:

```bash
./start.sh          # if present
```

Open the URL Streamlit prints (usually `http://localhost:8501`).

---

## First-time setup

1. On first launch with no users, create the **administrator** account (setup code is printed in the server console, or use headless env vars — see [Environment variables](#environment-variables)).
2. Sign in. Enable **MFA** under Account and store recovery codes.
3. Create additional users and roles if needed:
   - **admin** — full control  
   - **analyst** — scans and exports  
   - **auditor** — read-only + exports  

---

## How to use

### Main tabs

| Tab | Purpose |
|-----|---------|
| **Scan** | Choose scan type, target, options; start the run |
| **Network** | Network Map workspace (devices, topology, CVEs) |
| **Agents** | Endpoint agent fleet (if enabled) |
| **Results** | Findings, charts, threat model, exports |
| **Compliance** | Framework mapping for the last scan |
| **Charts** | Severity / status visuals |
| **History** | Past scans, trends, compare current vs previous |
| **Account** | Users, MFA, audit trail |

### Typical workflow

1. Open **Scan**.
2. Select a **scan type** (e.g. Local Machine, Web Application, Network Map).
3. Enter the **target** (URL, IP range, DC, etc.).
4. Record authorization when the UI asks (especially for public targets).
5. Click **Start Scan** and wait for the progress bar.
6. Open **Results** for findings, risk score, and threat model.
7. Export CSV / JSON / SARIF / HTML or the remediation checklist as needed.

---

## Web application scans

Web scans are designed to stay **short and understandable**.

### Depth (one choice)

| Option | Intent |
|--------|--------|
| **Fast — about 1 minute** | Headers + short built-in path list; stops quickly |
| **Recommended — about 3 minutes** | Default; still capped; good for most sites |
| **Thorough — up to 10 minutes** | Larger path list / hostnames; still time-limited |

Path checking **stops on a timer** so the session does not expire mid-run. You can continue later (resume) if a run stopped early.

### Extra settings (optional)

Under **Show extra settings**:

- Signed-in checks (cookies / headers) — use only on private installs  
- Certificate / HTTPS, script endpoint extraction, cloud-metadata-style paths on the target  
- Custom path list size and max paths  
- Related hostnames (certificate logs / limited name guessing)

### Favorites and bulk URLs

- Save frequent URLs to favorites  
- Optional bulk list (capped so multi-site runs stay short)

### SecLists

If a SecLists checkout is present (`../SecLists`, `~/.auditforensics/SecLists`, or bundled `SecLists/`), larger path and DNS lists are available under advanced settings. Without SecLists, built-in high-value paths still run.

---

## Results, threat model & exports

### Audit findings

Filter by host, category, and risk. Status values include PASS, FAIL, WARNING, INFO, ERROR, SKIPPED. Checks that did not run appear under **Scan Coverage** — they are not passes.

### Threat model (separate stream)

Open FAIL/WARNING findings are mapped to playbooks with:

- Priority (P0–P3)  
- Actors, assets, attack path, impact, likelihood  
- MITRE techniques  
- **Detect**, **Hunt**, and **Triage** guidance  

This is **not** mixed into the PASS/FAIL table.

### Other outputs

- Global **risk score** (0–100) and charts  
- **Remediation checklist** (CSV)  
- **ATT&CK Navigator** layer JSON  
- Audit exports: CSV, JSON, SARIF, HTML  
- Threat model Markdown / CSV  

---

## Project layout

| Path | Role |
|------|------|
| `auditforensics_web.py` | UI, scanners, orchestration |
| `af_auth.py` | Login, roles, MFA, session idle timeout |
| `af_scope.py` | Scope, rate limits, authorization records |
| `af_web_enhance.py` | Scan profiles, TLS/JS helpers, checkpoints, executive report |
| `af_subdomains.py` | Subdomain discovery |
| `af_threatmodel.py` | Threat model + hunt/triage stream |
| `af_extras.py` | Risk score, MITRE coverage, checklist, favorites, compare |
| `af_checks.py` / `af_catalog.py` | Host checks and catalog |
| `af_netmap.py` (+ net helpers) | Network Map |
| `af_manager_api.py` / `af_agent.py` | Agents |
| `requirements.txt` | Core dependencies |
| `requirements-optional.txt` | Optional scanners / exporters |
| `SecLists/` | Optional wordlists (if bundled) |

Data and history default under `~/.auditforensics/` (SQLite history, scope, favorites, checkpoints).

---

## Optional modules

Install from `requirements-optional.txt` only what you need, for example:

- `ldap3` — Active Directory  
- `yara-python` — memory / file rules  
- `boto3` — AWS-related config checks  
- Report / document libraries for consent PDFs and richer exports  

Missing optional packages degrade gracefully; related scan types may be unavailable or show install hints.

---

## Environment variables

| Variable | Purpose |
|----------|---------|
| `AF_UI_AUTH=0` | Disable UI auth (not recommended on shared hosts) |
| `AF_UI_IDLE_MIN` | Idle timeout in minutes (default **180**) |
| `AF_ADMIN_USER` / `AF_ADMIN_PASSWORD` | Headless first admin (password rules apply) |
| `AF_SSO=1` | Enable OIDC SSO (Streamlit ≥ 1.42 + secrets) |
| `AF_SCAN_RATE_PER_HOST` | Per-host request rate limit |
| `AF_SYSLOG` | e.g. `udp://siem:514` to mirror audit events |

See `INSTALL.md` for SSO claims, step-up auth, and scope file details.

---

## Troubleshooting

**`cryptography` build fails (maturin / Rust / libLLVM)**  
Install a wheel only:

```bash
pip install --only-binary=:all: "cryptography>=42,<51"
pip install -r requirements.txt
```

**Session expired after a long scan**  
Use **Recommended** or **Fast** web depth. Path checks stop on a timer, and finished scans refresh the session clock. Raise idle time with `AF_UI_IDLE_MIN` if needed.

**Web scan still feels slow**  
Turn off Thorough, subdomains, and large custom path lists. Prefer built-in paths.

**No SecLists**  
Built-in paths still work. Place SecLists under `~/.auditforensics/SecLists` or next to the app for larger lists.

**Port already in use**  

```bash
streamlit run auditforensics_web.py --server.port 8502
```

---

## Security notes

- Only test systems you are authorized to assess.  
- External targets may require a recorded authorization in the UI.  
- Do not paste production session cookies into shared or untrusted deployments.  
- Treat findings and exports as sensitive.  
- Keep the app and dependencies updated; restrict who can reach the Streamlit port.

---

## License & disclaimer

This tool is provided for defensive security assessment and operations. You are responsible for lawful use, scope, and handling of results.

---

*AuditForensics Advanced v10.5*
