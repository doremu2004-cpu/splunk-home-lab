# Architecture & Incident Timeline

## Lab Topology (Simulated)

The sample logs represent a small, simplified environment:

```
                 Internet
                     │
             ┌───────┴────────┐
             │   fw01 (perimeter firewall)  │
             └───────┬────────┘
                     │
        ┌────────────┼────────────┐
        │                         │
┌───────┴────────┐       ┌────────┴────────┐
│  webserver01    │       │  internal hosts  │
│  (SSH + web app)│       │  (10.0.0.0/24)   │
└─────────────────┘       └─────────────────┘
```

- `webserver01` generates the SSH/auth events (`linux_auth.log`) and the web application access events (`web_access.log`)
- `fw01` generates perimeter firewall events (`firewall.log`) for all traffic reaching the environment
- Internal users (`jsmith`, `adeploy`, `mrodriguez`, `kpatel`) represent legitimate day-to-day activity from the `10.0.0.0/24` range

All three logs are ingested into a single Splunk index (`security_lab`) with distinct sourcetypes, mirroring how a real SIEM correlates multiple log sources against a shared timeline.

## Embedded Incident Timeline (Jul 26, 22:40 - 23:00 UTC)

The sample data contains one connected, multi-stage incident, designed so the detections in this repo can be chained together the way a real investigation would:

| Time (UTC) | Log Source | Event |
|---|---|---|
| 22:40:01 - 22:40:15 | Firewall | `198.51.100.23` sweeps 15 distinct ports against `10.0.0.7` — **reconnaissance / port scan** |
| 22:58:01 - 22:59:19 | SSH Auth | `203.0.113.45` sends 14 failed password attempts for user `admin` — **brute-force attempt** |
| 22:59:30 | SSH Auth + Firewall | `203.0.113.45` succeeds on the 15th attempt — **initial access achieved** |
| 22:59:52 - 23:00:10 | Sudo | The now-authenticated `admin` session creates a new user `svc_backup` and adds it to the `sudo` group — **privilege escalation / persistence** |
| 23:00:44 | SSH Auth | `svc_backup` logs in directly — **use of newly created backdoor account** |
| 22:59:40 - 23:00:20 | Web Access | `203.0.113.45` (same actor) browses `/admin/` and exports user data via `/admin/users?export=1` — **data access following compromise** |

This is intentionally realistic: a scan, followed by a credential attack, followed by account creation for persistence, followed by data access — the same shape as many real-world intrusions. Every stage is independently detectable by one of the searches in `searches/`, and stitching them together (by `src_ip` and time proximity) is suggested as an extension in the README's Roadmap section.

A second, separate incident is the SQL injection probing from `185.220.101.7` against `/search.php` and `/login.php` around 16:02 UTC — unrelated to the SSH incident above, included to exercise the web-log detection independently.

## Design Decisions

- **Synthetic data over live capture:** Guarantees the lab is reproducible and safe to publish publicly (no real hostnames, credentials, or IP ownership).
- **Three log sources, not one:** Demonstrates cross-source correlation, which is the core value proposition of a SIEM over grepping individual log files.
- **Comments embedded directly in `.spl` files:** Makes each search self-documenting when read directly from GitHub, without requiring the reader to also open a separate explanation doc.
- **Simple XML dashboard instead of Splunk Dashboard Studio:** Simple XML is plain-text, diffable, and reviewable in a pull request — better suited to a version-controlled repo than Studio's JSON-heavy format, though either would work.
