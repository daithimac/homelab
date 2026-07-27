---
type: Architecture
title: DNS via AdGuard Home
description: Tailnet-wide DNS design on CT109 — upstream config, local-name rewrites, and why it replaced NextDNS.
tags: [dns, adguard, tailscale, architecture]
timestamp: 2026-07-25T00:00:00Z
---

[AdGuard Home (CT109, 192.168.0.20)](/containers/adguard.md) is the resolver for all
**personal/tailnet devices**, delivered via [Tailscale (CT105)](/containers/tailscale.md):
admin console → DNS → global nameserver `192.168.0.20` with **Override DNS servers** ON
and **Use with exit node** ON (otherwise exit-node use silently bypasses AdGuard). This
replaced the earlier NextDNS-via-Tailscale setup (Jul 2026).

# Deliberate design, chosen over Virgin hub modem-mode

* The Virgin hub stays in router mode; it won't let you change DHCP-issued DNS anyway.
* **Work machines (Dave's and his wife's) are not on the tailnet and never see AdGuard** —
  interference is architecturally impossible, not just avoided.
* The TV / non-tailnet devices get Virgin's DNS and their ads. Intentional; Dave doesn't
  care.
* Wife's personal devices join via a tailnet user invite (free plan: 3 users / 100
  devices).

# AdGuard config

Web UI at `http://192.168.0.20`, port 80 (3000 is only first-run setup):

* **Upstreams**: `https://dns.cloudflare.com/dns-query` +
  `https://dns.google/dns-query`, **Parallel requests** mode. Quad9
  (`dns10.quad9.net`) was removed after it caused a household-wide outage as the sole
  load-balanced upstream (timeouts + `unexpected EOF`).
* **Fallbacks**: `1.1.1.1`, `9.9.9.10`. **Bootstrap**: `9.9.9.10`, `1.1.1.1`.
* **Local names** via Filters → DNS rewrites, `.lan` TLD (never `.local` — mDNS owns it).
  Names resolve tailnet-wide, so `jellyfin.lan` works remotely too. Most of them now go
  through the [Caddy reverse proxy](/playbooks/reverse-proxy-caddy.md) on docker-stack,
  which handles HTTPS and the port for you — plain `https://jellyfin.lan` just works. A few
  domains (`nas.lan`, `tailscale.lan`, `opencode.lan`) aren't proxied because there's no web
  UI behind them; those still need whatever port the underlying service actually listens on
  if one exists at all.

  **7 of the 12 rewrites were broken, fixed 2026-07-25.** Found by reading
  `/opt/AdGuardHome/AdGuardHome.yaml` directly (`pct exec 109`) while chasing an unrelated
  DNS check — they'd all been miscopied to answer `192.168.0.14` (docker-stack's IP),
  evidently a copy-paste error while adding entries (`n8n.docker.lan` legitimately
  answers `.14` since n8n really does run on docker-stack; whatever was added right after
  it seems to have kept that same answer value instead of being updated per domain).

  | Domain            | Answer | |
  |--------------------|-------------------|----------------|
  | `nas.lan`          | 192.168.0.11      | was already correct |
  | `ollama.lan`       | 192.168.0.13      | was already correct |
  | `jellyfin.lan`     | 192.168.0.12      | fixed, was `.14` |
  | `tailscale.lan`    | 192.168.0.16      | was already correct |
  | `openwebui.lan`    | 192.168.0.17      | fixed, was `.14` |
  | `opencode.lan`     | 192.168.0.18      | fixed, was `.14` |
  | `sabnzbd.lan`      | 192.168.0.19      | fixed, was `.14` |
  | `adguard.lan`      | 192.168.0.20      | fixed, was `.14` |
  | `proxmox.lan`      | 192.168.0.10      | fixed, was `.14` — host's own `vmbr0` IP, previously undocumented |
  | `grafana.lan`      | 192.168.0.21      | fixed, was `.14` |
  | `n8n.docker.lan`   | 192.168.0.14      | correct as-is (n8n really runs on docker-stack) |
  | `sillytavern.lan`  | 192.168.0.22      | was already correct |

  Fixed via `pct exec 109` (backed up the config first to
  `AdGuardHome.yaml.bak-20260725`, patched each domain's `answer` field with a `sed`
  anchored to that domain's block so the untouched entries and `n8n.docker.lan` couldn't
  be hit by accident, then `systemctl restart AdGuardHome`). Verified live with `dig`
  against `192.168.0.20` for all 12 domains post-restart — all correct.

  **Repointed again, later the same day, once the [Caddy reverse
  proxy](/playbooks/reverse-proxy-caddy.md) was wired up.** 9 of these same domains
  (everything except `nas.lan`, `tailscale.lan`, `n8n.docker.lan`) now answer
  `192.168.0.14` (docker-stack, where Caddy runs) instead of the backend's own IP — Caddy
  routes each request to the real backend by Host header. A new `qdrant.docker.lan` entry
  was also added (Caddy already had a block for it; AdGuard just never had a matching
  rewrite). See the playbook for the full current table, and
  [actions.md](/actions.md) for the fix record.

  **`kokoro.docker.lan` and `audiobooks.lan`, both → `192.168.0.14`, added 2026-07-26**,
  bringing the table to 15 rewrites. **`audiobookshelf.lan` → `192.168.0.14` was added the
  same day** for [Audiobookshelf (CT112)](/containers/audiobookshelf.md), making it 16.
  **`coder.lan` → `192.168.0.14` followed on 2026-07-26** for
  [code-server (CT113)](/containers/code-server.md) — 17. Note that one is the sole rewrite
  whose name doesn't match its guest's hostname (the container is `code-server`); it was
  requested as `coder.lan`. Because these `.14` answers are now *correct by design* (Caddy fronts them),
  a fresh reader should not mistake today's block of identical `.14` answers for a
  recurrence of the copy-paste bug above — the tell is whether the domain has a matching
  Caddy block, not the answer value.

  The edit was done by pulling the config to the host (`pct pull 109`), editing there, and
  pushing it back — rather than `sed`ing in place inside CT109 — specifically so the result
  could be **parsed with PyYAML and diffed before restarting** the service. Given this file
  has already crash-looped AdGuardHome once over indentation, validating offline is worth
  the extra two steps:

  ```bash
  pct exec 109 -- cp /opt/AdGuardHome/AdGuardHome.yaml /opt/AdGuardHome/AdGuardHome.yaml.bak-YYYYMMDD
  pct pull 109 /opt/AdGuardHome/AdGuardHome.yaml /root/agh.yaml
  # edit /root/agh.yaml, then:
  python3 -c "import yaml; d=yaml.safe_load(open('/root/agh.yaml')); print(len(d['filtering']['rewrites']))"
  diff /root/agh.yaml.orig /root/agh.yaml     # want ONLY your intended lines
  pct push 109 /root/agh.yaml /opt/AdGuardHome/AdGuardHome.yaml
  pct exec 109 -- systemctl restart AdGuardHome && pct exec 109 -- systemctl is-active AdGuardHome
  ```
