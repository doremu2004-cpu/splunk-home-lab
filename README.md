# 🛡️ Splunk Home Lab — Log Monitoring & Alert Detection

A self-contained home lab that stands up Splunk, feeds it realistic sample logs (SSH auth, firewall/network, and web access traffic), and builds SPL searches, a security dashboard, and alerting rules to detect suspicious activity — brute-force attempts, port scans, SQL injection, off-hours logins, and privilege escalation.

Built to demonstrate practical SOC-analyst / detection-engineering skills: log onboarding, field extraction, correlation searches, dashboarding, and alert design.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [What's Included](#whats-included)
- [Quick Start](#quick-start)
- [Sample Data](#sample-data)
- [Detection Searches](#detection-searches)
- [Dashboard](#dashboard)
- [Alerts](#alerts)
- [Skills Demonstrated](#skills-demonstrated)
- [Repo Structure](#repo-structure)
- [Roadmap / Ideas for Extension](#roadmap--ideas-for-extension)
- [License](#license)

---

## Overview

This project simulates a small enterprise environment being monitored by Splunk. Instead of relying on live infrastructure, it uses **synthetic but realistic log data** so anyone can clone the repo, spin up Splunk locally (via Docker), ingest the sample data, and immediately start practicing detection engineering.

The lab covers three common log sources:

| Log Source | Format | Simulates |
|---|---|---|
| `linux_auth.log` | Linux `/var/log/auth.log` (syslog) | SSH logins, sudo usage, brute-force attempts |
| `firewall.log` | iptables/netfilter style | Port scans, blocked connections, traffic spikes |
| `web_access.log` | Apache combined log format | SQLi attempts, scanner traffic, abnormal status codes |

Each log file was generated to contain **both benign "normal" activity and embedded attack patterns**, so the searches in this repo have real signal to detect.

## Architecture

```
                     ┌────────────────────────┐
                     │   Sample Log Sources    │
                     │  (data/*.log files)     │
                     └───────────┬─────────────┘
                                 │  manual upload / monitor input
                                 ▼
                     ┌────────────────────────┐
                     │     Splunk Indexer      │
                     │  (Docker container)     │
                     │  index=security_lab      │
                     └───────────┬─────────────┘
                                 │  props.conf / transforms.conf
                                 │  (field extraction, sourcetypes)
                                 ▼
                     ┌────────────────────────┐
                     │   SPL Detection Searches │
                     │  searches/*.spl          │
                     └───────────┬─────────────┘
                       ┌─────────┴──────────┐
                       ▼                    ▼
             ┌──────────────────┐  ┌──────────────────┐
             │ Security Dashboard│  │  Scheduled Alerts │
             │ dashboards/*.xml  │  │  alerts/*.md      │
             └──────────────────┘  └──────────────────┘
```

## What's Included

- ✅ `docker-compose.yml` — spins up a single-instance Splunk Enterprise (free dev license) in one command
- ✅ Three sample log datasets with embedded attack scenarios
- ✅ `props.conf` / `transforms.conf` for correct sourcetyping and field extraction
- ✅ 7 documented SPL searches for common suspicious-activity patterns
- ✅ A ready-to-import Splunk Simple XML dashboard (`security_monitoring_dashboard.xml`)
- ✅ Alert definitions (trigger conditions + suggested actions) for each detection
- ✅ Step-by-step setup guide and architecture notes

## Quick Start

### Prerequisites
- Docker & Docker Compose installed
- 4GB+ RAM free
- A modern browser

### 1. Clone the repo
```bash
git clone https://github.com/<your-username>/splunk-home-lab.git
cd splunk-home-lab
```

### 2. Start Splunk
```bash
docker-compose up -d
```
Splunk Web will be available at **http://localhost:8000** (default login: `admin` / `changeme123!` — set in `docker-compose.yml`, change it before exposing this anywhere).

### 3. Create the lab index
In Splunk Web: **Settings → Indexes → New Index** → name it `security_lab`.

### 4. Upload the sample data
**Settings → Add Data → Upload**, and add each file in `data/` one at a time, applying the matching sourcetype:

| File | Sourcetype to assign |
|---|---|
| `data/linux_auth.log` | `linux_secure` |
| `data/firewall.log` | `iptables_syslog` |
| `data/web_access.log` | `access_combined` |

(Full field-by-field walkthrough in [`docs/SETUP.md`](docs/SETUP.md).)

### 5. Run the detection searches
Copy any query from `searches/` into Splunk's search bar, or save them as reports.

### 6. Import the dashboard
**Settings → User Interface → Views → Import XML** and paste in `dashboards/security_monitoring_dashboard.xml`.

### 7. Configure alerts
Follow [`alerts/alert_configs.md`](alerts/alert_configs.md) to turn any search into a scheduled, actionable alert.

Full details for every step: see [`docs/SETUP.md`](docs/SETUP.md).

## Sample Data

All log files live in `data/` and span a **24-hour window** with embedded incidents:

- **Brute-force SSH attack** from `203.0.113.45` against user `admin` (14 failed attempts in 3 minutes, followed by one success)
- **Port scan** from `198.51.100.23` sweeping ports 20–1024 on the firewall
- **SQL injection probing** against `/login.php` and `/search.php` on the web server
- Normal background traffic (legitimate logins, regular web browsing, routine firewall ACCEPTs) to keep the data realistic and give searches something to filter out

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full incident timeline.

## Detection Searches

| File | Detects |
|---|---|
| `searches/01_failed_login_bruteforce.spl` | ≥5 failed SSH logins from one source in a 5-minute window |
| `searches/02_successful_login_after_failures.spl` | A successful login immediately following a burst of failures (possible compromise) |
| `searches/03_port_scan_detection.spl` | A single source hitting many distinct destination ports in a short window |
| `searches/04_off_hours_logins.spl` | Successful logins outside of standard business hours |
| `searches/05_sql_injection_attempts.spl` | Web requests containing common SQLi signatures |
| `searches/06_privilege_escalation.spl` | New sudo usage or user/group creation by non-standard accounts |
| `searches/07_traffic_spike_top_talkers.spl` | Top source IPs by connection/request volume, for spotting anomalies |

Each `.spl` file has inline comments explaining the logic, so they double as an SPL learning reference.

## Dashboard

`dashboards/security_monitoring_dashboard.xml` provides a single-pane-of-glass view with:

- Failed vs. successful login trend (time chart)
- Top attacking source IPs (table)
- Port scan activity (table)
- SQL injection attempts over time (time chart)
- Off-hours login events (table)
- Key metric panels: total failed logins, total blocked connections, total SQLi attempts (last 24h)

## Alerts

`alerts/alert_configs.md` documents how to convert each search above into a real-time or scheduled Splunk alert, including:
- Recommended cron schedule / trigger condition
- Throttling settings (to avoid alert fatigue)
- Example alert actions (email, webhook, add to triggered-alerts index)

## Skills Demonstrated

- Splunk data onboarding (`props.conf`, `transforms.conf`, sourcetyping)
- SPL: `stats`, `eventstats`, `streamstats`, `transaction`, `where`, subsearches, `timechart`
- Detection engineering / threat-hunting logic (brute force, scanning, injection, privilege escalation)
- Dashboard design (Simple XML)
- Alert design and tuning (thresholds, throttling, false-positive reduction)
- Security log analysis across host, network, and application layers

## Repo Structure

```
splunk-home-lab/
├── README.md
├── docker-compose.yml
├── config/
│   ├── props.conf
│   └── transforms.conf
├── data/
│   ├── linux_auth.log
│   ├── firewall.log
│   └── web_access.log
├── searches/
│   ├── 01_failed_login_bruteforce.spl
│   ├── 02_successful_login_after_failures.spl
│   ├── 03_port_scan_detection.spl
│   ├── 04_off_hours_logins.spl
│   ├── 05_sql_injection_attempts.spl
│   ├── 06_privilege_escalation.spl
│   └── 07_traffic_spike_top_talkers.spl
├── dashboards/
│   └── security_monitoring_dashboard.xml
├── alerts/
│   └── alert_configs.md
└── docs/
    ├── SETUP.md
    └── ARCHITECTURE.md
```

## Roadmap / Ideas for Extension

- [ ] Add Sysmon/Windows Event Log sample data for endpoint detections
- [ ] Add a lookup table for known-malicious IP enrichment
- [ ] Build a correlation search that chains recon (port scan) → access (brute force) → action (privilege escalation) into a single "attack chain" alert
- [ ] Add Splunk Enterprise Security-style risk scoring
- [ ] Automate data ingestion via `inputs.conf` monitor stanzas instead of manual upload

## License

MIT — see [LICENSE](LICENSE). Sample log data is entirely synthetic; no real hosts, users, or IP ownership is implied.
