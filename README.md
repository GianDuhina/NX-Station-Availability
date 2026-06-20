# NX Station Availability

Automated daily / weekly / monthly availability reports for Skylark NX stations across the Orion production clusters (ap, eu, na).

**Metric**: `orion_station_healthiness{stage="prod-all-freq", type="sel"}`

**Status mapping** (mirrors the Grafana Station Healthiness panel):
- 🟢 **Healthy** — value ≥ 0.80
- 🟠 **Degraded** — 0 < value < 0.80
- 🔴 **Unhealthy** — value = 0

Reports include All Regions overview + per-region drilldowns (AP / EU / NA), Swift Navigation light/dark themed, hard-defaulted to dark.

**Live reports**: see the `latest_nx_stationavailability_*.html` files at this repo's root.