* Rule of thumb: single-upstream DNS for a whole household is a self-inflicted outage
  waiting to happen. Keep ≥2 independent upstreams + plaintext fallbacks.

# Guest DNS policy — changed 2026-07-25

This originally said guest containers should inherit DNS
from the host (leave the DNS field blank on creation) and that the AdGuard path was for
tailnet clients only. In practice, three guests had already drifted onto AdGuard DNS
during unrelated fixes that day (docker-stack, home-assistant, sillytavern) before an
audit found the other 9 LXCs still pointed at the router — matching what was, at the time,
the *documented* policy. Asked Dave to settle which was actually intended: **guests should
use AdGuard**, so it was rolled out fleet-wide rather than reverted. Every guest except
[adguard (CT109)](/containers/adguard.md) itself (avoids a self-reference) and
[tailscale (CT105)](/containers/tailscale.md) (uses Tailscale's own MagicDNS resolver,
`100.100.100.100`) now points at `192.168.0.20`. Applied via `pct set <id> -nameserver
192.168.0.20` (LXCs) — note this does **not** take effect on a running container until it's
rebooted; `pct set` only rewrites the saved config. See [actions.md](/actions.md) for the
full rollout record.

# Privacy note (asked and answered)

This hides DNS queries from Virgin but not destinations — SNI is plaintext. Exit-node use
from away routes traffic through the home pipe (counts double against it).
Surfshark-in-gluetun on [docker-stack (VM103)](/containers/docker-stack.md) chained
behind the exit node is the discussed but unbuilt upgrade for actually blinding the ISP.

# Citations

[1] n5-pro-homelab Claude Skill — references/network-conventions.md, references/tailscale.md (Dave's claude.ai account, last updated 2026-07-19)
