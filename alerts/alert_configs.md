# Alert Configurations

This document turns each search in `searches/` into a real Splunk alert. For each one: open the search, run it, click **Save As → Alert**, and apply the settings below.

---

## 1. SSH Brute-Force Attempt

- **Based on search:** `01_failed_login_bruteforce.spl`
- **Type:** Scheduled
- **Schedule:** Run every 5 minutes, over the last 5 minutes
- **Trigger condition:** Number of results > 0
- **Throttling:** Suppress further alerts for 30 minutes, grouped by `src_ip` (prevents repeat-alerting on the same ongoing attack)
- **Severity:** High
- **Suggested actions:**
  - Send email to SOC distribution list
  - Add triggered `src_ip` to a "watchlist" lookup for correlation in other searches
  - (Optional, if integrated) fire a webhook to a SOAR platform to auto-block the IP at the firewall

## 2. Successful Login After Failures (Possible Compromise)

- **Based on search:** `02_successful_login_after_failures.spl`
- **Type:** Scheduled
- **Schedule:** Run every 10 minutes, over the last 15 minutes
- **Trigger condition:** Number of results > 0
- **Throttling:** Suppress for 60 minutes, grouped by `src_ip` + `user`
- **Severity:** Critical — this is the strongest single indicator in this lab that an account was actually compromised, not just probed
- **Suggested actions:**
  - Immediate email/SMS to on-call analyst
  - Auto-disable the affected account pending investigation (if integrated with IAM)

## 3. Port Scan Detected

- **Based on search:** `03_port_scan_detection.spl`
- **Type:** Scheduled
- **Schedule:** Run every 5 minutes, over the last 5 minutes
- **Trigger condition:** Number of results > 0
- **Throttling:** Suppress for 30 minutes, grouped by `src_ip`
- **Severity:** Medium (recon activity — worth tracking, not always actionable alone)
- **Suggested actions:**
  - Log to a "reconnaissance activity" summary index for later correlation
  - Escalate to High if the same `src_ip` also appears in Alert #1 or #2 within the same hour

## 4. Off-Hours Login

- **Based on search:** `04_off_hours_logins.spl`
- **Type:** Scheduled
- **Schedule:** Run every 30 minutes, over the last 30 minutes
- **Trigger condition:** Number of results > 0
- **Throttling:** Suppress for 4 hours per `user`
- **Severity:** Low/Medium — highly dependent on your organization; tune the business-hours window in the search first
- **Suggested actions:**
  - Email summary to team lead for review the next morning (non-urgent)
  - Exclude known on-call/night-shift accounts via a lookup to cut noise

## 5. SQL Injection Attempt

- **Based on search:** `05_sql_injection_attempts.spl`
- **Type:** Scheduled
- **Schedule:** Run every 5 minutes, over the last 5 minutes
- **Trigger condition:** Number of results > 0
- **Throttling:** Suppress for 30 minutes, grouped by `src_ip`
- **Severity:** High
- **Suggested actions:**
  - Email SOC + web application owner
  - Cross-check `response_codes`: a 500 response suggests the payload may have reached the database layer

## 6. Privilege Escalation / New Admin Account

- **Based on search:** `06_privilege_escalation.spl`
- **Type:** Scheduled
- **Schedule:** Run every 5 minutes, over the last 10 minutes
- **Trigger condition:** Number of results > 0
- **Throttling:** Do not throttle — every privilege escalation event should be individually reviewed
- **Severity:** Critical
- **Suggested actions:**
  - Immediate page to on-call security engineer
  - Auto-open a ticket in your incident-tracking system with the full command and actor included

## 7. Traffic Spike / Top Talkers

- **Based on search:** `07_traffic_spike_top_talkers.spl`
- **Type:** Not typically alerted directly — used as a triage/hunting search
- **Suggested use:** Pin to the dashboard and review manually, or schedule a daily digest email rather than a real-time alert, since "top talkers" is inherently a ranking/summary view rather than a binary trigger condition

---

## General Alert Hygiene Tips

- **Start every new alert with generous throttling.** It's much easier to tighten a noisy alert than to win back trust after an analyst gets paged 40 times for the same brute-force attempt.
- **Group throttling by the entity that represents "the same incident"** (usually `src_ip`, sometimes `src_ip + user`), not by the raw search itself.
- **Tier your severities** so downstream paging/ticketing systems can route Critical alerts differently from Low ones.
- **Review fired alerts weekly** in the early days of a new detection to tune thresholds — the values in this lab (5 failed logins, 10 distinct ports, etc.) are reasonable starting points, not universal truths.
