# NX Station Availability

Automated daily / weekly / monthly availability reports for **Skylark NX**
stations across the Orion production clusters (`ap`, `eu`, `na`). Pulled
from Grafana/Thanos, rendered to HTML + PNG + CSV, published to GitHub Pages
at <https://gianduhina.github.io/NX-Station-Availability/>, and posted to
Slack.

Sibling project to [`station-availability`](https://github.com/GianDuhina/station-availability-reports)
(DFMG NetCloud router availability). Same brand, same patterns, different
metric scope.

---

## What this report covers

- **2,984 Skylark NX stations** across three Orion production clusters
- Source metric: **`orion_station_healthiness`**
- Filter: `stage="prod-all-freq"`, `type="sel"`, `cluster=~"ap|eu|na"`
- Aggregation: `avg by (station_name)` — averages across the two container
  replicas (`sel-1` + `sel-2`) per station
- Identical query + value mapping as the Grafana
  **Station Healthiness** panel (id=32) on the NX Monitoring System dashboard,
  so the numbers in this report match the dashboard 1:1

---

## Status mapping (from the Grafana panel value mapping)

| Value range | Status | Color |
|---|---|---|
| `0.80 ≤ value ≤ 1.00` | **Healthy** | 🟢 green |
| `0 < value < 0.80` | **Degraded** | 🟠 orange |
| `value = 0` | **Unhealthy** | 🔴 red |

A station is classified by its **avg healthiness over the report window**
(not just its current state).

### Container redundancy semantics

Most stations are monitored by two selection pods running side-by-side
(`sel-1` and `sel-2`). Both report independently. The `avg by (station_name)`
aggregation collapses their views per station, which means:

- Both containers say healthy → avg=1.0 → **Healthy**
- One container disagrees (sees the station unhealthy) → avg=0.5 → **Degraded**
- Both containers say unhealthy → avg=0.0 → **Unhealthy**

So "Degraded" is often a signal that **one redundancy replica has lost sight
of the station** — worth investigating separately from a full outage.

| Containers per station | # stations |
|---|---|
| Both `sel-1` and `sel-2` | 2,852 |
| Only one container | 133 |

---

## Region breakdown (May 2026 snapshot)

| Region | Stations | Avg healthiness | Healthy | Degraded | Unhealthy |
|---|---|---|---|---|---|
| **All Regions** | 2,984 | 92.38% | 2,698 | 165 | 121 |
| Asia Pacific | 143 | 94.27% | 135 | 5 | 3 |
| Europe | 1,941 | 93.32% | 1,774 | 99 | 68 |
| North America | 900 | 90.06% | 789 | 61 | 50 |

Regions map 1:1 to Orion clusters:
- AP cluster → **Asia Pacific**
- EU cluster → **Europe**
- NA cluster → **North America**

The `eu-dev-2` cluster (preprod) and `na2` cluster (prod-test) are
intentionally excluded.

---

## Scripts

```
NXstationavailability/
├── scripts/
│   ├── nx_station_availability.py    ← production script
│   └── station_availability.py        ← shared plumbing (fetch / push / Slack)
├── assets/
│   └── swift_nav_logo.png             ← embedded base64 in every report
├── cache/                             ← (planned — hybrid layer for fast monthly)
└── README.md                          ← this file
```

The shared `station_availability.py` plumbing is a copy of the file from
the sibling project, with `GITHUB_REPO` repointed to
`GianDuhina/NX-Station-Availability` so all `_gh_put` calls land here.

---

## CLI

Default behaviour: **publish to GitHub Pages + post to Slack** on every run.

```bash
# Default = previous full calendar month
python3 scripts/nx_station_availability.py --mode monthly

# Specific period
python3 scripts/nx_station_availability.py --mode daily   --date 2026-06-19
python3 scripts/nx_station_availability.py --mode weekly  --date 2026-W21
python3 scripts/nx_station_availability.py --mode monthly --date 2026-05

# Opt-outs
python3 scripts/nx_station_availability.py --mode daily --dry-run    # skip Pages + Slack
python3 scripts/nx_station_availability.py --mode daily --no-slack   # push to Pages, skip Slack
```

| Flag | Effect |
|---|---|
| `--mode {daily,weekly,monthly}` | Report mode (default `monthly`) |
| `--date SPEC` | Anchor date — Daily `YYYY-MM-DD` · Weekly `YYYY-MM-DD` or `YYYY-Www` · Monthly `YYYY-MM` |
| `--dry-run` | Skip both GitHub Pages push AND Slack post |
| `--no-slack` | Push to Pages but skip Slack |

---

## Report layout

### All Regions - Overview (default tab)

Executive summary view. Strips per-station detail:
- 5 KPI tiles: **Avg healthiness · Healthy · Degraded · Unhealthy · Data window**
- Compact 3-row regional rollup table (Asia Pacific / Europe / North America with their respective counts)

### Per-region tabs (Asia Pacific / Europe / North America)

Full per-station detail. Same KPI tiles + per-station table:
- Station name
- Region label
- Status pill (Healthy / Degraded / Unhealthy)
- Avg healthiness %
- Healthy time · Degraded time · Unhealthy time (per-status duration breakdown)

Sorted worst-first (unhealthy → degraded → healthy).

### Theme

- Swift Navigation light/dark branded theme
- **Hard-defaulted to dark** every page load — `localStorage` not used so partners see consistent default
- Sun/moon toggle works for the current session
- Logo embedded as base64 PNG so reports are self-contained

---

## Output locations

Each run produces 3 local files in the project root:

```
nx_stationavailability_<period>.html   ← interactive (theme toggle, region tabs)
nx_stationavailability_<period>.png    ← headless-Chrome screenshot (dark mode)
nx_stationavailability_<period>.csv    ← all 2,984 stations, full detail
```

Then pushes to <https://gianduhina.github.io/NX-Station-Availability/> in
three places:

| Path | Purpose |
|---|---|
| `nx_stationavailability_<period>.<ext>` | Canonical dated archive |
| `latest_nx_stationavailability_<mode>.<ext>` | Convenience alias (overwritten per run) |
| `snapshots/nx_stationavailability_<period>_<utcstamp>.<ext>` | Unique-per-publish (Slack target — cache-bust) |

### CSV schema

```
station_name, region, cluster, status,
avg_healthiness, avg_healthiness_pct,
healthy_seconds, degraded_seconds, unhealthy_seconds, missing_seconds
```

---

## Architecture decisions

- **Active-cluster tiebreak for roster assignment**: when a station has data
  in multiple clusters with equal value, prefer `eu > na > ap`. (Doesn't
  affect availability math — only which region the station is filed under.)
- **Cluster→Region mapping**: 1:1 — no country-code derivation needed because
  Orion already partitions geographically by cluster.
- **`avg` aggregation across containers** (vs `max` or `min`): chosen to
  match the Grafana panel exactly. A station with one bad replica lands in
  Degraded — which is the right signal.

---

## Project journey

| Phase | Outcome |
|---|---|
| **Phase 1 — Scaffold + first publish** (2026-06-21) | Folder structure created. Shared `station_availability.py` plumbing copied locally. Swift logo asset copied. `nx_station_availability.py` written with full feature parity to the DFMG regional report (Swift-branded dark/light theme, region tabs, sentence-case headers). |
| **Grafana panel value mapping** (2026-06-21) | Aligned status classification to the Grafana Station Healthiness panel exactly: Healthy ≥ 0.80, Degraded 0–0.80, Unhealthy = 0. Switched from `max by (station_name)` to `avg by (station_name)` + added `type="sel"` filter. |
| **Separate GitHub repo** (2026-06-21) | Created `GianDuhina/NX-Station-Availability` (public). GitHub Pages enabled at <https://gianduhina.github.io/NX-Station-Availability/>. Repo is isolated from `station-availability-reports` (DFMG/regions). |
| **May 2026 monthly published** (2026-06-21) | 31-day window, autotuned step (300s), 2,984 stations, 92.38% avg healthiness. NA had the most degraded fleet (10.0% unhealthy + degraded combined). |
| **Light/dark text contrast fix** (2026-06-21) | KPI denominator (e.g. " / 2984") was invisible in dark mode due to inline navy color. Replaced with `.kpi-denom` class + dark-mode override (`#b8c2d4`). |
| **All Regions = executive overview** (2026-06-21) | Stripped the 2,984-row per-station table from the All Regions tab. Replaced with a compact 3-row regional rollup table (AP / EU / NA with per-status counts). HTML size dropped 1.5 MB → 800 KB. Per-region tabs still show full detail. |

---

## Backlog (Phase 2+)

- Hybrid per-region cache layer (mirror station-availability's pattern; `--rebuild-cache` flag for backfill)
- BETA twin script (`BETA_TESTING_nx.py`)
- T-1 auto-regen (if scheduled via launchd)
- Publish queue + morning drain (mirrors station-availability's GitHub timeout resilience)
- launchd plists for daily/weekly/monthly schedules
- Sub-panels for `orion_station_healthiness_availability`, `_constellation_presence`, `_calculated_location` (the Grafana dashboard's "Station Healthiness Metrics" panel)
- Stations with bad calculated location detection (the dashboard's diagnostic panel)
- Confluence import file (HTML + Word + ZIP, as we did for the DFMG project)

---

## Source metric notes

The Grafana dashboard "NX Monitoring System" contains multiple panels around
station healthiness — this report only mirrors the primary `Station Healthiness`
panel (id=32). Other panels in the dashboard cover:

- `orion_station_healthiness_availability` — availability subcomponent
- `orion_station_healthiness_constellation_presence` — GNSS constellation tracking
- `orion_station_healthiness_calculated_location` — location quality
- `orion_station_selection_baseline` — rover↔station baseline distance
- `orion_calculated_location_horizontal_error` / `_vertical_error` / `_estimated_accuracy` — location quality detail

These are Phase 2+ targets.

---

_Last reviewed: 2026-06-21 · Swift Navigation branded · light/dark themes (dark default) · separate GitHub repo · Grafana panel value-mapping locked in_
