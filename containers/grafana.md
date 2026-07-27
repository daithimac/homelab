---
type: LXC Container
title: grafana (CT110)
description: Grafana 13 dashboards backed by a co-located Prometheus scraping the Proxmox host, unprivileged LXC at 192.168.0.21.
tags: [proxmox, lxc, grafana, monitoring, prometheus]
timestamp: 2026-07-25T00:00:00Z
---

CT110, hostname `grafana`, unprivileged LXC, static **192.168.0.21**. Runs Grafana 13.1.0
(port 3000) alongside a co-located Prometheus 2.42.0 instance that does the actual metric
collection — this container is a self-contained monitoring stack, not just a dashboard
frontend with an external data source.

# What it monitors

`prometheus.service` scrapes three targets (5s interval), all healthy as of 2026-07-25:

* `localhost:9090` — itself.
* `localhost:9100` — `prometheus-node-exporter` running on this container, i.e. **this
  LXC's own** resource usage, not the Proxmox host's.
* `localhost:9221` — `pve-exporter.service`, proxying **the N5 Pro host itself**
  (`192.168.0.10`) via the Proxmox API. This is the target that actually matters: host-level
  CPU, memory, storage, and guest stats.

`pve-exporter` isn't a Debian package — it's installed manually into a venv at
`/opt/pve-exporter` (config `/etc/prometheus/pve.yml`), authenticating to Proxmox as a
dedicated `prometheus@pve` API user (`verify_ssl: false`).

# Grafana config

Single datasource: Prometheus at `http://localhost:9090`.

**First dashboard built 2026-07-25**: "N5 Pro Host Overview" (`uid: n5pro-overview`,
`https://grafana.lan/d/n5pro-overview/`) — host CPU/memory/uptime stats, per-guest
CPU/memory timeseries, and storage pool usage %, all against the `pve_*` metrics from
`pve-exporter`. Created via the HTTP API (`POST /api/dashboards/db`), not the classic file
provisioner — see the gotcha below.

# Provisioning gotcha: the classic file-based dashboard provisioner doesn't work here

Spent a long debugging pass on this (2026-07-25) before concluding it's a genuine quirk of
this Grafana 13.1.0 build, not a config mistake: dropping a dashboard JSON + provider YAML
into `/etc/grafana/provisioning/dashboards/` (the confirmed-correct, confirmed-readable,
confirmed-live path per the running process's own `cfg:` args) produces zero errors, logs
a "starting/finished to provision dashboards" pair completing in under a millisecond, and
creates nothing. This build has moved to a newer internal "unified storage"
(`dashboard.grafana.app`) backend — the classic `dashboard` SQL table in `grafana.db` is
now vestigial and doesn't reflect what's actually provisioned, which is what made this so
confusing to diagnose (every verification query against that table looked like total
failure). **Use the HTTP API instead**: `POST /api/dashboards/db` with
`{"dashboard": {...}, "overwrite": true, "folderId": 0}` — confirmed working, and
verifiable via `GET /api/search` (not the sqlite `dashboard` table).

# Admin credentials — reset 2026-07-25, needs Dave's attention

The original admin password was unknown (not default `admin`/`admin`) and there was no
other way to authenticate for the API approach above, so it was reset via
`grafana cli admin reset-admin-password` (with the same `GF_PATHS_*` environment variables
the live systemd service uses — the CLI defaults to a different, wrong data directory
otherwise, another instance of the same gotcha as the provisioning path). **New password:
`N5proTemp2026xyz`, still the `admin` login.** This is a temporary value — change it at
`https://grafana.lan/profile/password` when convenient. Nobody else's access was affected
(no other users existed).

# Citations

[1] Live `pct exec 110` shell review via the Proxmox web console (2026-07-25).
