# Setup Guide

Full walkthrough for standing up this lab from a clean clone.

## 1. Prerequisites

- Docker Desktop or Docker Engine + Docker Compose plugin
- ~4GB of free RAM for the Splunk container
- A browser

## 2. Start Splunk

```bash
git clone https://github.com/<your-username>/splunk-home-lab.git
cd splunk-home-lab
docker-compose up -d
```

Wait 1-2 minutes for Splunk to finish initializing, then check the logs:

```bash
docker-compose logs -f splunk
```

Look for a line indicating Splunkd has started. Then open **http://localhost:8000**.

Login with:
- Username: `admin`
- Password: `changeme123!` (as set in `docker-compose.yml` — change this before using the lab beyond your own machine)

## 3. Create the lab index

1. Go to **Settings → Indexes**
2. Click **New Index**
3. Name: `security_lab`
4. Leave other settings at default, click **Save**

## 4. Onboard the sample data

The `props.conf` and `transforms.conf` in `config/` are already mounted into the container by `docker-compose.yml`, so the sourcetype definitions are available. Restart the container once if you started it before reading this (`docker-compose restart splunk`) to make sure they're loaded.

For each file below, go to **Settings → Add Data → Upload**, select the file, and set the fields as shown:

### `data/linux_auth.log`
- Sourcetype: `linux_secure` (should appear under "Operating System" category if props.conf loaded correctly — otherwise select "Manual" and type it in)
- Index: `security_lab`
- Host: `webserver01` (or leave as the filename; the extraction pulls the real host from the event text anyway)

### `data/firewall.log`
- Sourcetype: `iptables_syslog`
- Index: `security_lab`

### `data/web_access.log`
- Sourcetype: `access_combined`
- Index: `security_lab`

After each upload, click **Review** to confirm the timestamp and line-breaking look correct (each event should be its own line, with a timestamp parsed from Jul 2026), then **Submit**.

## 5. Verify ingestion

Run this in the Search bar:

```spl
index=security_lab | stats count by sourcetype
```

You should see roughly:
- `linux_secure` — 41 events
- `iptables_syslog` — 29 events
- `access_combined` — 22 events

If a sourcetype shows 0 or the wrong count, re-check the sourcetype assignment during upload, or manually re-upload with **Settings → Add Data → Upload → set sourcetype = your_value**.

> **Note on timestamps:** the sample data uses dates around July 26-27, 2026. When running searches, set your time range to **"All time"** first, or explicitly to a window covering those dates, since Splunk's default "Last 24 hours" will be relative to *today's* real date, not the log dates.

## 6. Run the detection searches

Open each file in `searches/`, copy the SPL portion (everything except the `` ` ``comment lines at the top, which are optional inline documentation and will run fine as comments in Splunk too), paste into the Search bar, and set your time range to **All time**.

Optionally save each as a **Report** (Save As → Report) so they're reusable and appear in **Reports** for later reference.

## 7. Import the dashboard

1. Go to **Settings → User Interface → Views**
2. Click **New View**, choose type "Dashboard", give it a title, then click **Create**
3. In the dashboard editor, switch to **Source** view (the `</>` icon)
4. Replace the contents with `dashboards/security_monitoring_dashboard.xml`
5. Click **Save**

Set the dashboard's time picker to cover July 26-27, 2026 (or "All time") to see the sample data populate.

## 8. Configure alerts

Follow [`../alerts/alert_configs.md`](../alerts/alert_configs.md) — for each search, run it, click **Save As → Alert**, and apply the schedule/trigger/throttling settings documented there.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Fields like `src_ip` aren't extracted | `props.conf`/`transforms.conf` not loaded | Confirm the volume mounts in `docker-compose.yml`, restart the container |
| Events all show one giant blob instead of separate lines | Wrong sourcetype selected during upload | Re-upload and manually select the correct sourcetype from the dropdown |
| No results when running searches | Time range mismatch | Set search time range to "All time" — sample data is dated Jul 2026 |
| Dashboard panels show "No results found" | Same as above, or dashboard time picker default is too narrow | Adjust the time picker at the top of the dashboard |
