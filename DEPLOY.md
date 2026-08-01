# Deployment runbook — TECO Billing add-on (homelab)

The product is a single **Home Assistant add-on**. It runs the headless-browser
engine, pushes data into Home Assistant itself (Energy Dashboard + sensors), and
serves a sidebar dashboard. A standalone Docker option is included for non-HA hosts.

> ⚠️ **Run on your LAN.** reCAPTCHA v3 scores datacenter IPs harshly. Home Assistant
> on your home network logs in reliably; a cloud VM may get blocked.

---

## A. Install the add-on

### A1. Add the repository (recommended)
The repo root is a valid Home Assistant add-on repository (`repository.yaml` +
`teco_billing/` at the top level). In HA:
1. **Settings → Add-ons → Add-on Store → ⋮ → Repositories** → paste
   `https://github.com/jjarboe01/TECO-HA-AddOn` → **Add**.
   (Or click the "Add repository to my Home Assistant" badge in the README.)
2. The **TECO Billing** add-on appears in the store → open it → **Install** (first
   build pulls the Playwright base image, a few hundred MB, one time).

### A2. (Alternative) local add-on
Copy the add-on folder onto HA (via the **Samba** or **SSH/Advanced Terminal**
add-on) so you have `/addons/teco_billing/config.yaml`:
```bash
rsync -a teco_billing/ root@<ha-host>:/addons/teco_billing/
```
Then **Add-on Store → ⋮ → Check for updates** → install from **Local add-ons**.

### A3. Configure & start
**Configuration** tab:
- **teco_user** / **teco_pass** — your TECO portal login
- **backfill_bills** — bills to pull on first run (default 36 ≈ 3 years)
- **poll_interval_hours** — how often to refresh + push to HA (default 6)
- **session_ttl_min** — re-login interval (default 30)
- **auth_token** — only needed if you expose the optional API port (see C)

**Start**, then watch the **Log** tab: `login OK` → `fetching bill …` → on the first
cycle, `import_statistics teco:energy_consumption: ok` and `updated N TECO sensor
entities`. The first backfill takes a few minutes; later polls are incremental.

> **Changing credentials?** The add-on reads them at startup — **restart** the add-on
> after editing them on the Configuration tab.

---

## B. What shows up in Home Assistant
- **Sidebar → TECO Billing** — the dashboard (bills, service periods, kWh, cost,
  $/kWh, meter reads; charts; CSV export; per-bill re-assemble).
- **Settings → Devices & Services → Entities** — `sensor.teco_amount_due`,
  `sensor.teco_last_bill_cost`, `sensor.teco_last_bill_rate` ($/kWh),
  `sensor.teco_service_period_start` / `_end`, `sensor.teco_account_status`,
  `binary_sensor.teco_paperless`, `…_autopay`, etc.
- **Settings → Dashboards → Energy → Add consumption** — pick the **TECO Energy**
  statistic (`teco:energy_consumption`). A `teco:energy_cost` statistic is also
  imported. External statistics can take a few hours to first appear; check
  **Developer Tools → Statistics**.

> REST-state sensors repopulate on each poll, so right after an HA restart they may
> read `unavailable` until the next cycle (or an add-on restart). The Energy
> Dashboard statistics are persistent and unaffected.

---

## C. Standalone Docker (non-HA host, or HA Core via docker-compose)
Runs the dashboard + JSON API. By default there's no Supervisor to push into HA
(dashboard/API-only mode), but if your HA is a docker-compose / Core install
(no Supervisor — e.g. not HAOS/Supervised), you can still get the same
sensors + Energy Dashboard statistics as the add-on:

```bash
cd sidecar
cp .env.example .env
docker compose up -d          # or --build if you're building from source instead of the published image
docker compose logs -f        # watch for "login OK"
```

In `.env`, set `TECO_USER` / `TECO_PASS` (+ optional `SIDECAR_TOKEN` to lock the API), and
optionally the two variables below to also push into HA:

- `HA_URL` — your HA base URL, e.g. `http://homeassistant.local:8123` or `http://192.168.1.10:8123`
  (if the sidecar and HA are on the same Docker network/compose project, this can be the HA
  container's service name, e.g. `http://homeassistant:8123`)
- `HA_TOKEN` — a Long-Lived Access Token from your HA user profile
  (Profile → Security → Long-Lived Access Tokens → Create Token)

With both set, sensors and Energy Dashboard statistics are pushed exactly as they would be
from the add-on. Without them, you get the dashboard + JSON API only, and can pull data into
HA yourself via a [RESTful sensor](https://www.home-assistant.io/integrations/rest/) pointed
at `http://<host>:8089/data`.

UI at `http://<host>:8089/`; archive persists on the `teco_archive` volume — back it up.

---

## Verify
```bash
# add-on log shows: login OK, import_statistics ... ok, updated N TECO sensor entities
# in HA: Developer Tools → Statistics → search "teco"
# in HA: Developer Tools → States → filter "teco"
```

## Troubleshoot
| Symptom | Fix |
|---|---|
| `login failed (reCAPTCHA score or bad credentials)` | Run on your LAN (not a cloud IP); recheck user/pass on the Configuration tab; restart after changes. |
| Panel loads but empty | First backfill still running — watch the add-on Log, or click **Refresh from TECO**. |
| No sensors / no Energy stat | Confirm `homeassistant_api: true` is in the add-on (it is by default); check the Log for `import_statistics`/`updated … entities`; stats can lag a few hours. |
| Sensors show `unavailable` after restart | Expected for REST-state sensors until the next poll; trigger a restart or wait for the cycle. |
| First build is slow / large | Expected — the Playwright base image bundles Chromium (one-time). |

## Maintenance
- After editing the engine/parsers in `sidecar/`, re-vendor the add-on: `./sync.sh`.
- Force-refresh one bill from the panel (↻) or `POST /reassemble?invoice_id=<id>`.
  Rebuild everything: `GET /data?force=true`.
- The archive is append-only and never purged; back up the add-on's `/data`.
